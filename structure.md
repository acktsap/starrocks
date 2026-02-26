# StarRocks Storage Architecture

> Tablet → Rowset → Segment → Column / Index 논리 계층 + 물리 파일 구조

---

## Part 1 — 논리 계층 구조 (Logical Hierarchy)

### 1. Tablet

> `be/src/storage/tablet.h`

테이블 파티션의 **물리적 데이터 단위**. DataDir에 소속되며, 여러 Rowset을 버전별로 관리한다.

| 핵심 멤버 | 타입 | 설명 |
|-----------|------|------|
| `_tablet_meta` | `TabletMetaSharedPtr` | 스키마, 상태, 버전 정보 |
| `_rs_version_map` | `map<Version, RowsetSharedPtr>` | 버전 → Rowset 매핑 |
| `_inc_rs_version_map` | `unordered_map<Version, RowsetSharedPtr>` | 증분 Rowset (clone 용) |
| `_updates` | `unique_ptr<TabletUpdates>` | Primary Key 테이블 전용 업데이트 관리 |
| `_keys_type` | enum | `DUP_KEYS` / `UNIQUE_KEYS` / `PRIMARY_KEYS` / `AGG_KEYS` |
| `_cumulative_point` | `atomic<int64_t>` | Base / Cumulative compaction 경계 |

**상태 전이:**

```
NOTREADY → RUNNING → TOMBSTONED → STOPPED → SHUTDOWN
```

**동시성 제어 (Locks):**

| Lock | 용도 |
|------|------|
| `_meta_lock` | 메타데이터 접근 |
| `_base_lock` | Base compaction |
| `_cumulative_lock` | Cumulative compaction |
| `_ingest_lock` | 데이터 ingestion |
| `_schema_lock` | 스키마 변경 |
| `_migration_lock` | Tablet 마이그레이션 |

---

### 2. Rowset

> `be/src/storage/rowset/rowset.h`

한 번의 load 또는 compaction으로 생성되는 **불변(immutable) 데이터 단위**.

| 핵심 멤버 | 타입 | 설명 |
|-----------|------|------|
| `_rowset_meta` | `RowsetMetaSharedPtr` | 버전, 행 수, 디스크 크기 등 |
| `_segments` | `vector<SegmentSharedPtr>` | 세그먼트 목록 |
| `_schema` | `TabletSchemaCSPtr` | 이 Rowset 생성 시점의 스키마 스냅샷 |
| `_refs_by_reader` | `atomic<uint64_t>` | 읽기 중인 reader 참조 카운트 |
| `_rowset_state_machine` | `RowsetStateMachine` | 상태 관리 |

**RowsetMeta 주요 필드** (`be/src/storage/rowset/rowset_meta.h`):

| 필드 | 설명 |
|------|------|
| `rowset_id` | 고유 식별자 |
| `version [start, end]` | 버전 범위 (e.g., [0, 100]) |
| `num_rows` | 총 행 수 |
| `total_disk_size` | 디스크 사용량 |
| `num_segments` | 세그먼트 파일 수 |
| `delete_predicate` | 삭제 조건 |

**상태 전이:**

```
UNLOADED → LOADED → UNLOADING → UNLOADED
```

**설계 원칙:**
- 생성 후 절대 수정 불가 (Immutable)
- Compaction → 새 Rowset 생성, 기존 Rowset 삭제

---

### 3. Segment

> `be/src/storage/rowset/segment.h`

**하나의 물리적 디스크 파일**. 컬럼별 데이터와 인덱스를 담는다.

| 핵심 멤버 | 타입 | 설명 |
|-----------|------|------|
| `_column_readers` | `map<int32_t, unique_ptr<ColumnReader>>` | 컬럼 unique_id별 리더 |
| `_num_rows` | `uint32_t` | 이 세그먼트의 총 행 수 |
| `_sk_index_decoder` | `unique_ptr<ShortKeyIndexDecoder>` | Short Key 인덱스 디코더 |
| `_short_key_index_page` | `PagePointer` | Short Key 인덱스 파일 내 위치 |
| `_segment_id` | `uint32_t` | Rowset 내 세그먼트 번호 |

**주요 API:**

| 메서드 | 설명 |
|--------|------|
| `open()` | 세그먼트 메타데이터(footer) 로드 |
| `load_index()` | Short Key 인덱스 메모리 로드 (thread-safe, 1회) |
| `new_iterator()` | 행 단위 이터레이터 생성 |
| `new_column_iterator()` | 컬럼 단위 이터레이터 생성 |

**Lazy Loading:** 인덱스는 필요할 때 한 번만 로드 (thread-safe once initialization)

---

### 4. ColumnReader

> `be/src/storage/rowset/column_reader.h`

컬럼 단위 데이터 접근. 각 컬럼이 자체 인덱스를 소유한다.

| 핵심 멤버 | 타입 | 설명 |
|-----------|------|------|
| `_encoding_info` | `EncodingInfo*` | Plain / RLE / Dictionary 등 |
| `_column_type` | `LogicalType` | INT, VARCHAR, ... |
| `_dict_page_pointer` | `PagePointer` | Dictionary 페이지 위치 |
| `_ordinal_index` | `unique_ptr<OrdinalIndexReader>` | 행 번호 → 페이지 매핑 |
| `_zonemap_index` | `unique_ptr<ZoneMapIndexReader>` | 페이지별 min/max |
| `_bitmap_index` | `unique_ptr<BitmapIndexReader>` | 딕셔너리 + 비트맵 |
| `_bloom_filter_index` | `unique_ptr<BloomFilterIndexReader>` | 블룸 필터 |
| `_segment_zone_map` | `unique_ptr<ZoneMapPB>` | 세그먼트 전체 min/max |

---

### 5. Index 구조

쿼리 시 불필요한 데이터를 pruning하여 I/O를 최소화한다.

#### 5.1 Short Key Index

> `be/src/storage/short_key_index.h`

| 항목 | 내용 |
|------|------|
| **용도** | 키 컬럼 prefix로 범위 탐색 (B-tree 유사) |
| **구조** | 행 블록마다 첫 번째 키의 prefix 저장 |
| **API** | `lower_bound()`, `upper_bound()` |
| **키 마커** | `0x00`=MIN, `0x01`=NULL↑, `0x02`=NORMAL, `0xFE`=NULL↓, `0xFF`=MAX |

#### 5.2 Ordinal Index

