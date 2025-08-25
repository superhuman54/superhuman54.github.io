---
layout: post
title: "정렬된 Parquet는 어떻게 Row Group을 스킵하는가"
date: 2025-01-07 12:00:00 +0900
categories: [Data Engineering, Parquet, Performance]
tags: [parquet, spark, performance, binary-search, push-down]
author: K4N
description: "정렬된 Parquet 파일에서 Binary Search를 활용한 Row Group 스킵 메커니즘을 상세히 분석합니다. ASCENDING/DESCENDING 정렬과 성능 최적화 방법을 다룹니다."
keywords: "parquet, spark, performance, binary-search, push-down, row-group, boundary-order, column-index"
---

정렬된 Parquet 파일을 사용하면 쿼리 성능이 크게 향상된다는 건 알고 있지만, 정말 궁금한 건 어떻게 그게 가능한지다. 분명히 어떤 Row Group들은 조건에 맞지 않아서 skip될 텐데, 어떤 메타데이터 덕분에 그런 판단이 가능했을까? 이 글에서는 정렬된 Parquet 파일이 어떻게 Row Group을 효율적으로 스킵하는지, 그리고 그 뒤에 숨겨진 Binary Search 알고리즘을 자세히 살펴보자.

<!-- more -->

## 개요

정렬된 Parquet 파일은 Binary Search 알고리즘을 활용해서 불필요한 Row Group을 스킵함으로써 쿼리 성능을 크게 향상시킨다. 이 메커니즘의 핵심은 `BoundaryOrder`와 Column Index를 통한 효율적인 필터링이다.

## Parquet 파일 구조와 Row Group

Parquet 파일은 다음과 같은 계층 구조를 가진다:

```
Parquet File
├── Row Group 0
│   ├── Column Chunk 0 (name)
│   ├── Column Chunk 1 (age)
│   └── Column Chunk 2 (city)
├── Row Group 1
│   ├── Column Chunk 0 (name)
│   ├── Column Chunk 1 (age)
│   └── Column Chunk 2 (city)
└── ...
```

각 Row Group은 독립적으로 처리될 수 있고, 이것이 병렬 처리와 필터링 최적화의 핵심이다.

## 정렬된 데이터의 BoundaryOrder

Parquet에서 컬럼이 정렬되어 있는지 여부는 `BoundaryOrder` enum으로 표현된다:

