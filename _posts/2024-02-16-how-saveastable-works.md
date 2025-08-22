---
layout: post
title: "Spark saveAsTable() 동작 원리"
date: 2024-02-16
categories: [Spark, Hive, AWS]
tags: [spark, hive, managed-table, external-table, glue, s3, troubleshooting]
---

Apache Spark에서 `saveAsTable()` 메소드는 DataFrame을 Hive 테이블로 저장하는 가장 일반적인 방법 중 하나다. 하지만 이 간단해 보이는 API 뒤에는 복잡한 메타데이터 관리와 파일 시스템 연산이 숨어있다. 특히 관리형 테이블(Managed Table)과 외부 테이블(External Table) 간의 변환 과정에서 예상치 못한 문제가 발생할 수 있다.

<!-- more -->

## 문제 상황

운영 중인 데이터 파이프라인에서 다음과 같이 관리형 테이블을 생성하고 있었다:
```scala
df.write.saveAsTable(name="my_database.my_table", format="parquet", mode="overwrite")
```
*DataFrame을 메타스토어의 테이블로 저장*


이 코드는 `path` 옵션을 명시하지 않았기 때문에 관리형(managed) 테이블로 생성된다. 관리형 테이블의 특징은 테이블을 삭제할 때 메타데이터뿐만 아니라 실제 데이터까지 함께 삭제된다는 점이다.

문제는 운영 중에 이 테이블을 외부 테이블로 변경해야 하는 상황이 발생했을 때 시작되었다.

## 관리형 테이블을 외부 테이블로 전환

### SparkSQL의 제약사항

처음에는 SparkSQL을 사용하여 테이블 속성을 변경하려고 시도했다:
```sql
ALTER TABLE my_database.my_table SET TBLPROPERTIES('EXTERNAL'='true')
```

하지만 이 명령은 실행되지 않았다. **SparkSQL은 보안상의 이유로 관리형 테이블을 외부 테이블로 직접 변환하는 것을 허용하지 않는다.** 이는 데이터 소유권과 생명주기 관리의 책임 소재가 명확히 구분되기 때문이다.

### Amazon Glue Catalog을 통한 해결

