# oGRAC Distributed Property Graph Storage Design

## 1. Scope

This design adds the first kernel-facing slice of a distributed property graph
engine to oGRAC. The target workload is:

- one billion vertices and ten billion edges in the first production stage;
- growth to tens of billions of vertices;
- 60–70% one- or two-hop queries and 30–40% three- to six-hop queries;
- query P99 below 30 ms;
- configurable statement-consistent, transaction-snapshot, and eventually
  consistent reads, with statement consistency as the default;
- single-writer/multi-reader operation in the first stage, while preserving a
  path to partition-affine multi-writer/multi-reader operation;
- future vector-to-graph seed lookup and vector-filtered edges.

The first implementation slice covered by this design consists of persistent
data structures, ID codecs, page-layout helpers, validation, and public kernel
interfaces. It does not yet implement the OpenCypher parser, a complete graph
optimizer, distributed traversal execution, or production DDL.

## 2. Architectural Decision

Use a native graph access method over existing oGRAC infrastructure:

- retain `page_head_t`, `page_tail_t`, Buffer Pool, PCR MVCC, undo/redo,
  tablespace extents, DTC/DRC/DCS, recovery, and shared storage;
- add fixed-slot vertex and edge pages;
- add typed adjacency directory and adjacency data pages;
- reuse PCR Heap and LOB storage for variable-length properties;
- add graph-specific traversal and language layers incrementally.

Pure relational mapping was rejected because repeated B-tree probes and joins
make multi-hop latency unpredictable. A fully independent graph transaction
engine was rejected because it would duplicate oGRAC consistency, logging, and
recovery machinery.

## 3. Partition and DRC Model

Graph partitioning and DRC partitioning are orthogonal.

- A graph virtual partition is a stable routing and allocation bucket encoded
  in a VID or EID.
- A placement partition groups virtual partitions for scheduling and movement.
- A DRC partition shards page-resource metadata derived from `page_id`.
- A preferred instance is a soft execution and cache-affinity hint.
- A page owner is the node currently holding a valid S or X page state.

Pages belonging to one graph partition may map to many DRC partitions. This is
desirable because it spreads DRC master load. Correctness always remains with
DRC. Graph routing attempts to keep page owners and readers local without
changing DRC hashing.

Moving a virtual partition between placement partitions or preferred instances
does not alter IDs. Changing the partition key of existing vertices changes
their virtual partition and therefore requires a graph rebuild or an explicit
forwarding layer; the default operation is rebuild.

## 4. Stable IDs

VID and EID are unsigned 64-bit encoded values:

```text
63                    48 47                    16 15                 0
+-----------------------+------------------------+--------------------+
| virtual_partition_id  | local_sequence         | generation         |
| 16 bits               | 32 bits                | 16 bits            |
+-----------------------+------------------------+--------------------+
```

- `virtual_partition_id` identifies a stable graph bucket, not a node or DRC
  partition.
- `local_sequence` identifies a fixed slot inside the partition.
- `generation` detects stale references after a deleted slot is reused.

Implement IDs with masks and shifts, not C bitfields, to avoid ABI and
endianness dependencies.

Logical location is:

```text
logical_page = local_sequence / slots_per_page
slot_id      = local_sequence % slots_per_page
page_id      = extent_map_lookup(segment, logical_page)
```

Generation is identity protection, not MVCC. Transaction visibility continues
to use PCR ITLs, undo, and query SCNs.

## 5. Multi-Label Vertices

A vertex supports multiple OpenCypher labels/tags. The fixed slot stores an
interned `label_set_id`, not a single `label_id`.

Catalog entries map:

```text
label_set_id -> label bitmap/list + schema_layout_id
```

Label scans use a per-partition inverted label index. Traversal from an existing
VID checks the cached label-set bitmap.

The external-ID map is keyed by an immutable namespace rather than a mutable
label:

```text
(graph_id, id_namespace_id, external_id) -> VID
```

Adding or removing labels updates the vertex `label_set_id` and label indexes
transactionally without changing VID or moving the vertex.

## 6. Segment and Page Types

Add these graph page types:

```text
PAGE_TYPE_GRAPH_SEG_HEAD
PAGE_TYPE_GRAPH_VERTEX
PAGE_TYPE_GRAPH_EDGE
PAGE_TYPE_GRAPH_ADJ_DIR
PAGE_TYPE_GRAPH_ADJ_DATA
```