```java
public enum BoundaryOrder {
  UNORDERED,    // 정렬되지 않음
  ASCENDING,    // 오름차순 정렬
  DESCENDING    // 내림차순 정렬
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java</span>
</div>

## BoundaryOrder는 Column Index에만 존재

**중요한 점**: `BoundaryOrder`는 Parquet 포맷 스펙에서 정의된 enum으로, **Column Index에만 존재**합니다. Row Group 레벨에는 정렬 정보가 저장되지 않습니다.

```java
// parquet-format-structures/target/generated-sources/thrift/org/apache/parquet/format/BoundaryOrder.java
public enum BoundaryOrder implements org.apache.thrift.TEnum {
  UNORDERED(0),
  ASCENDING(1), 
  DESCENDING(2);
}
```

<div class="code-footer">
  <span class="file-path">parquet-format-structures/target/generated-sources/thrift/org/apache/parquet/format/BoundaryOrder.java</span>
</div>

## Row Group 저장 시 메타데이터 구조

Row Group을 저장할 때는 정렬 정보 없이 단순한 통계 정보만 저장됩니다:

```java
// parquet-hadoop/src/main/java/org/apache/parquet/format/converter/ParquetMetadataConverter.java
private void addRowGroup(
    ParquetMetadata parquetMetadata,
    List<RowGroup> rowGroups,
    BlockMetaData block,
    InternalFileEncryptor fileEncryptor) {

  List<ColumnChunkMetaData> columns = block.getColumns();
  List<ColumnChunk> parquetColumns = new ArrayList<ColumnChunk>();
  
  for (ColumnChunkMetaData columnMetaData : columns) {
    ColumnChunk columnChunk = new ColumnChunk();
    columnChunk.setFile_path(columnMetaData.getPath().toDotString());
    columnChunk.setFile_offset(columnMetaData.getFirstDataPageOffset());
    columnChunk.setMeta_data(columnMetaData.getParquetMetadata());
    
    // Column Index 정보 추가 (BoundaryOrder 포함)
    if (columnMetaData.getColumnIndexReference() != null) {
      columnChunk.setColumn_index_offset(columnMetaData.getColumnIndexReference().getOffset());
      columnChunk.setColumn_index_length(columnMetaData.getColumnIndexReference().getLength());
    }
    
    parquetColumns.add(columnChunk);
  }
  
  // Row Group 생성 (정렬 정보 없음)
  RowGroup rowGroup = new RowGroup(parquetColumns, block.getTotalByteSize(), block.getRowCount());
  rowGroup.setFile_offset(block.getStartingPos());
  rowGroup.setTotal_compressed_size(block.getCompressedSize());
  rowGroup.setOrdinal((short) rowGroupOrdinal);
  rowGroups.add(rowGroup);
}
```

<div class="code-footer">
  <span class="file-path">parquet-hadoop/src/main/java/org/apache/parquet/format/converter/ParquetMetadataConverter.java</span>
</div>

**Row Group 저장 시 특징:**
- **정렬 정보 없음**: Row Group 자체에는 `BoundaryOrder` 정보가 저장되지 않음
- **통계 정보만**: Row Group은 단순한 통계 정보(min/max, 행 개수, 크기 등)만 저장
- **Column Index 분리**: 정렬 정보는 Column Chunk 내의 Column Index에 별도로 저장

## 1차 필터: 통계 기반 스킵

Parquet는 세 단계의 필터링을 통해 Row Group을 효율적으로 스킵합니다:

```java
// parquet-hadoop/src/main/java/org/apache/parquet/filter2/compat/RowGroupFilter.java
@Override
public List<BlockMetaData> visit(FilterCompat.FilterPredicateCompat filterPredicateCompat) {
  FilterPredicate filterPredicate = filterPredicateCompat.getFilterPredicate();
  
  List<BlockMetaData> filteredBlocks = new ArrayList<BlockMetaData>();

  for (BlockMetaData block : blocks) {
    boolean drop = false;

    // 1. 통계 기반 스킵 (Statistics Level)
    if (levels.contains(FilterLevel.STATISTICS)) {
      drop = StatisticsFilter.canDrop(filterPredicate, block.getColumns());
    }

    // 2. 딕셔너리 기반 스킵 (Dictionary Level)
    if (!drop && levels.contains(FilterLevel.DICTIONARY)) {
      try (DictionaryPageReadStore dictionaryPageReadStore = reader.getDictionaryReader(block)) {
        drop = DictionaryFilter.canDrop(filterPredicate, block.getColumns(), dictionaryPageReadStore);
      }
    }

    // 3. 블룸 필터 기반 스킵 (Bloom Filter Level)
    if (!drop && levels.contains(FilterLevel.BLOOMFILTER)) {
      drop = BloomFilterImpl.canDrop(
          filterPredicate, block.getColumns(), reader.getBloomFilterDataReader(block));
    }

    if (!drop) {
      filteredBlocks.add(block);
    }
  }

  return filteredBlocks;
}
```

<div class="code-footer">
  <span class="file-path">parquet-hadoop/src/main/java/org/apache/parquet/filter2/compat/RowGroupFilter.java</span>
</div>

**통계 기반 스킵의 세 가지 단계:**

1. **Statistics Level**: Row Group의 min/max 통계로 스킵 판단
2. **Dictionary Level**: 딕셔너리 값으로 스킵 판단
3. **Bloom Filter Level**: 블룸 필터로 스킵 판단

**Statistics Level의 실제 동작:**

```java
// parquet-hadoop/src/main/java/org/apache/parquet/filter2/statisticslevel/StatisticsFilter.java
@Override
@SuppressWarnings("unchecked")
public <T extends Comparable<T>> Boolean visit(Gt<T> gt) {
  Column<T> filterColumn = gt.getColumn();
  ColumnChunkMetaData meta = getColumnChunk(filterColumn.getColumnPath());
  
  Statistics<T> stats = meta.getStatistics();
  T value = gt.getValue();

  // 핵심 로직: value >= max이면 Row Group을 스킵
  return stats.compareMaxToValue(value) <= 0;
}

