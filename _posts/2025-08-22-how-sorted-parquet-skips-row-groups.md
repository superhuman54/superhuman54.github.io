---
layout: post
title: "정렬된 Parquet는 어떻게 Row Group을 스킵하는가"
date: 2025-08-22 12:00:00 +0900
categories: [Data Engineering, Parquet, Performance]
tags: [parquet, spark, performance, binary-search, push-down]
---

## 개요

Parquet 파일에서 정렬된 컬럼을 사용하면 쿼리 성능을 크게 향상시킬 수 있습니다. 이 글에서는 정렬된 Parquet 파일이 어떻게 Row Group을 효율적으로 스킵하는지, 그리고 그 뒤에 숨겨진 Binary Search 알고리즘에 대해 자세히 살펴보겠습니다.

## Parquet 파일 구조와 Row Group

Parquet 파일은 다음과 같은 계층 구조를 가집니다:

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

각 Row Group은 독립적으로 처리될 수 있으며, 이는 병렬 처리와 필터링 최적화의 핵심입니다.

## 정렬된 데이터의 BoundaryOrder

Parquet에서 컬럼이 정렬되어 있는지 여부는 `BoundaryOrder` enum으로 표현됩니다:

```java
// parquet-column/src/main/java/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java
public enum BoundaryOrder {
  UNORDERED,    // 정렬되지 않음
  ASCENDING,    // 오름차순 정렬
  DESCENDING    // 내림차순 정렬
}
```

## BoundaryOrder 계산 과정

### 1. Column Index 생성 시점

`BoundaryOrder`는 Column Index가 생성될 때 자동으로 계산됩니다:

```java
// parquet-column/src/main/java/org/apache/parquet/internal/column/columnindex/ColumnIndexBuilder.java
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

### 2. 정렬 판단 로직

```java
// parquet-column/src/main/java/org/apache/parquet/internal/column/columnindex/ColumnIndexBuilder.java
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

## ASCENDING 정렬된 Row Group에서 Binary Search

### 1. gt (greater than) 연산

```java
// parquet-column/src/main/java/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java
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

### 2. 실제 예시

ASCENDING 정렬된 age 컬럼이 있다고 가정해보겠습니다:

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

**최종 결과**: Row Group 2, 3, 4만 읽기 (Row Group 0, 1은 스킵)

### 3. lt (less than) 연산

```java
// parquet-column/src/main/java/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java
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

## DESCENDING 정렬된 Row Group에서 역순 Binary Search

### 1. gt (greater than) 연산

```java
// parquet-column/src/main/java/org/apache/parquet/internal/column/columnindex/BoundaryOrder.java
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

### 2. 실제 예시

DESCENDING 정렬된 age 컬럼이 있다고 가정해보겠습니다:

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

**최종 결과**: Row Group 0, 1만 읽기 (Row Group 2, 3, 4는 스킵)

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
// parquet-hadoop/src/main/java/org/apache/parquet/hadoop/ParquetFileWriter.java
if (columnIndexBuilder.getMinMaxSize() > columnIndexBuilder.getPageCount() * MAX_STATS_SIZE) {
  currentColumnIndexes.add(null);  // Column Index 생성 안함
} else {
  currentColumnIndexes.add(columnIndexBuilder.build());  // Column Index 생성
}
```

Column Index의 크기가 4KB × 페이지 수를 초과하면 생성되지 않습니다.

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

## 결론

정렬된 Parquet 파일은 Binary Search 알고리즘을 활용하여 Row Group을 효율적으로 스킵할 수 있습니다. 이는 특히 대용량 데이터에서 쿼리 성능을 크게 향상시킵니다.

### 핵심 포인트
1. **ASCENDING 정렬**: 오름차순으로 Binary Search 수행
2. **DESCENDING 정렬**: 내림차순으로 Binary Search 수행  
3. **성능 향상**: O(n) → O(log n) 시간 복잡도 개선
4. **실용적 적용**: Spark에서 `orderBy()` 후 Parquet 저장

정렬된 데이터의 이러한 특성을 활용하면 데이터 웨어하우스나 빅데이터 환경에서 쿼리 성능을 크게 개선할 수 있습니다.