> `be/src/storage/rowset/ordinal_page_index.h`

| 항목 | 내용 |
|------|------|
| **용도** | 행 번호(ordinal) → 데이터 페이지 매핑 |
| **구조** | `_ordinals[i]` = i번째 페이지의 첫 행 번호, `_pages[i]` = 페이지 오프셋 |
| **탐색** | 이진 탐색으로 특정 행의 페이지 위치 결정 |

#### 5.3 Zone Map Index

> `be/src/storage/rowset/zone_map_index.h`

| 항목 | 내용 |
|------|------|
| **용도** | 페이지/세그먼트별 min/max 통계 → 값 범위 pruning |
| **구조** | 페이지별 `{min, max, has_null}`, 세그먼트 전체 글로벌 min/max |
| **효과** | WHERE 조건이 범위 밖이면 페이지/세그먼트 통째로 스킵 |

#### 5.4 Bitmap Index

> `be/src/storage/rowset/bitmap_index_reader.h`

| 항목 | 내용 |
|------|------|
| **용도** | 등호(=) / IN 조건에서 빠른 필터링 |
| **구조** | 딕셔너리(고유 값 목록) + 각 값에 대한 Roaring Bitmap |
| **추가** | N-gram 지원 (substring 검색), NULL 전용 bitmap 별도 관리 |

#### 5.5 Bloom Filter Index

> `be/src/storage/rowset/bloom_filter_index_reader.h`

| 항목 | 내용 |
|------|------|
| **용도** | "이 값이 이 블록에 존재하는가?" 확률적 빠른 체크 |
| **구조** | 블록 단위 Bloom Filter (`BLOCK_BLOOM_FILTER`) |
| **해시** | `HASH_MURMUR3_X64_64` |
| **특성** | False Positive 가능, False Negative 없음. N-gram 지원 |

---

### 전체 계층 관계도

```
Tablet
│  _tablet_meta (TabletMetaPB)
│  _rs_version_map: Version → Rowset
│
├── Rowset_1 [version 0-100]     ← Immutable
│   │  _rowset_meta (RowsetMetaPB)
│   │  _schema (스키마 스냅샷)
│   │
│   ├── Segment_0 (.dat 파일)
│   │   │  _num_rows, _segment_id
│   │   │
│   │   ├── ColumnReader_0  (encoding + type)
│   │   │   ├── OrdinalIndex     ordinal → page
│   │   │   ├── ZoneMapIndex     min/max per page
│   │   │   ├── BitmapIndex      dict + roaring bitmap
│   │   │   └── BloomFilterIndex block bloom filter
│   │   │
│   │   ├── ColumnReader_1
│   │   │   └── (같은 인덱스 구조)
│   │   │
│   │   └── Short Key Index (키 prefix 기반 탐색)
│   │
│   ├── Segment_1
│   └── ...
│
├── Rowset_2 [version 101-200]
└── ...
```

---

## Part 2 — 물리 파일 구조 (Physical File Layout)

> 디스크에 실제로 기록되는 바이트 수준 구조
> 기반: `segment.proto`, `olap_file.proto`, `page_io.cpp`, `segment_writer.cpp`

---

### 2-1. 디스크 디렉토리 레이아웃

```
data_dir/
├── tablet_meta.pb                  ← TabletMetaPB: schema, rs_metas[], updates
├── {rowset_id}_0.dat               ← Rowset 0, Segment 0
├── {rowset_id}_1.dat               ← Rowset 0, Segment 1
├── {rowset_id2}_0.dat              ← Rowset 1, Segment 0
├── {rowset_id2}_0.cols             ← PK 인덱스 (Primary Key 테이블)
└── ...
```

**파일 네이밍 규칙:**

| 파일 | 패턴 |
|------|------|
| 세그먼트 | `{rowset_id}_{segment_id}.dat` |
| 메타데이터 | `tablet_meta.pb` |
| PK 인덱스 | `{rowset_id}_{segment_id}.cols` |

**Rowset → Segment 매핑:**
- `RowsetMetaPB.num_segments = N` → N개의 `.dat` 파일이 연속 segment_id(0, 1, ..., N-1)로 생성

---

### 2-2. 세그먼트 파일 바이트 레이아웃

> Magic: `"D0R1"` (0x44303152)

```
Offset      Content
──────────────────────────────────────────────────────────
0x0000      ┌─────────────────────────────────────────┐
            │  Column 0 — Data Pages                  │
            │  (Page 0 | Page 1 | ... | Page N)       │
            ├─────────────────────────────────────────┤
            │  Column 0 — Ordinal Index + ZoneMap     │
            ├─────────────────────────────────────────┤
            │  Column 1 — Data Pages                  │
            ├─────────────────────────────────────────┤
            │  Column 1 — Ordinal Index + ZoneMap     │
            ├─────────────────────────────────────────┤
            │  ...                                    │
            ├─────────────────────────────────────────┤
            │  Column N — Data Pages + Index Pages    │
            ├─────────────────────────────────────────┤
            │  Bitmap Index Pages | Bloom Filter Pages│
            ├─────────────────────────────────────────┤
            │  Short Key Index Page                   │
            ├─────────────────────────────────────────┤
Y           │  SegmentFooterPB (Protobuf 직렬화)       │
            ├──────────┬──────────┬───────────────────┤
Y+K         │Footer    │Checksum  │Magic "D0R1"       │
            │Size(u32) │CRC32C    │0x44303152          │
            │LE 4bytes │LE 4bytes │LE 4bytes           │
            └──────────┴──────────┴───────────────────┘
                                                    EOF
```

**파일 읽기 절차 (Read Path):**

1. 파일 끝에서 **12바이트** 읽기 → Magic `"D0R1"` 검증
2. Footer Size (4B) → SegmentFooterPB 역직렬화
3. **CRC32C** 체크섬 검증 (Footer 데이터 대상)
4. ColumnMetaPB → PagePointer로 필요한 페이지만 I/O
5. Short Key Index → 키 기반 범위 탐색 시작

---

### 2-3. SegmentFooterPB 구조

> `gensrc/proto/segment.proto`

```protobuf
message SegmentFooterPB {
    optional uint32 version = 1;                     // 파일 포맷 버전
    repeated ColumnMetaPB columns = 2;               // 컬럼별 메타데이터 배열
    optional uint32 num_rows = 3;                    // 세그먼트 총 행 수
    optional PagePointerPB short_key_index_page = 9; // Short Key 인덱스 위치
}
```

