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

## 이 글을 쓰게 된 동기

데이터 엔지니어링을 하면서 항상 궁금했던 것이 있다. "정렬된 Parquet 파일이 정말로 성능이 좋다는데, 도대체 어떻게 그게 가능한 거지?"라는 질문이었다. 

분명히 어떤 Row Group들은 조건에 맞지 않아서 skip될 텐데, 어떤 메타데이터 덕분에 그런 판단이 가능했을까? 단순히 "정렬되어 있으니까 빠르다"는 설명으로는 부족했다. 실제로 어떤 알고리즘이 동작하고, 어떤 메타데이터가 저장되어 있는지 궁금했다.

특히 Spark에서 `orderBy()` 후 Parquet로 저장하면 성능이 좋아진다는 건 알았지만, 그 뒤에 숨겨진 기술적 세부사항을 이해하고 싶었다. 이 글에서는 정렬된 Parquet 파일이 어떻게 Row Group을 효율적으로 스킵하는지, 그리고 그 뒤에 숨겨진 Binary Search 알고리즘을 자세히 살펴보려고 한다.

<!-- more -->

## 개요

정렬된 Parquet 파일은 두 단계의 필터링을 통해 효율적인 쿼리 성능을 달성한다. 첫 번째는 Row Group 레벨의 통계 기반 순차 검색이고, 두 번째는 Column Index 레벨의 Binary Search이다.

**중요한 점**: Row Group 레벨에서는 정렬 여부와 관계없이 순차 검색을 한다. 정렬의 진정한 효과는 Column Index 레벨에서 나타난다.

## Parquet 파일 구조 상세 분석

### 기본 계층 구조

Parquet 파일은 다음과 같은 계층 구조를 가진다:

```
Parquet File
├── File Header
├── Row Group 0
│   ├── Column Chunk 0 (name)
│   │   ├── Page 0 (min=Alice, max=Bob)
│   │   ├── Page 1 (min=Charlie, max=David)
│   │   └── Column Index (BoundaryOrder=UNORDERED)
│   ├── Column Chunk 1 (age)
│   │   ├── Page 0 (min=10, max=30)
│   │   ├── Page 1 (min=31, max=50)
│   │   └── Column Index (BoundaryOrder=ASCENDING)
│   └── Column Chunk 2 (city)
│       ├── Page 0 (min=Seoul, max=Tokyo)
│       ├── Page 1 (min=London, max=Paris)
│       └── Column Index (BoundaryOrder=DESCENDING)
├── Row Group 1
│   ├── Column Chunk 0 (name)
│   ├── Column Chunk 1 (age)
│   └── Column Chunk 2 (city)
└── File Footer
    ├── Row Group Metadata
    ├── Column Index Metadata
    └── Offset Index Metadata
```

### 각 구성 요소의 역할

**1. Row Group (Row Group)**
- Parquet 파일을 논리적으로 나누는 단위
- 독립적으로 처리 가능한 데이터 블록
- 통계 정보만 저장 (min/max, 행 개수, 크기)
- **중요**: Row Group 자체에는 정렬 정보가 없다

**2. Column Chunk (Column Chunk)**
- 하나의 컬럼에 대한 데이터 블록
- Row Group 내에서 컬럼별로 분리
- Column Index와 Offset Index를 포함

**3. Page (Page)**
- 실제 데이터가 저장되는 최소 단위
- 압축, 인코딩이 적용되는 레벨
- 각 페이지마다 min/max 통계 정보
- **중요**: Page 자체에는 BoundaryOrder 정보가 없다

**4. Column Index**
- 페이지 레벨의 정렬 정보 (`BoundaryOrder`) 포함
- Binary Search를 위한 메타데이터
- 페이지별 min/max 값들의 정렬 상태
- **중요**: BoundaryOrder는 Column Index에만 존재한다

**5. Offset Index**
- 페이지의 물리적 위치 정보
- 실제 데이터 접근을 위한 오프셋

### 메타데이터 저장 위치

```
File Footer
├── Row Group Metadata
│   ├── Row Group 0: min/max, 행 개수, 크기 (정렬 정보 없음)
│   └── Row Group 1: min/max, 행 개수, 크기 (정렬 정보 없음)
├── Column Index Metadata (각 Row Group의 각 Column Chunk마다)
│   ├── Row Group 0
│   │   ├── Column Chunk 0: BoundaryOrder=UNORDERED
│   │   ├── Column Chunk 1: BoundaryOrder=ASCENDING
│   │   └── Column Chunk 2: BoundaryOrder=DESCENDING
│   └── Row Group 1
│       ├── Column Chunk 0: BoundaryOrder=UNORDERED
│       ├── Column Chunk 1: BoundaryOrder=ASCENDING
│       └── Column Chunk 2: BoundaryOrder=DESCENDING
└── Offset Index Metadata
    └── 페이지별 물리적 위치 정보
```

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

### BoundaryOrder의 의미

**UNORDERED**: 페이지들이 임의의 순서로 배치되어 있다. 필터링 시 모든 페이지를 순차적으로 검사해야 한다.

