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

정렬된 Parquet 파일을 사용하면 쿼리 성능이 크게 향상된다는 건 알고 있지만, 정말 궁금한 건 어떻게 그게 가능한지다. 분명히 어떤 Row Group들은 조건에 맞지 않아서 skip될 텐데, 어떤 메타데이터 덕분에 그런 판단이 가능했을까? 이 글에서는 정렬된 Parquet 파일이 어떻게 Row Group을 효율적으로 스킵하는지, 그리고 그 뒤에 숨겨진 Binary Search 알고리즘을 자세히 살펴보자. 단, 여기서는 쿼리 엔진이 이미 정렬된 데이터를 가정하고 있다.

<!-- more -->

## 개요

정렬된 Parquet 파일은 Binary Search 알고리즘을 활용해서 불필요한 Row Group을 스킵함으로써 쿼리 성능을 크게 향상시킨다. 이 메커니즘의 핵심은 `BoundaryOrder`와 Column Index를 통한 효율적인 필터링이다. 쿼리 엔진은 이미 정렬된 데이터를 전제로 하여 Binary Search를 수행한다.

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

## ASCENDING 정렬된 Row Group에서 Binary Search

쿼리 엔진은 ASCENDING 정렬된 Row Group에서 Binary Search를 수행할 때, 이미 정렬된 데이터임을 전제로 한다. 이 가정 하에서 효율적인 스킵이 가능하다.

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

쿼리 엔진은 DESCENDING 정렬된 Row Group에서도 마찬가지로 이미 정렬된 데이터임을 가정하고 역순 Binary Search를 수행한다.

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

## 실제 성능 향상 효과

### 1. 정렬되지 않은 데이터
- **필터링 방식**: 모든 Row Group을 순차적으로 검사
- **시간 복잡도**: O(n)
- **예시**: 100개 Row Group에서 `age > 120` 검색 시 모든 Row Group 검사

### 2. 정렬된 데이터
- **필터링 방식**: Binary Search로 효율적 검색
- **시간 복잡도**: O(log n)
- **예시**: 100개 Row Group에서 `age > 120` 검색 시 약 7개 Row Group만 검사

### 3. 성능 비교

| 데이터 크기 | 정렬되지 않은 데이터 | 정렬된 데이터 | 성능 향상 |
|------------|-------------------|-------------|----------|
| 1,000 Row Groups | 1,000 검사 | ~10 검사 | 100배 |
| 10,000 Row Groups | 10,000 검사 | ~13 검사 | 770배 |
| 100,000 Row Groups | 100,000 검사 | ~17 검사 | 5,880배 |

## 마무리

정렬된 Parquet 파일은 Binary Search 알고리즘을 활용해서 Row Group을 효율적으로 스킵할 수 있다. 이는 특히 대용량 데이터에서 쿼리 성능을 크게 향상시킨다. 쿼리 엔진이 정렬된 데이터를 가정하고 Binary Search를 수행하기 때문에 O(log n) 시간 복잡도로 효율적인 필터링이 가능하다.

### 핵심 포인트
1. **ASCENDING 정렬**: 오름차순으로 Binary Search 수행
2. **DESCENDING 정렬**: 내림차순으로 Binary Search 수행  
3. **성능 향상**: O(n) → O(log n) 시간 복잡도 개선
4. **쿼리 엔진 가정**: 정렬된 데이터를 전제로 Binary Search 수행
5. **실용적 적용**: Spark에서 `orderBy()` 후 Parquet 저장
6. **Column Index 제한**: 크기 제한으로 인한 BoundaryOrder 생성 실패 가능성 고려

정렬된 데이터의 이런 특성을 활용하면 데이터 웨어하우스나 빅데이터 환경에서 쿼리 성능을 크게 개선할 수 있다.