**ColumnMetaPB:**

```protobuf
message ColumnMetaPB {
    optional uint32 column_id = 1;         // 컬럼 ID
    optional uint32 unique_id = 2;         // 고유 컬럼 ID
    optional int32 type = 3;               // LogicalType 열거형
    optional EncodingTypePB encoding = 5;  // 인코딩 방식
    optional CompressionTypePB compression = 6;  // 압축 방식
    optional bool is_nullable = 7;
    repeated ColumnIndexMetaPB indexes = 8;  // ZoneMap, Bitmap, BloomFilter
    optional PagePointerPB dict_page = 9;    // Dictionary 페이지 포인터
    optional uint64 num_rows = 11;
}
```

---

### 2-4. 페이지 내부 구조 (Page Format)

> `page_io.cpp` / `parsed_page.h` — 모든 페이지 공통 포맷

```
┌──────────────────────────────────────────────────┐
│  Page Body (인코딩 + 압축된 데이터)                 │
│  압축 시: body_size < uncompressed_size            │
├──────────────────────────────────────────────────┤
│  PageFooterPB (Protobuf 직렬화)                    │
├────────────────────┬─────────────────────────────┤
│  Footer Size       │  Checksum CRC32C             │
│  (uint32 LE)       │  (uint32 LE)                 │
└────────────────────┴─────────────────────────────┘
```

**체크섬 범위:** Body + FooterPB + Footer Size (체크섬 자체 제외)

**PageFooterPB:**

```protobuf
message PageFooterPB {
    optional PageTypePB type = 1;           // DATA_PAGE / INDEX_PAGE /
                                            // DICTIONARY_PAGE / SHORT_KEY_PAGE
    optional uint32 uncompressed_size = 2;  // 압축 전 크기
    optional DataPageFooterPB data_page_footer = 7;
    optional IndexPageFooterPB index_page_footer = 8;
}
```

**PagePointer** (`page_pointer.h`):

```cpp
class PagePointer {
    uint64_t offset;  // 세그먼트 파일 내 절대 바이트 오프셋
    uint32_t size;    // 페이지 전체 크기 (footer + checksum 포함)
};
// 직렬화: varint64(offset) + varint32(size)
```

---

### 2-5. Data Page Body 내부 구조

> format version 2

```
┌──────────────────────────────────────────────────┐
│  Encoded Values                                  │
│  (인코딩 방식에 따라 다른 바이너리 포맷)              │
├──────────────────────────────────────────────────┤
│  Null Bitmap (Nullable 컬럼만)                    │
│  (BITSHUFFLE / RLE / LZ4 중 하나로 압축)           │
└──────────────────────────────────────────────────┘
```

**DataPageFooterPB:**

```protobuf
message DataPageFooterPB {
    optional uint64 first_ordinal = 1;      // 이 페이지 첫 행 번호
    optional uint64 num_values = 2;         // 행 수 (NULL 포함)
    optional uint32 nullmap_size = 3;       // NULL 비트맵 크기
    optional uint32 format_version = 20;    // 1 or 2
    optional NullEncodingPB null_encoding = 21;  // BITSHUFFLE / RLE / LZ4
}
```

**참고:**
- Non-null 컬럼: Encoded Values만 존재 (Null Bitmap 생략)
- Dict-encoded 컬럼: 별도 `DICTIONARY_PAGE`에 사전 저장, 값은 코드 배열로 참조

---

### 2-6. 인코딩 타입 (EncodingTypePB)

> `encoding_info.h` / `segment.proto`

| 코드 | 이름 | 설명 | 주 사용처 |
|------|------|------|-----------|
| 2 | `PLAIN` | 원본 그대로 저장 | 문자열 기본 |
| 3 | `PREFIX` | 앞 키와의 공통 prefix 제거 | 정렬된 문자열 |
| 4 | `RLE` | 연속 같은 값 → (값, 횟수) | Boolean, 낮은 카디널리티 |
| 5 | `DICT` | 사전 + 코드 배열 | 카디널리티 낮은 문자열 |
| 6 | `BIT_SHUFFLE` | 비트 재배치 후 LZ4 | 숫자형 기본 |
| 7 | `FOR` | Frame-of-Reference, 델타 인코딩 | 정수 시퀀스 |

**압축 (CompressionTypePB):**

| 타입 | 비고 |
|------|------|
| `LZ4_FRAME` | 기본값 |
| `SNAPPY` | |
| `ZSTD` | 높은 압축률 |
| `GZIP` | |

**압축 판단 기준:**

```
space_saving = 1.0 - (compressed_size / uncompressed_size)
→ space_saving ≥ min_saving (0.1 = 10%) 이면 압축 적용, 아니면 원본 저장
```

---

### 2-7. Binary Plain Page (문자열 인코딩)

> `binary_plain_page.h` — `PLAIN_ENCODING` 시 문자열 저장 방식

```
┌───────────────────────┬────────────────────┬──────────┐
│  Raw String Data      │  Offset Array      │  Count   │
│  (연속 바이트)          │  (uint32 × N)      │  (u32)   │
└───────────────────────┴────────────────────┴──────────┘
```

- **읽기:** `offset[i]` ~ `offset[i+1]` 사이가 i번째 문자열
- **장점:** 가변 길이 문자열을 O(1) 랜덤 액세스

---

### 2-8. 인덱스 페이지 물리 구조

#### Short Key Index Page

> `ShortKeyFooterPB`

```
┌──────────────────────────────────────────────────┐
│  Key Content (raw binary keys — prefix 인코딩)    │
├──────────────────────────────────────────────────┤
│  Offset Array (varint32 per key → 각 키 시작 위치) │
└──────────────────────────────────────────────────┘
```

| 키 마커 | 값 | 의미 |
|---------|-----|------|
| `KEY_MINIMAL_MARKER` | `0x00` | 최솟값 |
| `KEY_NULL_FIRST_MARKER` | `0x01` | NULL (nulls first) |
| `KEY_NORMAL_MARKER` | `0x02` | 일반 값 |
| `KEY_NULL_LAST_MARKER` | `0xFE` | NULL (nulls last) |
| `KEY_MAXIMAL_MARKER` | `0xFF` | 최댓값 |