Properties initially reuse `PAGE_TYPE_PCRH_DATA` and LOB pages.

Each graph virtual partition has logical segment entries for:

- vertices;
- canonical edges;
- outbound adjacency;
- inbound adjacency;
- optional adjacency directories;
- properties.

A graph segment header records graph ID, partition ID, segment type, layout
version, slot size, slots per page, high-water sequence, reusable-slot list,
extent map, and statistics. Sequence numbers are reserved in batches to avoid a
segment-head hotspot.

## 7. Vertex Slot

The base vertex slot is 64 bytes:

```c
typedef struct st_graph_vertex_meta {
    uint16 flags;
    uint16 generation;
    uint32 label_set_id;
    uint16 schema_layout_id;
    uint16 hot_property_bitmap;
    uint8 itl_id;
    uint8 inline_out_count;
    uint8 inline_in_count;
    uint8 adjacency_mode;
} graph_vertex_meta_t; /* 16 bytes */

typedef struct st_graph_adj_desc {
    rowid_t root_locator;
    uint32 total_degree;
} graph_adj_desc_t; /* 12 bytes */

typedef struct st_graph_vertex_base_slot {
    graph_vertex_meta_t meta;       /* 16 */
    uint8 inline_properties[16];    /* 16 */
    rowid_t property_rowid;         /*  8 */
    graph_adj_desc_t out_adj;       /* 12 */
    graph_adj_desc_t in_adj;        /* 12 */
} graph_vertex_base_slot_t; /* 64 bytes */
```

`flags` includes allocated, deleted, has-property-row, outbound overflow,
inbound overflow, supernode, moving, and frozen states.

`schema_layout_id` identifies the inline property codec and physical schema.
`hot_property_bitmap` identifies present inline values. `itl_id` follows PCR
row semantics.

`property_rowid` points to a graph-owned PCR Heap property row. A base business
record is referenced by stable business key, not by permanent physical RowID.

### 7.1 Optional Vertex Inline Edges

Vertex inline edges are optional and disabled by default. They are suitable
only for sparse, low-update, topology-heavy edge types.

Outbound and inbound arrays are physically separate because each direction may
carry different inline properties:

```c
typedef struct st_graph_inline_edge_ref {
    graph_vid_t neighbor_vid;
    graph_edge_token_t edge_token;
} graph_inline_edge_ref_t; /* 16 bytes before directional properties */
```

Supported profiles are configured per stable vertex storage type. The preferred
profiles are zero, two, or four inline edges. Eight- and twelve-edge profiles
are not defaults because they reduce vertex-page density and increase page
ownership churn.

An edge token is allowed only in mixed-type vertex-inline arrays. It contains:

```text
edge_local_sequence:32 | generation:16 | compact_edge_type_code:16
```

Normal adjacency pages store a full EID.

## 8. Canonical Edge Slot

Every logical edge has one authoritative 64-byte edge slot:

```c
typedef struct st_graph_edge_meta {
    uint16 flags;
    uint16 generation;
    uint32 edge_type_id;
    uint16 schema_layout_id;
    uint16 hot_property_bitmap;
    uint8 itl_id;
    uint8 property_mode;
    uint8 validity_flags;
    uint8 reserved_zero;
} graph_edge_meta_t; /* 16 bytes */

typedef struct st_graph_edge_slot {
    graph_edge_meta_t meta;       /* 16 */
    graph_vid_t src_vid;          /*  8 */
    graph_vid_t dst_vid;          /*  8 */
    uint8 inline_properties[24];  /* 24 */
    rowid_t property_rowid;       /*  8 */
} graph_edge_slot_t; /* 64 bytes */
```

The canonical slot is the source of truth for endpoints and edge properties.
The property row contains cold or variable-length properties.

## 9. Adjacency Directory

The adjacency directory is a logical per-vertex, per-direction edge-type
router:

```text
(vertex, direction, edge_type)
    -> format, count, schema, sort order, root pages, shard policy
```

It is not a DRC directory and need not occupy a separate page.

Physical modes are:

- `NO_ADJACENCY`;
- `DIRECT`, where one edge type points directly to an adjacency page;
- `PACKED_DIRECTORY`, where a mini-directory and several small adjacency lists
  share a page;
- `EXTERNAL_DIRECTORY`, where many edge types use a separate directory;
- `SUPERNODE_SHARDED`, where the directory points to a shard directory.

An external entry is:

```c
typedef struct st_graph_adj_dir_entry {
    uint32 edge_type_id;
    uint8 direction;
    uint8 entry_format;
    uint16 shard_count;
    uint64 edge_count;
    rowid_t root_locator;
    uint32 flags;
    uint16 property_schema_version;
    uint8 shard_policy;
    uint8 sort_key_id;
} graph_adj_dir_entry_t; /* 32 bytes */
```

The directory enables type-specific formats, counts, sorting, zone maps,
compression, supernode sharding, and atomic root switching during online
rebuilds.

## 10. Adjacency Entries and Directional Inline Properties

Normal adjacency entries use a full EID:

```c
typedef struct st_graph_adj_entry_base {
    graph_vid_t neighbor_vid;
    graph_eid_t eid;
} graph_adj_entry_base_t; /* 16 bytes */
```

The entry can append zero, eight, or sixteen bytes of direction-specific inline
properties. User-facing names are:

- `EDGE INLINE PROPERTIES`: authoritative values in the canonical Edge Slot;
- `OUT INLINE PROPERTIES`: copies in outbound inline/adjacency records;
- `IN INLINE PROPERTIES`: copies in inbound inline/adjacency records.

The term “covering property” remains internal and is not exposed in DDL.

An adjacency slot directory carries entry offset, ITL ID, and flags:

```c
typedef struct st_graph_adj_slot {
    uint16 entry_offset;
    uint8 itl_id;
    uint8 flags;
} graph_adj_slot_t; /* 4 bytes */
```

Adjacency pages contain one owner VID, direction, and edge type. Their headers
record entry format, shard ID, counts, free-space boundaries, next page,
optional min/max values, Bloom offset, layout version, and PCR ITLs.

Copied directional properties are updated in the same transaction as the
canonical edge. Frequently updated, variable-length, or large properties are
not copied.

## 11. Supernodes

Supernodes are promoted by a configurable degree threshold. Supported sharding
policies are:

- hash by neighbor VID;
- range by neighbor VID;
- range by time;
- edge subtype;
- hybrid edge type and neighbor hash.

The default is automatic hash sharding by neighbor VID. Promotion creates a
versioned shard directory, backfills existing entries, routes new writes to the
target shards, atomically switches the directory root, waits for old snapshots,
and reclaims old pages.

## 12. User-Facing Interface

OpenCypher remains the query language. oGRAC-specific DDL extensions define
physical layout.

Example:

```sql
CREATE EDGE TYPE CLICKED IN GRAPH ad_graph
FROM User TO Advertisement (
    timestamp TIMESTAMP NOT NULL,
    weight FLOAT,
    status SMALLINT,
    device_type SMALLINT,
    detail JSON
)
STORAGE (
    EDGE INLINE PROPERTIES (
        timestamp, weight, status, device_type
    ),
    OUT INLINE PROPERTIES (
        timestamp ENCODE UINT32_DELTA,
        weight ENCODE FP16,
        status ENCODE UINT8,
        device_type ENCODE UINT8
    ),
    IN INLINE PROPERTIES (
        timestamp ENCODE UINT32_DELTA,
        status ENCODE UINT8
    ),
    OUT ADJACENCY (
        SORT BY timestamp DESC,
        ZONEMAP (timestamp, weight)
    ),
    IN ADJACENCY (
        SORT BY timestamp DESC,
        ZONEMAP (timestamp)
    )
);
```

The DDL compiler validates byte budgets, fixed-width codecs, direction-specific
formats, and write-amplification warnings.

The first code slice exposes definition structures and validators for this
interface. Parser grammar and DDL execution are subsequent slices.

## 13. Kernel Interfaces

The first public kernel interface is organized around opaque graph handles and
typed definitions:

```c
status_t graph_validate_storage_def(const graph_storage_def_t *def);
status_t graph_validate_vertex_def(const graph_vertex_def_t *def);
status_t graph_validate_edge_def(const graph_edge_def_t *def);

graph_vid_t graph_make_vid(uint16 vpart, uint32 sequence, uint16 generation);
uint16 graph_vid_vpart(graph_vid_t vid);
uint32 graph_vid_sequence(graph_vid_t vid);
uint16 graph_vid_generation(graph_vid_t vid);

status_t graph_vertex_locate(const graph_segment_layout_t *layout,
    graph_vid_t vid, graph_slot_location_t *location);
status_t graph_edge_locate(const graph_segment_layout_t *layout,
    graph_eid_t eid, graph_slot_location_t *location);

status_t graph_open(knl_session_t *session, uint32 graph_id,
    graph_handle_t **handle);
void graph_close(graph_handle_t *handle);

status_t graph_get_vertex(graph_handle_t *handle, graph_vid_t vid,
    knl_scn_t query_scn, graph_vertex_view_t *vertex);
status_t graph_get_edge(graph_handle_t *handle, graph_eid_t eid,
    knl_scn_t query_scn, graph_edge_view_t *edge);
status_t graph_expand(graph_handle_t *handle, const graph_expand_req_t *request,
    graph_expand_result_t *result);
```

The initial implementation supplies codecs, validation, layout calculation,
and API declarations. Storage mutation and traversal bodies can return a
specific not-implemented status until their TDD slices are implemented; they
must not silently succeed.

## 14. Write and Read Flows

Creating an edge atomically:

1. allocate EID from the source virtual partition;
2. insert canonical Edge Slot;
3. update source outbound inline/adjacency entry;
4. update destination inbound inline/adjacency entry;
5. update directories or promote layout when required;
6. write undo and redo;
7. commit all changes together.

Traversing:

1. resolve external ID to VID once;
2. locate and read Vertex Slot;
3. choose direct, packed, external, or sharded adjacency mode;
4. prune pages by type, min/max, and Bloom metadata;
5. filter directional inline properties;
6. group neighbor VIDs by virtual partition;
7. prefetch a small number of remote pages or ship a large frontier to the
   preferred instance;
8. batch-read surviving vertex and property rows.

## 15. Schema Evolution

Metadata-only nullable cold properties require only a catalog version.

Changing inline layout, codecs, or slot size creates a new layout and follows:

1. prepare and validate;
2. publish schema epoch and invalidate plans;
3. dual-read old and new layouts;
4. backfill under PCR MVCC with redo/undo;
5. switch default layout;
6. wait for old snapshots and reclaim old storage.

Changing slot size requires a new segment. Adding or removing a label changes
only `label_set_id`, label indexes, and cold role-specific properties unless a
separate inline-layout change is explicitly requested.

## 16. Error Handling

Validation returns explicit errors for:

- invalid ID fields or reserved values;
- layout sizes that exceed page capacity;
- unsupported inline-property codecs;
- OUT or IN byte budgets above the selected entry format;
- slot count overflow;
- unsupported adjacency mode or shard policy;
- generation mismatch;
- page type or layout-version mismatch;
- dangling RowID, VID, or EID;
- attempts to change a partition key without rebuild/forwarding mode.

Persistent readers reject unknown required layout versions. Reserved bytes are
written as zero and checked during page verification.

## 17. Testing

The initial code slice uses test-driven development.

Automated tests cover:

- VID/EID boundary values and round trips;
- generation mismatch detection;
- vertex and edge persistent structure sizes and offsets;
- page capacity for 8 KB and supported page sizes;
- sequence-to-page-and-slot location;
- zero/two/four vertex-inline profiles;
- zero/eight/sixteen directional property budgets;
- adjacency-directory mode validation;
- supernode policy validation;
- invalid and overflow definitions;
- serialization independent of host bitfield layout.

Compile-time assertions protect persistent sizes. Runtime tests use the
repository’s existing C unit-test conventions. Broader recovery, DRC, MVCC, and
OpenCypher tests are added with their implementation slices.

## 18. Code Placement

Planned files for the first slice:

```text
pkg/src/upgrade_check/knl_graph_persistent.h
pkg/src/kernel/graph/knl_graph_defs.h
pkg/src/kernel/graph/knl_graph_id.h
pkg/src/kernel/graph/knl_graph_layout.h
pkg/src/kernel/graph/knl_graph_layout.c
pkg/src/kernel/graph/knl_graph_interface.h
pkg/test/unit_test/ut/kernel/kernel_graph_test.cpp
```

Integration changes are expected in:

```text
pkg/src/kernel/buffer/knl_page.h
pkg/src/kernel/CMakeLists.txt
pkg/src/upgrade_check/upgrade_struct_check.h
pkg/test/unit_test/ut/kernel/CMakeLists.txt
```

Redo/undo registration, recovery, dump tools, catalog persistence, parser
grammar, optimizer operators, and DTC graph CR paths are later implementation
slices and must be designed and tested separately.