@Override
@SuppressWarnings("unchecked")
public <T extends Comparable<T>> Boolean visit(Lt<T> lt) {
  Column<T> filterColumn = lt.getColumn();
  ColumnChunkMetaData meta = getColumnChunk(filterColumn.getColumnPath());
  
  Statistics<T> stats = meta.getStatistics();
  T value = lt.getValue();

  // 핵심 로직: value <= min이면 Row Group을 스킵
  return stats.compareMinToValue(value) >= 0;
}
```

<div class="code-footer">
  <span class="file-path">parquet-hadoop/src/main/java/org/apache/parquet/filter2/statisticslevel/StatisticsFilter.java</span>
</div>

## BoundaryOrder 계산 과정

### 1. Column Index 생성 시점

`BoundaryOrder`는 Column Index가 생성될 때 자동으로 계산된다:

```java
public ColumnIndex build() {
  ColumnIndexBase<?> columnIndex = build(type);
  if (columnIndex == null) {
    return null;
  }
  columnIndex.boundaryOrder = calculateBoundaryOrder(type.comparator());
  return columnIndex;
}

private BoundaryOrder calculateBoundaryOrder(PrimitiveComparator<Binary> comparator) {
  if (isAscending(comparator)) {
    return BoundaryOrder.ASCENDING;
  } else if (isDescending(comparator)) {
    return BoundaryOrder.DESCENDING;
  } else {
    return BoundaryOrder.UNORDERED;
  }
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/ColumnIndexBuilder.java</span>
</div>

### 2. 정렬 판단 로직

```java
// min[i] <= min[i+1] && max[i] <= max[i+1]
private boolean isAscending(PrimitiveComparator<Binary> comparator) {
  for (int i = 1, n = pageIndexes.size(); i < n; ++i) {
    if (compareMinValues(comparator, i - 1, i) > 0 || compareMaxValues(comparator, i - 1, i) > 0) {
      return false;
    }
  }
  return true;
}

// min[i] >= min[i+1] && max[i] >= max[i+1]
private boolean isDescending(PrimitiveComparator<Binary> comparator) {
  for (int i = 1, n = pageIndexes.size(); i < n; ++i) {
    if (compareMinValues(comparator, i - 1, i) < 0 || compareMaxValues(comparator, i - 1, i) < 0) {
      return false;
    }
  }
  return true;
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/ColumnIndexBuilder.java</span>
</div>

## Column Index 크기 제한과 BoundaryOrder 생성

### 1. Column Index 크기 제한 메커니즘

Parquet에서 Column Index의 크기가 설정된 제한을 초과하면 `BoundaryOrder`가 포함된 Column Index 자체가 생성되지 않는다:

```java
public void endColumn() throws IOException {
  state = state.endColumn();
  LOG.debug("{}: end column", out.getPos());
  
  // Column Index 크기 체크
  if (columnIndexBuilder.getMinMaxSize() > columnIndexBuilder.getPageCount() * MAX_STATS_SIZE) {
    currentColumnIndexes.add(null);  // Column Index 생성 안함
  } else {
    currentColumnIndexes.add(columnIndexBuilder.build());  // Column Index 생성 (BoundaryOrder 포함)
  }
  // ...
}
```

<div class="code-footer">
  <span class="file-path">parquet-hadoop/src/main/java/org/apache/parquet/hadoop/ParquetFileWriter.java</span>
</div>

### 2. MAX_STATS_SIZE 제한

```java
public static final int MAX_STATS_SIZE = 4096;  // 4KB
```

<div class="code-footer">
  <span class="file-path">parquet-hadoop/src/main/java/org/apache/parquet/format/converter/ParquetMetadataConverter.java</span>
</div>

### 3. 크기 계산 방식

`getMinMaxSize()`는 각 페이지의 min/max 값들의 총 크기를 계산한다:

```java
// IntColumnIndexBuilder의 경우
public long getMinMaxSize() {
  return (long) minValues.size() * Integer.BYTES + (long) maxValues.size() * Integer.BYTES;
}

// BinaryColumnIndexBuilder의 경우  
public long getMinMaxSize() {
  long minSizesSum = minValues.stream().mapToLong(Binary::length).sum();
  long maxSizesSum = maxValues.stream().mapToLong(Binary::length).sum();
  return minSizesSum + maxSizesSum;
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/IntColumnIndexBuilder.java</span>
</div>

### 4. 실제 예시

**예시 1: Column Index 생성되는 경우**

```
페이지 수: 10개
각 페이지 min/max 크기: 200 bytes (Integer 타입)
총 크기: 10 × 200 = 2,000 bytes
제한: 10 × 4,096 = 40,960 bytes
결과: 2,000 < 40,960 → Column Index 생성됨 (BoundaryOrder 포함)
```

**예시 2: Column Index 생성되지 않는 경우**

```
페이지 수: 50개
각 페이지 min/max 크기: 10,000 bytes (매우 긴 String)
총 크기: 50 × 10,000 = 500,000 bytes
제한: 50 × 4,096 = 204,800 bytes
결과: 500,000 > 204,800 → Column Index 생성 안됨 (BoundaryOrder도 없음)
```

### 5. 영향과 대응 방안

**Column Index가 생성되지 않을 때의 영향:**
- `BoundaryOrder` 정보 없음
- Binary Search 기반 필터링 불가능
- 모든 Row Group을 순차적으로 검사해야 함
- 성능 저하 발생

**대응 방안:**
```scala
// 페이지 크기를 작게 설정하여 페이지 수 줄이기
spark.conf.set("parquet.page.size", "1MB")

// Row Group 크기를 조정하여 페이지 수 제어
spark.conf.set("parquet.block.size", "50MB")

// String 컬럼의 경우 길이 제한 고려
val truncatedDF = df.withColumn("long_string", substring(col("long_string"), 1, 100))
```

## 2차 필터: ASCENDING 정렬된 Row Group에서 Binary Search

통계 기반 스킵 후, Column Index의 `BoundaryOrder`를 활용한 Binary Search가 수행됩니다.

### 1. gt (greater than) 연산

```java
@Override
OfInt gt(ColumnIndexBase<?>.ValueComparator comparator) {
  int length = comparator.arrayLength();
  if (length == 0) {
    return IndexIterator.EMPTY;
  }
  int left = 0;
  int right = length;
  do {
    int i = floorMid(left, right);
    if (comparator.compareValueToMax(i) >= 0) {
      left = i + 1;  // 이 페이지의 최대값이 검색값보다 작으면 다음 페이지로
    } else {
      right = i;     // 이 페이지에 검색값보다 큰 값이 있을 수 있음
    }
  } while (left < right);
  return IndexIterator.rangeTranslate(right, length - 1, comparator::translate);
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java</span>
</div>

### 2. 실제 예시

ASCENDING 정렬된 age 컬럼이 있다고 가정해보자.

```
Row Group 0: min=10, max=50
Row Group 1: min=51, max=100  
Row Group 2: min=101, max=150
Row Group 3: min=151, max=200
Row Group 4: min=201, max=250
```

**검색 조건: `age > 120`**

Binary Search 과정:
1. **초기 상태**: left=0, right=5
2. **중간값 계산**: i = floorMid(0, 5) = 2
3. **Row Group 2 검사**: max=150 >= 120? → false → right=2
4. **중간값 계산**: i = floorMid(0, 2) = 1  
5. **Row Group 1 검사**: max=100 >= 120? → false → right=1
6. **중간값 계산**: i = floorMid(0, 1) = 0
7. **Row Group 0 검사**: max=50 >= 120? → false → right=0
8. **결과**: right=0부터 끝까지 (Row Group 2, 3, 4)

**최종 결과**: Row Group 2, 3, 4만 읽는다 (Row Group 0, 1은 스킵)

### 3. lt (less than) 연산

```java
@Override
OfInt lt(ColumnIndexBase<?>.ValueComparator comparator) {
  int length = comparator.arrayLength();
  if (length == 0) {
    return IndexIterator.EMPTY;
  }
  int left = -1;
  int right = length - 1;
  do {
    int i = ceilingMid(left, right);
    if (comparator.compareValueToMin(i) <= 0) {
      right = i - 1;  // 이 페이지의 최소값이 검색값보다 크면 이전 페이지로
    } else {
      left = i;       // 이 페이지에 검색값보다 작은 값이 있을 수 있음
    }
  } while (left < right);
  return IndexIterator.rangeTranslate(0, left, comparator::translate);
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java</span>
</div>

## DESCENDING 정렬된 Row Group에서 역순 Binary Search

### 1. gt (greater than) 연산

```java
@Override
OfInt gt(ColumnIndexBase<?>.ValueComparator comparator) {
  int length = comparator.arrayLength();
  if (length == 0) {
    return IndexIterator.EMPTY;
  }
  int left = -1;
  int right = length - 1;
  do {
    int i = ceilingMid(left, right);
    if (comparator.compareValueToMax(i) >= 0) {
      right = i - 1;  // 이 페이지의 최대값이 검색값보다 작으면 이전 페이지로
    } else {
      left = i;       // 이 페이지에 검색값보다 큰 값이 있을 수 있음
    }
  } while (left < right);
  return IndexIterator.rangeTranslate(0, left, comparator::translate);
}
```

<div class="code-footer">
  <span class="file-path">parquet-column/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java</span>
</div>

### 2. 실제 예시

DESCENDING 정렬된 age 컬럼이 있다고 가정해보자.

```
Row Group 0: min=250, max=300
Row Group 1: min=201, max=249
Row Group 2: min=151, max=200
Row Group 3: min=101, max=150
Row Group 4: min=51, max=100
```

**검색 조건: `age > 120`**

역순 Binary Search 과정:
1. **초기 상태**: left=-1, right=4
2. **중간값 계산**: i = ceilingMid(-1, 4) = 2
3. **Row Group 2 검사**: max=200 >= 120? → true → right=1
4. **중간값 계산**: i = ceilingMid(-1, 1) = 0
5. **Row Group 0 검사**: max=300 >= 120? → true → right=-1
6. **결과**: 0부터 left까지 (Row Group 0, 1)

**최종 결과**: Row Group 0, 1만 읽는다 (Row Group 2, 3, 4는 스킵)

## Spark에서 정렬된 Parquet 생성하기

### 1. DataFrame 정렬 후 저장

```scala
// 오름차순 정렬
val sortedDF = df.orderBy("age")
sortedDF.write.parquet("/path/to/output")

// 내림차순 정렬
val sortedDFDesc = df.orderBy(col("age").desc)
sortedDFDesc.write.parquet("/path/to/output")

// 여러 컬럼 정렬
val multiSortedDF = df.orderBy("age", "name")
multiSortedDF.write.parquet("/path/to/output")
```

### 2. Java 예시

```java
// 오름차순 정렬
Dataset<Row> sortedDF = df.orderBy("age");
sortedDF.write().parquet("/path/to/output");

// 내림차순 정렬
Dataset<Row> sortedDFDesc = df.orderBy(functions.col("age").desc());
sortedDFDesc.write().parquet("/path/to/output");
```

## 성능 최적화 팁

### 1. 페이지 크기 조정

```scala
// 페이지 크기를 작게 설정하여 정렬 효과 극대화
spark.conf.set("parquet.page.size", "1MB")
spark.conf.set("parquet.block.size", "10MB")
```

### 2. Column Index 크기 제한

```java
if (columnIndexBuilder.getMinMaxSize() > columnIndexBuilder.getPageCount() * MAX_STATS_SIZE) {
  currentColumnIndexes.add(null);  // Column Index 생성 안함
} else {
  currentColumnIndexes.add(columnIndexBuilder.build());  // Column Index 생성
}
```
<div class="code-footer">
  <span class="file-path">parquet-hadoop/src/main/java/org/apache/parquet/hadoop/ParquetFileWriter.java</span>
</div>

Column Index의 크기가 4KB × 페이지 수를 초과하면 생성되지 않는다.

## 마무리

정렬된 Parquet 파일은 두 단계의 필터링을 통해 Row Group을 효율적으로 스킵할 수 있다. 첫 번째는 Row Group 레벨의 통계 정보를 활용한 순차 검색이고, 두 번째는 Column Index의 BoundaryOrder를 활용한 Binary Search이다.

### 핵심 포인트
1. **BoundaryOrder 위치**: Column Index에만 존재하며, Row Group에는 정렬 정보가 저장되지 않음
2. **1차 필터링**: Row Group 통계 정보로 순차 검색 (Statistics, Dictionary, Bloom Filter 레벨)
3. **2차 필터링**: Column Index의 BoundaryOrder로 Binary Search 수행
4. **Row Group 메타데이터**: 정렬 정보 없이 단순 통계만 저장
5. **Column Index**: BoundaryOrder로 페이지 정렬 정보 저장
6. **Column Index 제한**: 크기 제한으로 인한 BoundaryOrder 생성 실패 가능성 고려

정렬된 데이터의 이런 특성을 활용하면 데이터 웨어하우스나 빅데이터 환경에서 쿼리 성능을 개선할 수 있다.