- `num_rows_per_block` 행마다 Short Key 엔트리 1개 생성
- `SegmentFooterPB.short_key_index_page` → PagePointer로 참조

#### Ordinal Index (B-Tree Page)

> `BTreeMetaPB`

```
IndexEntry × N개 반복:
┌─────────────────┬──────────┬────────────────────┬────────────────────┐
│  Key Len        │  Key     │  Page Offset       │  Page Size         │
│  (varint32)     │  (bytes) │  (varint64)        │  (varint32)        │
└─────────────────┴──────────┴────────────────────┴────────────────────┘
```

**B-Tree 구조:**
- Root Page → Internal Pages → Leaf Pages → Data Pages
- 데이터 페이지 1개뿐이면 `is_root_data_page = true` (root가 곧 data page)
- 이진 탐색으로 ordinal → page offset 결정
- `IndexPageFooterPB.type`: `LEAF(1)` / `INTERNAL(2)`

#### Zone Map Index

| 레벨 | 저장 위치 | 수량 |
|------|-----------|------|
| 세그먼트 레벨 | ColumnMetaPB 안에 직접 포함 | 1개 |
| 페이지 레벨 | IndexedColumn으로 별도 페이지 저장 | N개 (페이지 수만큼) |

```protobuf
message ZoneMapPB {
    optional bytes min = 1;          // 인코딩된 최솟값
    optional bytes max = 2;          // 인코딩된 최댓값
    optional bool has_null = 3;
    optional bool has_not_null = 4;
}
```

#### Bitmap Index / Bloom Filter

| 인덱스 | 물리 저장 |
|--------|-----------|
| **Bitmap** | IndexedColumn × 2: `dict_column` (고유 값 정렬 배열) + `bitmap_column` (Roaring Bitmap) |
| **Bloom Filter** | IndexedColumn × 1: 페이지별 블룸 필터 바이트 배열 |

---

### 2-9. Write / Read 데이터 흐름

#### Write Path

> `segment_writer.cpp` → `column_writer.cpp` → `page_io.cpp`

```
Raw Values → Encode → Compress → PageFooterPB → CRC32C + Write → .dat File
```

1. 원본 값을 인코딩 (PLAIN / DICT / BIT_SHUFFLE / ...)
2. 인코딩된 데이터를 압축 (LZ4_FRAME / SNAPPY / ZSTD / GZIP)
3. PageFooterPB 생성 (type, uncompressed_size, first_ordinal, ...)
4. CRC32C 체크섬 계산 (body + footer + footer_size)
5. `[Body][FooterPB][Footer Size(4B)][CRC32C(4B)]` 순서로 파일에 기록

#### Read Path

> `page_io.cpp` → `column_reader.cpp` → `segment.cpp`

```
PagePointer → Read + CRC → Parse Footer → Decompress → Decode → Column Data
```

1. PagePointer(offset, size)로 파일에서 페이지 읽기
2. 끝 4바이트에서 체크섬 추출, CRC32C 검증
3. Footer Size → PageFooterPB 역직렬화
4. `body_size < uncompressed_size`이면 Decompress
5. 인코딩 타입에 맞는 PageDecoder로 디코딩
6. Column 데이터로 반환

---

## Part 3 — Data Ingestion (쓰기 흐름)

> FE로부터 데이터를 받아 디스크에 세그먼트 파일로 기록하고 버전을 발행하는 전체 파이프라인

---

### 3-1. 전체 파이프라인 개요

```
FE Load Request (RPC)
    │
    ▼
LoadChannel              runtime/load_channel.h
    │  open() → 스키마 생성, Chunk 역직렬화
    ▼
TabletsChannel           runtime/tablets_channel.h
    │  tablet별 DeltaWriter 생성/라우팅
    ▼
DeltaWriter              storage/delta_writer.h  |  storage/lake/delta_writer.h
    │  write(chunk) → MemTable에 삽입
    ▼
MemTable                 storage/memtable.h
    │  버퍼링 → 정렬 → 집계 → flush 트리거
    ▼
TabletWriter             storage/lake/tablet_writer.h
    │  Chunk를 SegmentWriter에 전달
    ▼
SegmentWriter            storage/rowset/segment_writer.h
    │  컬럼별 페이지 생성 → 인덱스 생성 → Footer 기록
    ▼
.dat Segment File        디스크에 기록 완료
    │
    ▼
Publish Version          버전 발행 → 데이터 가시화
```

---

### 3-2. 각 컴포넌트 상세

#### LoadChannel (`runtime/load_channel.h`)

- FE로부터 RPC를 받는 진입점
- 주요 메서드: `open()`, `add_chunk()`, `add_chunks()`, `add_segment()`, `cancel()`
- Chunk 데이터를 protobuf에서 역직렬화
- load_id + sink_id + index_id 조합으로 TabletsChannel 관리
- 메모리 트래커로 load 리소스 제한

#### TabletsChannel (`runtime/tablets_channel.h`)

- 하나의 load job 내 여러 tablet에 대한 쓰기를 관리
- tablet별로 DeltaWriter 1개 생성
- 라이프사이클: open → incremental_open → add_chunk → abort/cancel

#### DeltaWriter (`storage/delta_writer.h`, `storage/lake/delta_writer.h`)

한 tablet에 대한 쓰기를 조율하는 핵심 컴포넌트.

| 멤버 | 설명 |
|------|------|
| `_tablet_writer` | TabletWriter (실제 세그먼트 생성) |
| `_mem_table` | MemTable (메모리 버퍼) |
| `_mem_table_sink` | MemTable → TabletWriter 연결 |
| `_flush_token` | 비동기 flush 작업 큐 |

상태 전이:

```
kUninitialized → kWriting → kClosed → kCommitted
                     │
                   (error)
                     ▼
                  kAborted
```

라이프사이클:

1. `open()` — tablet 메타 로드, TabletWriter 생성
2. `write(chunk)` — MemTable에 삽입, 가득 차면 flush 트리거
3. `finish_with_txnlog()` — 잔여 데이터 flush, TxnLogPB 생성
4. `close()` — 정리

#### MemTable (`storage/memtable.h`)

메모리에 데이터를 버퍼링하고 정렬/집계 후 flush하는 컴포넌트.

| 멤버 | 설명 |
|------|------|
| `_chunk` | 현재 버퍼링 중인 Chunk |
| `_result_chunk` | 처리(정렬/집계) 완료된 결과 Chunk |
| `_aggregator` | DUP_KEYS 중복 제거용 |
| `_max_buffer_size` | 최대 메모리 크기 (config: `write_buffer_size`) |
| `_max_buffer_row` | 최대 행 수 |

