## BoundaryOrder는 Column Index에만 존재

**중요한 점**: `BoundaryOrder`는 Parquet 포맷 스펙에서 정의된 enum으로, **Column Index에만 존재**한다. Row Group 레벨이나 Page 레벨에는 정렬 정보가 저장되지 않는다.

```java
public enum BoundaryOrder implements org.apache.thrift.TEnum {
  UNORDERED(0),
  ASCENDING(1), 
  DESCENDING(2);
}
```

<div class="code-footer">
  <span class="file-path">parquet-format-structures/target/generated-sources/thrift/org/apache/parquet/format/BoundaryOrder.java</span>
</div>