대안으로 Amazon Glue Catalog 콘솔에서 직접 테이블 속성을 수정했다. Glue Catalog의 웹 인터페이스에서는 테이블의 `EXTERNAL` 속성을 `true`로 설정할 수 있다. 이렇게 되면 Hive 메타스토어 관점에서 해당 테이블은 외부 테이블로 인식된다.
![Amazon Glue Data Catalog Console](https://github.com/user-attachments/assets/abb317d5-a3ae-40d4-a8ae-2b8db6093de9)

또 다른 방법으로는 HiveQL을 직접 사용하는 것이다:
```sql
/*
Hive CLI 또는 Beeline에서 실행
*/
ALTER TABLE my_database.my_table SET TBLPROPERTIES('EXTERNAL'='true')
```

## 문제 발생: LOCATION_ALREADY_EXISTS 에러

외부 테이블로 전환 후, 동일한 DataFrame을 다시 `saveAsTable()`로 저장하려고 하면 다음과 같은 에러가 발생했다:
```text
org.apache.spark.SparkRuntimeException: [LOCATION_ALREADY_EXISTS] Cannot name the managed table as `spark_catalog`.`my_database`.`my_table`, as its associated location ‘s3://my_s3_bucket/databases/my_database/my_table’ already exists.
Please pick a different table name, or remove the existing location first.
  at org.apache.spark.sql.errors.QueryExecutionErrors$.locationAlreadyExists(QueryExecutionErrors.scala:2793)
  at org.apache.spark.sql.catalyst.catalog.SessionCatalog.validateTableLocation(SessionCatalog.scala:414)
  at org.apache.spark.sql.execution.command.CreateDataSourceTableAsSelectCommand.run(createDataSourceTables.scala:176)
  at org.apache.spark.sql.execution.command.ExecutedCommandExec.sideEffectResult$lzycompute(commands.scala:75)
  at org.apache.spark.sql.execution.command.ExecutedCommandExec.sideEffectResult(commands.scala:73)
  at org.apache.spark.sql.execution.command.ExecutedCommandExec.executeCollect(commands.scala:84)
```

이 에러 메시지는 관리형 테이블로 생성하려는데 해당 테이블의 웨어하우스 저장소 경로에 이미 데이터가 존재한다는 의미다. Spark는 관리형 테이블의 데이터 생명주기를 완전히 제어해야 하기 때문에, 기존 데이터가 있는 위치에는 새로운 관리형 테이블을 생성할 수 없다.

## SaveMode.Overwrite의 동작 원리 분석

### Spark 내부 구현 분석

이 문제를 이해하기 위해 Spark의 `DataFrameWriter.saveAsTable()` 메소드의 내부 구현을 분석해보자:

```scala
  private def saveAsTable(tableIdent: TableIdentifier): Unit = {
    val catalog = df.sparkSession.sessionState.catalog
    val qualifiedIdent = catalog.qualifyIdentifier(tableIdent)
    val tableExists = catalog.tableExists(qualifiedIdent)
    val tableName = qualifiedIdent.unquotedString

    (tableExists, mode) match {
      case (true, SaveMode.Ignore) =>
        // Do nothing

      case (true, SaveMode.ErrorIfExists) =>
        throw QueryCompilationErrors.tableAlreadyExistsError(qualifiedIdent)

      case (true, SaveMode.Overwrite) =>
        // Get all input data source or hive relations of the query.
        val srcRelations = df.logicalPlan.collect {
          case LogicalRelation(src: BaseRelation, _, _, _) => src
          case relation: HiveTableRelation => relation.tableMeta.identifier
        }

        val tableRelation = df.sparkSession.table(qualifiedIdent).queryExecution.analyzed
        EliminateSubqueryAliases(tableRelation) match {
          // check if the table is a data source table (the relation is a BaseRelation).
          case LogicalRelation(dest: BaseRelation, _, _, _) if srcRelations.contains(dest) =>
            throw QueryCompilationErrors.cannotOverwriteTableThatIsBeingReadFromError(tableName)
          // check hive table relation when overwrite mode
          case relation: HiveTableRelation
              if srcRelations.contains(relation.tableMeta.identifier) =>
            throw QueryCompilationErrors.cannotOverwriteTableThatIsBeingReadFromError(tableName)
          case _ => // OK
        }

        // 기존 테이블을 제거한다.
        catalog.dropTable(qualifiedIdent, ignoreIfNotExists = true, purge = false)
        createTable(qualifiedIdent)
        // Refresh the cache of the table in the catalog.
        catalog.refreshTable(qualifiedIdent)

      case _ => createTable(qualifiedIdent)
    }
  } 
```
*DataFrameWrite.scala*

핵심은 **`SaveMode.Overwrite`일 때 Spark는 단순히 기존 테이블을 삭제하고 새 테이블을 생성한다는 점**이다. 문제는 테이블 삭제 과정에서 발생한다.

### 외부 테이블 삭제 시의 동작

Spark가 외부 테이블을 삭제할 때의 동작을 `HiveClientImpl.dropTable()` 메소드에서 확인할 수 있다:
```scala
override def dropTable(
      dbName: String,
      tableName: String,
      ignoreIfNotExists: Boolean,
      purge: Boolean): Unit = withHiveState {
    shim.dropTable(client, dbName, tableName, deleteData = true, ignoreIfNotExists, purge)
  }
```
*org.apache.spark.sql.hive.client.HiveClientImpl.scala*

여기서 `deleteData = true`로 설정되지만, 실제 데이터 삭제 여부는 테이블의 타입에 따라 결정된다. Amazon Glue의 경우 `GlueMetastoreClientDelegate.dropTable()`에서 최종 처리된다:
```java
   public void dropTable(
          String dbName,
          String tableName,
          boolean deleteData,
          boolean ignoreUnknownTbl,
          boolean ifPurge
  ) throws TException {
    checkArgument(StringUtils.isNotEmpty(dbName), "dbName cannot be null or empty");
    checkArgument(StringUtils.isNotEmpty(tableName), "tableName cannot be null or empty");

    if (!tableExists(dbName, tableName)) {
      if (!ignoreUnknownTbl) {
        throw new UnknownTableException("Cannot find table: " + dbName + "." + tableName);
      } else {
        return;
      }
    }

    org.apache.hadoop.hive.metastore.api.Table tbl = getTable(dbName, tableName);
    String tblLocation = tbl.getSd().getLocation();
    boolean isExternal = isExternalTable(tbl);
    dropPartitionsForTable(dbName, tableName, deleteData && !isExternal);

    try {
      // 메타데이터 삭제
      glueMetastore.deleteTable(dbName, tableName); 
    } catch (AmazonServiceException e){
      throw catalogToHiveConverter.wrapInHiveException(e);
    } catch (Exception e){
      String msg = "Unable to drop table: ";
      logger.error(msg, e);
      throw new MetaException(msg + e);
    }
    // 데이터 삭제 조건: deleteData=true AND !isExternal
    if (StringUtils.isNotEmpty(tblLocation) && deleteData && !isExternal) {
      Path tblPath = new Path(tblLocation);
      try {
        hiveShims.deleteDir(wh, tblPath, true, ifPurge);
      } catch (Exception e){
        logger.error("Unable to remove table directory " + tblPath, e);
      }
    }
  }
```
*GlueMetastoreClientDelegate.java*


**핵심 포인트:**
- **관리형 테이블**: `isExternal = false` → 메타데이터와 데이터 모두 삭제
- **외부 테이블**: `isExternal = true` → 메타데이터만 삭제, 데이터는 보존

### 외부 테이블 판별 로직

```java
public static boolean isExternalTable(org.apache.hadoop.hive.metastore.api.Table table) {
  if (table == null)
    return false;
  Map<String, String> params = table.getParameters();
  String paramsExternalStr = params == null ? null : params.get("EXTERNAL");
  
  // EXTERNAL 파라미터 확인
  if (paramsExternalStr != null) {
      return "TRUE".equalsIgnoreCase(paramsExternalStr);
  }
  
  // TableType으로 판별
  return table.getTableType() != null && 
         EXTERNAL_TABLE.name().equalsIgnoreCase(table.getTableType());
}
```


## 문제의 근본 원인

문제의 전체적인 흐름을 정리하면:

1. **관리형 테이블 생성**: `saveAsTable()`로 관리형 테이블 생성, 데이터는 기본 웨어하우스 경로에 저장
2. **외부 테이블로 전환**: Glue Console에서 `EXTERNAL=true` 속성 설정
3. **재저장 시도**: `SaveMode.Overwrite`로 다시 저장
4. **테이블 삭제**: 외부 테이블로 인식되어 메타데이터만 삭제, **데이터는 그대로 유지**
5. **새 테이블 생성 시도**: 관리형 테이블로 생성하려 하지만 기존 데이터가 남아있어 실패

이는 Spark의 테이블 생성 검증 로직 때문이다:
```scala
  def validateTableLocation(table: CatalogTable): Unit = {
    // SPARK-19724: the default location of a managed table should be non-existent or empty.
    if (table.tableType == CatalogTableType.MANAGED) {
      val tableLocation =
        new Path(table.storage.locationUri.getOrElse(defaultTablePath(table.identifier)))
      val fs = tableLocation.getFileSystem(hadoopConf)

      if (fs.exists(tableLocation) && fs.listStatus(tableLocation).nonEmpty) {
      // 다음 에러를 발생시킨다.
        throw QueryExecutionErrors.locationAlreadyExists(table.identifier, tableLocation)
      }
    }
  }
```

**관리형 테이블은 데이터의 완전한 소유권을 가져야 하므로, 기존 데이터가 있는 위치에는 생성할 수 없다.**

## 해결 방법

### 1. 기존 데이터 삭제 후 재생성

S3에서 기존 데이터 삭제
```shell
aws s3 rm s3://my_s3_bucket/databases/my_database/my_table/ –recursive
```

그 후 saveAsTable() 실행
```scala
df.write.saveAsTable(name="my_database.my_table", format="parquet", mode="overwrite")
```

### 2. 외부 테이블로 유지

명시적으로 외부 테이블로 생성
```scala
df.write.option("path", "s3://my_s3_bucket/databases/my_database/my_table/")
  .saveAsTable(name="my_database.my_table", format="parquet", mode="overwrite")
```

### 3. 새로운 테이블명 사용
다른 이름으로 테이블 생성
```scala
df.write.saveAsTable(name="my_database.my_table_v2", format="parquet", mode="overwrite")
```

### 4. 임시 테이블을 통한 우회 방법

```scala
// 임시 테이블로 생성 후 RENAME
df.write.saveAsTable(name="my_database.my_table_temp", format="parquet", mode="overwrite")
```
SQL로 테이블명 변경
```sql
ALTER TABLE my_database.my_table_temp RENAME TO my_database.my_table
```


## 운영 환경에서의 권장사항

### 1. 테이블 타입 일관성 유지

처음부터 외부 테이블로 생성하여 일관성 유지
```scala
df.write.option("path", s"${warehouse_location}/${database}/${table}/")
  .saveAsTable(name=s"${database}.${table}", format="parquet", mode="overwrite")
```

### 2. 모니터링 및 알림

```python
def safe_save_as_table(df, table_name, **options):
    """안전한 saveAsTable wrapper 함수"""
    try:
        df.write.saveAsTable(name=table_name, **options)
        print(f"테이블 {table_name} 저장 완료")
    except Exception as e:
        if "LOCATION_ALREADY_EXISTS" in str(e):
            print(f"경고: {table_name} 위치에 기존 데이터가 존재합니다.")
            print("해결 방안: 1) 기존 데이터 삭제 2) 외부 테이블로 생성 3) 다른 테이블명 사용")
        raise e

```

## 결론과 교훈

이번 문제를 해결하면서 가장 큰 깨달음은 `saveAsTable()` API의 숨겨진 복잡성이었다. 단순해 보이는 이 메소드가 실제로는 메타스토어 상호작용, 파일 시스템 검증, 테이블 생명주기 관리 등 복잡한 연산을 수행한다는 것을 알게 되었다. 특히 관리형 테이블과 외부 테이블의 차이가 단순한 속성 차이가 아니라 데이터 소유권에 대한 철학적 차이라는 점이 인상적이었다. Spark는 관리형 테이블에 대해서는 완전한 통제권을 주장하기 때문에, 기존 데이터가 있는 위치에 새로운 관리형 테이블을 생성하는 것을 엄격히 금지한다.

운영 환경에서 겪은 이 경험을 통해 “처음부터 명확한 설계 원칙을 세우는 것”의 중요성을 깨달았다. 테이블 타입을 미리 결정하고 가급적 변경하지 않는 것이 최선이며, AWS Glue Catalog와 Spark 간의 미묘한 동작 차이도 고려해야 한다는 교훈을 얻었다. 앞으로는 간단해 보이는 API 뒤에 숨어있는 복잡한 비즈니스 로직을 항상 염두에 두고, 메타데이터 관리 작업은 충분한 이해와 계획을 바탕으로 수행해야겠다.