**ASCENDING**: 페이지들이 오름차순으로 정렬되어 있다. Binary Search를 통해 효율적인 필터링이 가능하다.

**DESCENDING**: 페이지들이 내림차순으로 정렬되어 있다. 역순 Binary Search를 통해 효율적인 필터링이 가능하다.

## BoundaryOrder는 Column Index에만 존재

**중요한 점**: `BoundaryOrder`는 Parquet 포맷 스펙에서 정의된 enum으로, **Column Index에만 존재**한다. Row Group 레벨이나 Page 레벨에는 정렬 정보가 저장되지 않는다.

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

### 왜 Row Group에는 정렬 정보가 없을까?

Row Group은 단순한 통계 정보만 저장한다. 정렬 정보는 Column Index에 저장하는 이유는:

1. **병렬 처리**: Row Group이 병렬 처리되기 때문에 Row Group 레벨의 정렬 정보는 비효율적
2. **메모리 효율성**: Row Group 레벨에서 정렬 정보를 저장하면 메타데이터 크기가 커진다
3. **유연성**: 같은 Row Group 내에서도 컬럼별로 다른 정렬 상태를 가질 수 있다
4. **성능**: Column Index 레벨에서 Binary Search가 더 효율적이다

### 왜 Page에는 정렬 정보가 없을까?

Page 레벨에서도 정렬 정보를 저장하지 않는 이유는:

1. **중복 제거**: Column Index에서 이미 페이지들의 정렬 상태를 관리한다
2. **메모리 효율성**: 각 페이지마다 정렬 정보를 저장하면 메타데이터 크기가 커진다
3. **일관성**: Column Index에서 페이지들의 전체적인 정렬 상태를 한 번에 관리하는 것이 더 효율적이다

## Row Group 저장 시 메타데이터 구조

Row Group을 저장할 때는 정렬 정보 없이 단순한 통계 정보만 저장된다:

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

## 1차 필터: Row Group 레벨 통계 기반 순차 검색

**중요**: Row Group 레벨에서는 정렬 여부와 관계없이 순차 검색을 한다. 모든 Row Group을 하나씩 검사하여 조건에 맞는지 판단한다.

### 필터링 단계

