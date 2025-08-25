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