flush 트리거 조건:

- 메모리 사용량이 `_max_buffer_size` 초과
- 행 수가 `_max_buffer_row` 초과
- 명시적 flush 호출 또는 EOS (End of Stream)

flush 전 데이터 처리:

1. Merge — 여러 insert를 병합
2. Sort — sort key 기준 정렬
3. Aggregate — AGG 테이블: 중복 키 집계
4. Split upserts/deletes — PK 테이블: upsert/delete 분리

#### SegmentWriter (`storage/rowset/segment_writer.h`)

개별 세그먼트 파일을 생성하는 컴포넌트.

```
init(column_indexes, has_key)      // 세그먼트 초기화
    ▼
append_chunk(chunk)                // 행 추가 (반복)
    ▼
finalize_columns(index_size)       // 컬럼 데이터 + 인덱스 기록
    ▼
finalize_footer(file_size)         // SegmentFooterPB 기록 + Magic
```

- 컬럼별 writer가 페이지 생성
- 블록 단위 압축 적용
- Short Key / Zone Map / Bloom Filter 인덱스 동시 생성

---

### 3-3. Publish / Commit (버전 발행)

데이터가 쓰여진 후 읽기 가능하게 만드는 과정.

#### Lake (Cloud-Native) 경로

```
DeltaWriter.finish_with_txnlog()
    │  TxnLogPB 생성 (모든 세그먼트 파일 목록)
    ▼
FE가 전체 replica에 publish 조율
    ▼
publish_version()                  storage/lake/transactions.h
    │  base metadata (version N) 로드
    │  TxnLogPB의 세그먼트 추가 적용
    │  새 metadata (version N+1) 생성
    │  object storage에 기록
    ▼
데이터 version N+1에서 가시화
```

#### Non-Lake (Shared-Nothing) 경로

```
DeltaWriter.commit()
    │  RowsetWriter.build() → Rowset 객체 생성
    ▼
tablet->add_rowset(rowset)
    │  _rs_version_map에 추가
    │  VersionGraph 업데이트
    ▼
TxnManager에 트랜잭션 커밋
    ▼
데이터 가시화
```

---

### 3-4. 메모리 관리와 흐름 제어

메모리 트래커 계층:

```
LoadChannel → TabletsChannel → DeltaWriter → MemTable
```

- 각 레벨에서 소비량 추적 및 한도 적용
- 메모리 압박 시 대기(wait) + 강제 flush

Spill 지원 (대용량 load):

- 메모리 초과 시 `LoadSpillBlockManager`를 통해 디스크에 spill
- `merge_blocks_to_segments()`로 spill 블록을 세그먼트로 병합

---

## Part 4 — Data Read (읽기 흐름)

> 쿼리 시 버전 스냅샷 캡처 → 인덱스 pruning → 컬럼 읽기 → 병합의 전체 과정

---

### 4-1. 전체 스캔 파이프라인

```
Query Execution (FE → BE)
    │
    ▼
TabletReader 생성            storage/tablet_reader.h
    │  prepare() → capture_consistent_rowsets(version)  ← MVCC 스냅샷
    │  open(params) → 이터레이터 스택 구성
    ▼
Merge/Union Iterator 스택
    │  keys_type에 따라 다른 전략
    ▼
SegmentIterator              storage/rowset/segment_iterator.cpp
    │  인덱스 평가 → 행 범위 결정 → 컬럼 읽기 → 필터링
    ▼
Chunk 반환 (최대 4096행)
```

---

### 4-2. TabletReader (`storage/tablet_reader.h`)

스캔의 최상위 조율자. `ChunkIterator`를 상속한다.

주요 메서드:

| 메서드 | 설명 |
|--------|------|
| `prepare()` | `capture_consistent_rowsets(version)` → MVCC 스냅샷 캡처 |
| `open(params)` | predicate 초기화, 이터레이터 스택 구성 |
| `do_get_next(chunk)` | 다음 Chunk 반환 |
| `get_segment_iterators()` | 세그먼트별 이터레이터 생성 |

내부 흐름:

```
prepare()
  └─ capture_consistent_rowsets(version)   // 이 버전의 rowset 집합 확보
  └─ rowset->load() → 세그먼트 메타 로드

open(params)
  └─ _init_collector()
      └─ get_segment_iterators()           // 세그먼트별 이터레이터 생성
      └─ keys_type에 따른 이터레이터 스택 구성
```

---

### 4-3. 이터레이터 병합 전략

keys_type에 따라 다른 이터레이터 스택을 구성한다.

| keys_type | 이터레이터 | 설명 |
|-----------|-----------|------|
| `PRIMARY_KEYS` | `UnionIterator` | Rowset 순서대로 연결 (delete vector로 중복 처리) |
| `DUP_KEYS` | `UnionIterator` | Rowset 순서대로 연결 (중복 허용) |
| `UNIQUE_KEYS` | `AggregateIterator(HeapMergeIterator)` | 키 기준 merge-sort + 중복 제거 |
| `AGG_KEYS` | `AggregateIterator(HeapMergeIterator)` | 키 기준 merge-sort + 집계 |

Compaction 전용:

| 모드 | 이터레이터 |
|------|-----------|
| Horizontal compaction | `HeapMergeIterator` |
| Vertical compaction | `MaskMergeIterator` (source mask 기반 병합) |

HeapMergeIterator:

- 다중 정렬 스트림의 힙 기반 merge-sort
- 키 컬럼만으로 비교
- 같은 키일 때 낮은 인덱스(=최신 버전) 우선

---

### 4-4. SegmentIterator 인덱스 평가 순서

> `segment_iterator.cpp` — `_init()` 내부

각 인덱스가 `SparseRange<>`를 반환하여 후보 행을 점진적으로 줄인다.