```java
// parquet-hadoop/src/main/java/org/apache/parquet/filter2/compat/RowGroupFilter.java
@Override
public List<BlockMetaData> visit(FilterCompat.FilterPredicateCompat filterPredicateCompat) {
  FilterPredicate filterPredicate = filterPredicateCompat.getFilterPredicate();
  
  List<BlockMetaData> filteredBlocks = new ArrayList<BlockMetaData>();

  // 모든 Row Group을 순차적으로 검사 (정렬 여부 무관)
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

### Statistics Level의 실제 동작

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

### Row Group 순차 검색의 실제 예시

ASCENDING 정렬된 age 컬럼이 있다고 가정해보자.

```
Row Group 0: min=10, max=50
Row Group 1: min=51, max=100  
Row Group 2: min=101, max=150
Row Group 3: min=151, max=200
Row Group 4: min=201, max=250
```

**검색 조건: `age > 120`**

Row Group 순차 검색 과정:
1. **Row Group 0 검사**: max=50 >= 120? → false → 스킵
2. **Row Group 1 검사**: max=100 >= 120? → false → 스킵
3. **Row Group 2 검사**: max=150 >= 120? → true → 포함
4. **Row Group 3 검사**: max=200 >= 120? → true → 포함
5. **Row Group 4 검사**: max=250 >= 120? → true → 포함

**결과**: Row Group 2, 3, 4만 선택됨 (Row Group 0, 1은 스킵)

**중요**: 정렬된 데이터라도 Row Group 레벨에서는 순차 검색을 한다. 정렬의 효과는 Row Group을 스킵하는 것이 아니라, 통과한 Row Group 내부의 페이지 필터링에서 나타난다.

### 왜 Row Group 레벨에서는 순차 검색을 할까?

Row Group 레벨에서 순차 검색을 하는 이유는:

1. **병렬 처리**: Row Group이 병렬 처리되기 때문에 Row Group 레벨의 정렬 정보는 비효율적
2. **메타데이터 구조**: Row Group에는 정렬 정보가 저장되지 않는다
3. **메모리 효율성**: Row Group 수가 많을 때 정렬 정보를 저장하면 메타데이터 크기가 커진다
4. **실용성**: 대부분의 경우 Row Group 수가 많지 않아 순차 검색으로도 충분하다

## 2차 필터: Column Index 레벨 Binary Search

통과한 Row Group 내부의 Column Index에서 `BoundaryOrder`를 활용한 Binary Search가 수행된다. 이 단계에서 정렬의 진정한 효과가 나타난다.

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

통과한 Row Group 2 내부의 페이지들이 ASCENDING 정렬되어 있다고 가정해보자.

```
Row Group 2 내부 페이지들:
Page 0: min=101, max=120
Page 1: min=121, max=130
Page 2: min=131, max=140
Page 3: min=141, max=150
```

**검색 조건: `age > 125`**

Column Index Binary Search 과정:
1. **초기 상태**: left=0, right=4
2. **중간값 계산**: i = floorMid(0, 4) = 2
3. **Page 2 검사**: max=140 >= 125? → true → right=2
4. **중간값 계산**: i = floorMid(0, 2) = 1
5. **Page 1 검사**: max=130 >= 125? → true → right=1
6. **중간값 계산**: i = floorMid(0, 1) = 0
7. **Page 0 검사**: max=120 >= 125? → false → right=0
8. **결과**: right=0부터 끝까지 (Page 1, 2, 3)

**최종 결과**: Page 1, 2, 3만 읽는다 (Page 0은 스킵)

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

## DESCENDING 정렬된 Column Index에서 역순 Binary Search

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

통과한 Row Group 내부의 페이지들이 DESCENDING 정렬되어 있다고 가정해보자.

```
Row Group 내부 페이지들:
Page 0: min=150, max=160
Page 1: min=140, max=149
Page 2: min=130, max=139
Page 3: min=120, max=129
```

**검색 조건: `age > 125`**

역순 Binary Search 과정:
1. **초기 상태**: left=-1, right=3
2. **중간값 계산**: i = ceilingMid(-1, 3) = 1
3. **Page 1 검사**: max=149 >= 125? → true → right=0
4. **중간값 계산**: i = ceilingMid(-1, 0) = 0
5. **Page 0 검사**: max=160 >= 125? → true → right=-1
6. **결과**: 0부터 left까지 (Page 0, 1)

**최종 결과**: Page 0, 1만 읽는다 (Page 2, 3는 스킵)

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

## 정리

이 글을 통해 정렬된 Parquet 파일이 어떻게 효율적인 필터링을 수행하는지 자세히 살펴봤다. 핵심은 두 단계의 필터링 메커니즘이다.

### 핵심 포인트

1. **BoundaryOrder 위치**: Column Index에만 존재하며, Row Group이나 Page에는 정렬 정보가 저장되지 않음
2. **1차 필터링**: Row Group 통계 정보로 순차 검색 (Statistics, Dictionary, Bloom Filter 레벨)
3. **2차 필터링**: Column Index의 BoundaryOrder로 Binary Search 수행
4. **Row Group 메타데이터**: 정렬 정보 없이 단순 통계만 저장
5. **Column Index**: BoundaryOrder로 페이지 정렬 정보 저장
6. **Column Index 제한**: 크기 제한으로 인한 BoundaryOrder 생성 실패 가능성 고려

### 왜 이렇게 설계했을까?

Parquet의 이런 설계는 메모리 효율성과 성능의 균형을 고려한 결과다:

- **Row Group 레벨**: 단순한 통계 정보만 저장하여 메타데이터 크기 최소화
- **Page 레벨**: 정렬 정보 없이 데이터만 저장하여 중복 제거
- **Column Index 레벨**: 정렬 정보를 저장하여 Binary Search 가능
- **크기 제한**: 메타데이터 크기가 너무 커지는 것을 방지

### 정렬된 데이터의 필터링 과정

**age 컬럼이 하나의 Parquet 파일에서 정렬되어 있어도:**

1. **Row Group 레벨: 순차 탐색 (피할 수 없음)**
   - Row Group 통계(min/max)를 통해 순차적으로 검사
   - 정렬 여부와 관계없이 모든 Row Group을 하나씩 확인
   - 조건에 맞는 Row Group만 선택

2. **Page 레벨: Binary Search (정렬 효과)**
   - 선택된 Row Group 내부의 Column Index에서 `BoundaryOrder` 활용
   - ASCENDING/DESCENDING 정렬된 페이지들을 Binary Search로 효율적 탐색
   - 정렬되지 않은 페이지들은 순차 탐색

**핵심**: 정렬의 효과는 Row Group을 스킵하는 것이 아니라, 통과한 Row Group 내부의 페이지 필터링에서 나타납니다.

- **Row Group 스킵**: 통계 기반 (정렬과 무관)
- **Page 스킵**: BoundaryOrder 기반 (정렬 효과)

### 실제 활용 시 고려사항

정렬된 데이터의 이런 특성을 활용할 때는 다음을 고려해야 한다:

1. **Column Index 생성 여부**: 크기 제한으로 인해 생성되지 않을 수 있다
2. **페이지 크기 조정**: 너무 큰 페이지는 Column Index 생성 실패의 원인이 될 수 있다
3. **String 컬럼 주의**: 긴 String은 min/max 크기를 크게 만들어 Column Index 생성에 실패할 수 있다

정렬된 데이터의 이런 특성을 활용하면 데이터 웨어하우스나 빅데이터 환경에서 쿼리 성능을 개선할 수 있다. 하지만 단순히 "정렬하면 빠르다"가 아니라, 그 뒤에 숨겨진 기술적 세부사항을 이해하는 것이 중요하다.