```
 순서   인덱스                          설명
──────────────────────────────────────────────────────────────
  1    _get_row_ranges_by_rowid_range()   명시적 rowid 범위 제약
  2    _get_row_ranges_by_keys()          SHORT KEY INDEX (lower/upper bound)
  3    _apply_tablet_range()              Tablet 범위 predicate
  4    _apply_del_vector()  [선택적]      Delete Vector (조기 적용 시)
  5    _apply_bitmap_index()              BITMAP INDEX (딕셔너리 기반)
  6    _get_row_ranges_by_zone_map()      ZONE MAP (min/max per page)
  7    _get_row_ranges_by_bloom_filter()  BLOOM FILTER (존재 여부)
  8    _apply_inverted_index()            INVERTED INDEX (텍스트 검색)
  9    _apply_del_vector()  [선택적]      Delete Vector (지연 적용 시)
 10    _get_row_ranges_by_vector_index()  VECTOR INDEX (ANN/유사도)
```

- 각 인덱스는 pessimistic — 필터 불가 시 전체 행 통과
- 후속 인덱스는 이전 결과와 교집합(intersect)
- `apply_del_vec_after_all_index_filter` config로 Delete Vector 적용 시점 제어

---

### 4-5. Predicate Pushdown

#### ConjunctivePredicates (`storage/conjunctive_predicates.h`)

```cpp
class ConjunctivePredicates {
    std::vector<const ColumnPredicate*> _vec_preds;      // 벡터화 가능
    std::vector<const ColumnPredicate*> _non_vec_preds;  // 비벡터화
};
```

#### ColumnPredicate 타입 (`storage/column_predicate.h`)

| 타입 | 용도 |
|------|------|
| `kEQ, kNE, kGT, kGE, kLT, kLE` | 비교 연산 |
| `kInList, kNotInList` | 집합 멤버십 |
| `kIsNull, kNotNull` | NULL 체크 |
| `kAnd, kOr` | 복합 predicate |
| `kExpr` | 표현식 기반 |

각 ColumnPredicate는 인덱스 필터 메서드를 제공:

- `zone_map_filter(detail)` — Zone Map과 비교
- `original_bloom_filter(bf)` — Bloom Filter 체크
- `seek_bitmap_dictionary(iter)` — Bitmap 인덱스 탐색

#### Predicate 평가 단계

1. 페이지 레벨 — 인덱스 기반 (Zone Map, Bloom Filter, Bitmap)
2. 블록/범위 읽기 — 인덱스 결과 기반
3. 컬럼 이터레이션 — 실제 페이지에서 데이터 읽기
4. 행 레벨 — Chunk 내 행별 predicate 적용

---

### 4-6. Delete Vector 처리 (Primary Key 테이블)

> `storage/del_vector.h`

```cpp
class DelVector {
    int64_t _version;
    std::shared_ptr<Roaring> _roaring;  // Roaring bitmap
};
```

- 각 rowset에 연관된 delete vector가 존재
- 논리적으로 삭제된 행의 rowid를 Roaring Bitmap으로 관리
- `_apply_del_vector()`에서 `_scan_range`에서 삭제된 rowid 제거

---

### 4-7. Late Materialization (지연 구체화)

- predicate 컬럼만 먼저 읽어 행 필터링
- 필터 통과한 행에 대해서만 나머지 컬럼 구체화
- `enable_predicate_col_late_materialize` config로 활성화

Dictionary Code 최적화:

- Dictionary 인코딩된 컬럼: predicate를 정수 코드로 변환
- 문자열 비교 대신 정수 비교로 필터링 → 큰 성능 향상

---

## Part 5 — Compaction + Rowset MVCC

> 다중 rowset 병합과 동시 읽기/쓰기를 가능케 하는 버전 관리 메커니즘

---

### 5-1. Rowset MVCC (Multi-Version Concurrency Control)

StarRocks의 핵심 동시성 메커니즘. Compaction 중에도 읽기가 일관된 스냅샷을 볼 수 있게 보장한다.

#### Version 구조 (`storage/olap_common.h`)

```cpp
struct Version {
    int64_t first;    // start version (inclusive)
    int64_t second;   // end version (inclusive)
};
```

- 각 Rowset은 연속적인 버전 범위를 가짐 (예: [0-1], [2-4], [5-5])
- 버전 번호는 FE가 순차적으로 할당
- 버전 범위 사이 빈틈(gap) 불허

#### 세 가지 버전 맵 (`storage/tablet.h`)

```
┌─────────────────────────────────────────────────────────────────┐
│  Tablet                                                         │
│                                                                 │
│  _rs_version_map          활성 rowset (현재 쿼리용)                │
│  map<Version, Rowset>     [0-50] [51-80] [81-100]              │
│                                                                 │
│  _stale_rs_version_map    Compaction 완료된 old rowset            │
│  unordered_map            [0-5] [6-10] ... (GC 대기)            │
│                                                                 │
│  _inc_rs_version_map      증분 rowset (clone 용)                 │
│  unordered_map            최근 rowset만 유지                      │
└─────────────────────────────────────────────────────────────────┘
```

| 맵 | 용도 | 업데이트 시점 |
|----|------|-------------|
| `_rs_version_map` | 활성 rowset — 새 쿼리가 사용 | rowset 추가/compaction 완료 시 |
| `_stale_rs_version_map` | compaction 전 rowset — 기존 reader가 참조 가능 | compaction 완료 시 이동 |
| `_inc_rs_version_map` | 증분 clone용 | clone 시 참조 |

#### MVCC 스냅샷 캡처 과정

```
1. Reader가 version [0-100] 요청
       │
       ▼
2. capture_consistent_versions(version)
       │  VersionGraph에서 [0-100]을 커버하는 버전 경로 탐색
       │  예: [0-50](compacted) + [51-80] + [81-100]
       ▼
3. _capture_consistent_rowsets_unlocked()
       │  버전 경로의 각 버전에 대해:
       │    ① _rs_version_map에서 먼저 검색
       │    ② 없으면 _stale_rs_version_map에서 검색 (fallback)
       ▼
4. Rowset 집합 반환 → Reader가 이 집합으로 스캔
```

#### 왜 동작하는가 (핵심 보장)

```
시간 →  T1         T2              T3
        │          │               │
Reader A: capture [0-50][51-80]    │  ← stale map에서도 찾을 수 있음
        │          │               │
        │    Compaction 시작        │
        │    [0-50][51-80] → [0-80] │
        │    active: [0-80]         │
        │    stale:  [0-50][51-80]  │  ← Reader A가 아직 참조 중
        │          │               │
        │          │    Reader B: capture [0-80][81-100]
        │          │    ← active map에서 새 버전 사용
        │          │               │
Reader A 완료      │               │
        │          │               │
        │    GC: stale TTL 만료     │
        │    [0-50][51-80] 삭제     │  ← Reader A 완료 후 안전하게 삭제
```

- Compaction이 active map을 교체해도 stale map에 이전 rowset 보존
- 기존 reader는 stale map에서 자신의 스냅샷을 여전히 찾을 수 있음
- TTL 기반 GC로 장기 미사용 stale rowset만 정리 (`tablet_rowset_stale_sweep_time_sec`, 기본 1800초)

#### Primary Key 테이블의 MVCC

PK 테이블은 `TabletUpdates` 클래스가 별도 관리:

```cpp
if (_updates != nullptr) {
    return _updates->get_applied_rowsets(version, rowsets);
}
```

- EditVersion 기반 MVCC
- 같은 키의 최신 버전만 반환
- Delete Vector로 삭제된 행 필터링

---

### 5-2. Compaction 개요

누적된 작은 rowset들을 병합하여 읽기 성능을 유지하는 백그라운드 프로세스.

```
            Cumulative Point
                 │
    Base Layer   │   Cumulative Layer
    ─────────────┼───────────────────
    [0────50]    │  [51-55] [56-60] [61-65] [66-70]
    (compacted)  │  (개별 load마다 생성된 작은 rowset들)
                 │
                 │
    Base         │  Cumulative
    Compaction   │  Compaction
    (드물게,      │  (자주,
     전체 병합)   │   소규모 병합)
```

---

### 5-3. Cumulative Point (`_cumulative_point`)

Base 레이어와 Cumulative 레이어를 나누는 경계 버전 번호.

- `_cumulative_point` 미만: Base 레이어 (base compaction 대상)
- `_cumulative_point` 이상: Cumulative 레이어 (cumulative compaction 대상)

계산 로직 (`tablet.cpp:1109-1145`):

```
1. 모든 rowset을 start_version 오름차순 정렬
2. version -1부터 연속성 검사하며 순회
3. 멈추는 조건:
   - 버전 hole (빈틈) 발견
   - segment가 겹치는(overlapping) rowset
   - singleton이면서 delete version이 아닌 rowset
4. 멈춘 rowset의 start_version = cumulative_point
```

예시:

```
Rowsets: [0-5] [6-8] [9-10] [11-12] [13-15]
         ↑ compacted  ↑ compacted    ↑ singleton with overlap
                                     cumulative_point = 13

Base Layer:       [0-5] [6-8] [9-10] [11-12]
Cumulative Layer: [13-15]
```

---

### 5-4. Base Compaction (`storage/base_compaction.h`)

| 항목 | 내용 |
|------|------|
| 대상 | `start_version < _cumulative_point`인 rowset들 |
| 빈도 | 드물게 (큰 데이터 이동) |
| 출력 | 단일 compacted rowset |

선택 정책:

1. 최소 rowset 수 임계값 (`min_base_compaction_num_singleton_deltas`)
2. Cumulative 대비 Base 비율 초과 (`base_cumulative_delta_ratio`)
3. 마지막 base compaction 이후 시간 초과

제약:

- 입력 rowset들의 segment가 겹치지 않아야 함 (`_check_rowset_overlapping`)

---

### 5-5. Cumulative Compaction (`storage/cumulative_compaction.h`)

| 항목 | 내용 |
|------|------|
| 대상 | `start_version >= _cumulative_point`인 rowset들 |
| 빈도 | 자주 (작은 rowset 빠르게 병합) |
| 출력 | 병합된 rowset |

선택 정책:

- cumulative_point 이상의 연속 버전 rowset 선택
- singleton/overlapping rowset에서 중단
- delete 버전에서 중단
- score 임계값 기반

compaction 후:

```
_cumulative_point = input_rowsets.back().end_version() + 1
```

---

### 5-6. Compaction 실행 흐름

```
Phase 1: Pick Rowsets
    │  tablet->pick_candidate_rowsets_to_[base|cumulative]_compaction()
    │  _rs_version_map에서 _cumulative_point 기준으로 필터
    │  버전 연속성 검증, 정책 기반 선택
    ▼
Phase 2: Merge
    │  _output_version = [input_first.start, input_last.end]
    │
    │  TabletReader 생성 (READER_BASE_COMPACTION / READER_CUMULATIVE_COMPACTION)
    │    → 입력 rowset들에서 데이터 읽기
    │    → 필터, 삭제, 갱신 적용
    │
    │  RowsetWriter로 출력 rowset 생성
    │    → 병합된 데이터를 새 세그먼트에 기록
    ▼
Phase 3: Modify Rowsets (Atomic Swap)
    │  wrlock(_meta_lock)
    │  modify_rowsets_without_lock({output_rowset}, input_rowsets)
    │    → _rs_version_map: 입력 제거, 출력 추가
    │    → _stale_rs_version_map: 입력 rowset 이동 (MVCC 보존)
    │  save_meta()
    │  Rowset::close_rowsets(input_rowsets)
    ▼
Phase 4: Cleanup (GC)
    │  stale rowset은 TTL 만료 후 자동 삭제
    │  delete_expired_stale_rowset()
    │    → TimestampedVersionTracker에서 만료 경로 조회
    │    → _stale_rs_version_map에서 제거
    │    → StorageEngine::add_unused_rowset()로 물리 삭제
    ▼
완료
```

---

### 5-7. Compaction 정책

#### Default Policy (`default_compaction_policy.h`)

- 2계층: Base + Cumulative
- 고정 `_cumulative_point`
- 크기/시간 기반 트리거

#### Size-Tiered Policy (`size_tiered_compaction_policy.h`)

- 크기 기반 다중 레벨
- Level 0 → Level 1 → ... → Level N
- 레벨 크기 임계값 초과 시 compaction 트리거
- 더 공격적인 병합 전략

Score 계산:

```
score = segment_num + data_bonus
data_bonus: DUP_KEYS일 때 더 공격적
```

---

### 5-8. CompactionManager (`storage/compaction_manager.h`)

모든 compaction 작업을 스케줄링하는 중앙 관리자.

| 멤버 | 설명 |
|------|------|
| `_compaction_candidates` | compaction이 필요한 tablet 우선순위 큐 |
| `_running_tasks` | 현재 실행 중 작업 (tablet_id 기반) |
| Thread Pool | 실행용 스레드 풀 |

스케줄링 알고리즘:

1. Compaction 필요도에 따라 tablet 점수 산정
2. 최고 점수 후보 선택
3. 전제 조건 확인 (마이그레이션 중 아님, 비활성화 아님)
4. Base 또는 Cumulative compaction 작업 생성
5. `max_compaction_concurrency` config로 동시성 제한

---

### 5-9. Primary Key 테이블 Compaction 특이사항

PK 테이블은 같은 키가 여러 rowset에 존재할 수 있어 추가 처리가 필요하다.

| 항목 | 내용 |
|------|------|
| Delete Vector | 각 rowset의 삭제된 행을 추적. compaction 시 적용 |
| Conflict Resolver | `primary_key_compaction_conflict_resolver.h` |
| 키 충돌 해결 | 나중 버전이 우선 (latest wins) |
| PK 인덱스 재구축 | compaction 출력 rowset에 대해 키 인덱스 재생성 |

---

### 5-10. Compaction + MVCC 통합 타임라인

```
시간 →
────────────────────────────────────────────────────────────────

[Active Map]
  Version: [0-50] [51-60] [61-70] [71-80]

                    │
                    ▼  Cumulative Compaction 시작
                    │  입력: [51-60] [61-70]
                    │
[Reader A 시작]     │
  snapshot: [0-50] [51-60] [61-70] [71-80]    ← active에서 캡처
                    │
                    ▼  Compaction 완료
                    │
[Active Map 갱신]   │
  Version: [0-50] [51-70] [71-80]             ← [51-60][61-70] → [51-70]
                    │
[Stale Map]        │
  [51-60] [61-70]                              ← 이전 rowset 보존
                    │
[Reader A 계속]    │
  여전히 [51-60] [61-70] 접근 가능              ← stale map에서 찾음
                    │
[Reader B 시작]    │
  snapshot: [0-50] [51-70] [71-80]            ← active에서 새 버전 캡처
                    │
[Reader A 완료]    │
                    │
                    ▼  TTL 만료 (1800초 후)
                    │
[Stale Map GC]     │
  [51-60] [61-70] 삭제                        ← 더 이상 참조 없음, 안전 삭제
```

---

## 설계 원칙 요약

| 원칙 | 설명 |
|------|------|
| 불변성 (Immutability) | Segment, Rowset은 생성 후 수정 불가. Compaction으로 새로 생성 |
| Lazy Loading | 인덱스는 필요할 때 한 번만 로드 (thread-safe once) |
| 참조 카운팅 | Rowset의 `_refs_by_reader`로 안전한 삭제 보장 |
| 컬럼 스토리지 | 각 컬럼이 독립적 ColumnReader → 필요한 컬럼만 읽기 |
| 스키마 버전닝 | 각 Rowset이 자체 스키마 스냅샷 보유 → 빠른 스키마 변경 |
| 다중 인덱스 pruning | Short Key → Bitmap → ZoneMap → BloomFilter 순 점진적 제거 |
| 체크섬 무결성 | 모든 페이지 + Footer에 CRC32C → 데이터 손상 탐지 |
| MVCC (Rowset 버전 관리) | active/stale 이중 맵으로 compaction 중 읽기 일관성 보장 |
| TTL 기반 GC | stale rowset은 TTL 만료 후 자동 삭제 (기본 1800초) |

---

## 주요 소스 파일 경로

| 구성 요소 | 파일 |
|-----------|------|
| Tablet | `be/src/storage/tablet.h` |
| TabletMeta | `be/src/storage/tablet_meta.h` |
| Rowset | `be/src/storage/rowset/rowset.h` |
| RowsetMeta | `be/src/storage/rowset/rowset_meta.h` |
| Segment | `be/src/storage/rowset/segment.h` |
| SegmentWriter | `be/src/storage/rowset/segment_writer.cpp` |
| ColumnReader | `be/src/storage/rowset/column_reader.h` |
| ColumnWriter | `be/src/storage/rowset/column_writer.cpp` |
| PageIO | `be/src/storage/rowset/page_io.cpp` |
| Short Key Index | `be/src/storage/short_key_index.h` |
| Ordinal Index | `be/src/storage/rowset/ordinal_page_index.h` |
| Zone Map Index | `be/src/storage/rowset/zone_map_index.h` |
| Bitmap Index | `be/src/storage/rowset/bitmap_index_reader.h` |
| Bloom Filter Index | `be/src/storage/rowset/bloom_filter_index_reader.h` |
| Page Pointer | `be/src/storage/rowset/page_pointer.h` |
| Encoding Info | `be/src/storage/rowset/encoding_info.h` |
| Binary Plain Page | `be/src/storage/rowset/binary_plain_page.h` |
| Segment Proto | `gensrc/proto/segment.proto` |
| OlapFile Proto | `gensrc/proto/olap_file.proto` |
| LoadChannel | `be/src/runtime/load_channel.h` |
| TabletsChannel | `be/src/runtime/tablets_channel.h` |
| DeltaWriter | `be/src/storage/delta_writer.h` |
| DeltaWriter (Lake) | `be/src/storage/lake/delta_writer.h` |
| MemTable | `be/src/storage/memtable.h` |
| TabletWriter | `be/src/storage/lake/tablet_writer.h` |
| RowsetWriter | `be/src/storage/rowset/rowset_writer.h` |
| TabletReader | `be/src/storage/tablet_reader.h` |
| SegmentIterator | `be/src/storage/rowset/segment_iterator.cpp` |
| ChunkIterator | `be/src/storage/chunk_iterator.h` |
| MergeIterator | `be/src/storage/merge_iterator.h` |
| ColumnPredicate | `be/src/storage/column_predicate.h` |
| ConjunctivePredicates | `be/src/storage/conjunctive_predicates.h` |
| DelVector | `be/src/storage/del_vector.h` |
| Compaction | `be/src/storage/compaction.h` |
| Base Compaction | `be/src/storage/base_compaction.h` |
| Cumulative Compaction | `be/src/storage/cumulative_compaction.h` |
| CompactionManager | `be/src/storage/compaction_manager.h` |
| Compaction Policy | `be/src/storage/size_tiered_compaction_policy.h` |
| VersionGraph | `be/src/storage/version_graph.h` |
| TxnManager | `be/src/storage/txn_manager.h` |
| Lake Transactions | `be/src/storage/lake/transactions.h` |
| PK Conflict Resolver | `be/src/storage/primary_key_compaction_conflict_resolver.h` |
