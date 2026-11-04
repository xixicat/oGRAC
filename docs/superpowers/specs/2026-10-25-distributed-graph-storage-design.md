# oGRAC 分布式属性图存储设计

状态：待总体确认
日期：2026-10-25
范围：广告与搜索场景的分布式属性图，第一期架构与持久化设计

本文汇总已讨论并确认的设计思路、数据结构、用户接口、适用场景和选择原因。确认本文后，再进入实施计划和编码。

## 1. 阅读方式

本文按以下顺序组织：

1. 业务场景和设计目标。
2. 总体架构及选择原因。
3. 集群一致性、分区和远程访问。
4. 编号化的数据结构和字段用途。
5. 复杂查询如何使用这些结构完成遍历。
6. 用户接口、Schema 变更和分区变更。
7. 模块落点、分期范围和待确认清单。

文中的编号 `01`–`28` 与数据结构关系图一致，后续评审可直接引用编号。

## 2. 业务场景

### 2.1 规模与延迟

| 项目 | 目标 |
|---|---|
| 第一期规模 | 十亿级顶点、百亿级边 |
| 后续规模 | 百亿级顶点 |
| 查询构成 | 1–2 跳占 60%–70%，3–6 跳占 30%–40% |
| 延迟 | P99 小于 30 ms |
| 超级节点 | 单点邻居可能超过 100 万 |
| 读一致性 | 事务快照、语句级强一致、最终一致均可配置，默认语句级强一致 |
| 写入阶段 | 第一期一写多读，架构预留分区亲和的多写多读 |
| 后续能力 | 向量召回图起点，以及按向量条件过滤边 |

### 2.2 典型业务图

广告搜索场景同时包含用户行为、广告关系和员工权限：

```text
顶点：
User:Person:Employee:Manager
Campaign
Advertisement
Query
Interest
Department

边：
(User)-[:SEARCHED]->(Query)
(User)-[:CLICKED]->(Advertisement)
(User)-[:INTERESTED_IN]->(Interest)
(Employee)-[:BELONGS_TO]->(Department)
(Manager)-[:MANAGES]->(Campaign)
(Advertisement)-[:BELONGS_TO]->(Campaign)
(Advertisement)-[:TARGETS]->(Interest)
(Advertisement)-[:SIMILAR]->(Advertisement)
```

一个用户可以同时拥有 `User`、`Person`、`Employee`、`Manager` 等多个 Label。

边属性过滤是核心路径，例如点击时间、权重、状态、相似分和有效期。完整 JSON、审计信息和向量本体属于冷属性。

### 2.3 三条代表性查询

#### 低度关系与时间有效性

```cypher
MATCH (u:User:Employee:Manager {external_id: $user_id})
      -[m:MANAGES]->(c:Campaign)
WHERE m.status = 1
  AND m.valid_from <= $now
  AND (m.valid_to IS NULL OR m.valid_to > $now)
RETURN id(c)
```

这类查询度数低、过滤字段固定，适合从顶点快速进入指定 Edge Type。

#### 向量召回起点

```cypher
CALL graph.vector.query(
    'ad_graph', 'Advertisement', 'embedding', $query_vector, 500
)
YIELD vid, distance
WHERE distance < 0.25
RETURN vid, distance
```

向量索引返回 VID，再按虚拟分区分桶并批量读取顶点页。

#### 短历史过滤加有限深度扩展

```cypher
MATCH (u:User {external_id: $user_id})
      -[click:CLICKED]->(seed:Advertisement)
WHERE click.timestamp >= $seven_days_ago
  AND click.status = 1
  AND click.weight >= 0.6

MATCH path =
      (seed)-[similar:SIMILAR*1..3]->(candidate:Advertisement)
WHERE ALL(e IN relationships(path) WHERE e.score >= 0.75)
  AND id(candidate) IN $vector_candidate_vids
RETURN candidate,
       MAX(click.weight) AS click_score,
       MIN(length(path)) AS graph_distance
ORDER BY click_score DESC, graph_distance
LIMIT 100
```

这条查询同时覆盖属性过滤、1–3 跳扩展、跨分区 Frontier 和最终少量属性读取，是存储与执行设计的主要验证场景。

## 3. 总体架构

### 3.1 选择的路线

采用“原生图访问方法 + oGRAC 一致性底座”：

```text
OpenCypher 前端
    → 图优化器与执行器
    → 图访问方法
    → oGRAC Buffer、PCR MVCC、Redo/Undo、DTC/DRC/DCS、Extent
    → 共享存储
```

复用：

- 页面头尾、校验和、Buffer Pool；
- PCR ITL、Undo、Redo、恢复；
- Extent 和表空间分配；
- DTC 页面传输、DRC 锁目录、DCS 本地可用性判断；
- 会话、事务、SCN 和网络协议。

新增：

- 固定槽顶点页和规范边页；
- 按 Edge Type 组织的邻接目录与邻接页；
- Label Set、ID Map、图分区 Catalog；
- `GraphScan`、`GraphExpand`、`PathExpand` 和图优化器；
- 后续 `VectorSeed`、`EdgeVectorFilter`。

### 3.2 选择原因

关系表映射可以最快接入 SQL，但每一跳都退化成索引探测和 Join。百亿边和 30 ms P99 下，重复 B-tree 探测与远程页访问难以稳定控制。

独立图事务引擎可以定制页面格式，但需要重新实现 MVCC、日志、恢复、集群锁和故障处理，第一期风险过高。

混合方案把性能关键的邻接布局做成原生页面，把正确性继续交给 oGRAC。图页通过现有 `buf_enter_page()` 和 `dtc_read_page()` 访问，DRC 自动管理页面 Owner。

### 3.3 分层边界

| 层 | 职责 |
|---|---|
| OpenCypher | 查询语言、模式匹配、路径和返回结果 |
| 图 DDL | 图、Label、Edge Type、内嵌属性、分区和超级节点策略 |
| 优化器 | 起点选择、Expand 顺序、本地执行或移动计算、批量大小 |
| 执行器 | Frontier 去重、分层扩展、过滤、聚合和限流 |
| 图访问方法 | VID/EID 定位、邻接读取、属性解码、布局迁移 |
| oGRAC Kernel | 页面锁、事务、日志、恢复、集群一致性 |
| 共享存储 | 所有计算节点读写同一份页面和日志 |

## 4. 一致性与多写

### 4.1 三种读模式

| 模式 | 语义 | 实现方向 |
|---|---|---|
| `STATEMENT` | 查询开始前已提交的数据对该语句可见，默认 | 使用查询 SCN，必要时构造 CR |
| `TRANSACTION` | 整个事务使用同一个快照 SCN | 所有跳使用事务 SCN 和 CR |
| `EVENTUAL` | 允许读取本地已缓存的旧版本 | 优先本地副本，后台刷新 |

一次图遍历中的所有页面使用同一个查询 SCN。最终一致模式显式放宽该要求。

### 4.2 第一期一写多读

所有图分区的 `preferred_instance` 指向同一个写节点：

- 写请求路由到该节点；
- 其他节点通过 DRC S 副本或 CR 页面读取；
- 写节点持有相关页面的 X Owner 时，普通读写不再访问远程 DRC Master。

### 4.3 多写预留

多写按图分区并行，页面仍保持单一 X Owner：

```text
节点 0：Placement Partition A
节点 1：Placement Partition B
节点 2：Placement Partition C
```

不同节点写不同页面集合。DRC 仍拒绝两个节点同时持有同一页的 X 权限。跨分区边在同一事务中按分区编号顺序更新，降低死锁概率。

跨分区原子提交继续使用 oGRAC 事务、Undo 和 Redo。第一期不引入新的分布式提交协议。

## 5. 分区、DRC 与远程访问

### 5.1 五个概念

| 概念 | 作用 | 是否编码进 VID |
|---|---|---|
| Virtual Partition | 稳定的数据桶，用于定位 Segment | 是，16 bit |
| Placement Partition | 一组 Virtual Partition 的调度和迁移单位 | 否 |
| Preferred Instance | 当前建议执行和缓存的计算节点 | 否 |
| DRC Partition | 按 `page_id` 哈希管理锁目录 | 否 |
| Page Owner | 当前持有页面 S 或 X 状态的节点 | 否 |

定位链路：

```text
VID
 → Virtual Partition
 → Segment Extent Map
 → page_id
 → DRC Partition
 → Page Owner
```

DRC Master 管理资源目录，不表示页面数据位于该节点。当前节点已有满足锁模式的本地页时，读写直接访问本地 Buffer：

- 有效 S 或 X：本地读；
- 有效 X：本地写；
- 仅有 S 且需要写：向 Master 申请升级并失效其他副本；
- 本地无有效页或 SCN 不满足：通过 DRC 定位 Owner 或构造 CR。

### 5.2 图分区的价值

图分区解决数据共置、查询路由、写入亲和和在线迁移。它让 VID 自带稳定数据桶，遍历时无需查询全局目录即可知道 Segment 和首选节点。

同一图分区的页面可以分散到多个 DRC Partition。这样可以分散 DRC Master 压力。查询本地性依赖 Page Owner 和 Preferred Instance，而不是把所有页面强行映射到同一个 DRC Partition。

### 5.3 远程访问控制

远程访问存在，但按页和按层批量处理：

```text
Frontier
 → 按 Virtual Partition 分桶
 → VID 与 page_id 去重
 → 少量远程页：批量 DCS Prefetch
 → 大量跨分区 Frontier：把计算发送到目标 Preferred Instance
 → 每层每个分区一次批量交换
```

目标是把网络复杂度从“每条边一次请求”降为“每一跳每个相关分区一次批量交换”。

### 5.4 分区键

分区键在创建顶点时计算：

```text
virtual_partition_id =
    hash(graph_id, partition_key) mapped into virtual partition space
```

分区键要求稳定、分散，并尽量让高频关联对象共置。动态 Label 和会修改的属性不作为分区键。

广告场景可以按租户、广告主、地域或稳定业务主键组合分区。具体键和分区数量通过边切割率测试确定，编码容量预留 65536 个 Virtual Partition。

## 6. 标识体系

### 6.1 VID 与 EID

VID 使用以下 64-bit 布局：

```text
+-----------------------+------------------------+--------------------+
| virtual_partition_id  | local_sequence         | generation         |
| 16 bit                | 32 bit                 | 16 bit             |
+-----------------------+------------------------+--------------------+
```

选择 `u64` 是因为 `u32` 约 42.9 亿的上限不能覆盖百亿级顶点。

| 字段 | 用途 | 选择原因 |
|---|---|---|
| `virtual_partition_id` | 稳定数据桶 | 节点和 DRC Master 会变化，不能编码进永久 ID |
| `local_sequence` | 分区内固定槽号 | 通过除法和取模 O(1) 得到逻辑页和槽 |
| `generation` | 槽重用代次 | 防止删除后的旧边指向新对象 |

16 bit generation 提供 65536 次代次，适合广告数据的增删。代次回绕后该槽退休。generation 不替代 SCN 或 MVCC。

单分区 32 bit sequence 约支持 42.9 亿槽。百亿总体规模由多个 Virtual Partition 共同承载。

边默认属于源顶点的 Virtual Partition，使出边记录与源顶点共置。

EID 同样使用 64 bit，但中间 32 bit 是物理无关的 Edge Location：

```text
+-----------------------+------------------------+----------+--------------------+
| virtual_partition_id  | logical_page_no        | slot_id  | generation         |
| 16 bit                | 24 bit                 | 8 bit    | 16 bit             |
+-----------------------+------------------------+----------+--------------------+
```

`logical_page_no` 定位 Virtual Partition 的 Edge Segment 中的逻辑页，`slot_id` 定位该页中的固定槽。8KB 页面中最小 Canonical Edge Slot 是 40B，单页不会超过 256 个槽，因此 8-bit `slot_id` 足够。

24-bit `logical_page_no` 可表示 16,777,216 个 Edge Page，即每个 Virtual Partition 约 128GiB 的原始页面地址空间。扣除页头、PCR ITL 和分配 Bitmap 后，按 40B–64B Slot 估算，单个 Virtual Partition 可保存约 17 亿–31 亿条 Canonical Edge。百亿条边即使只使用 1,024 个 Virtual Partition，平均每分区也只有约 977 万条边，第一期余量充足。

不从 generation 借位扩大页号。generation 用于阻止 Slot 复用后的陈旧 EID 命中新边，缩短为 8 bit 或 12 bit 会显著增加高频增删场景的回绕风险。Catalog 预留 `eid_layout_version`；若未来明确要求单个 Virtual Partition 超过上述容量，可以增加 `VP12_PAGE28_GEN16` 或更宽 EID，而不是静默改变现有编码。

Edge Slot删除后再次分配时 generation递增；generation即将回绕时该Slot永久退休。第一期中，一个已经分配的 `logical_page_no` 永久绑定其Edge Type、Schema Layout、Slot宽度和逻辑页身份：即使所有Slot都已删除，也不能把该逻辑页号重新格式化或重新分配给另一种Layout。允许在保持全部Slot字节、generation、页面身份和Layout不变的前提下搬迁整个Extent，并原子递增 `extent_epoch`。这样旧EID不会因为页面重格式化而按新的 `slot_size` 命中另一条边。系统在逻辑页号使用率达到75%时告警；回收该限制需要未来EID格式增加页面代次，不能复用当前Slot generation。

### 6.2 External ID

`external_id` 是业务主键，例如 `user_id`、`ad_id`、`campaign_id`。它在 ID Namespace 内唯一且创建后不变。

```text
(graph_id, id_namespace_id, external_id) → VID
```

使用 ID Namespace 而不是 Label，因为一个顶点的 Label 可以增减，业务身份保持不变。

创建顶点时，ID Map、Vertex Slot 和属性记录在同一事务提交。首次查询通过 ID Map 得到 VID；后续遍历直接使用 VID。

### 6.3 三种地址

| 标识 | 含义 | 稳定性 |
|---|---|---|
| External ID | 业务身份 | 稳定 |
| VID/EID | 图内部身份和逻辑槽 | 分区键不变时稳定 |
| `rowid_t` | 当前内部物理位置 | 可能因迁移改变 |

业务表记录通过稳定业务主键关联。物理 RowID 可以缓存，但访问时需要校验。

## 7. 编号化存储关系

```text
01 Graph Partition / Segment Map
 ├─02 Vertex Page
 │   ├─03 Vertex Page Header + PCR ITLs
 │   └─04 Vertex Slot Array
 │       └─05 Vertex Slot
 │           ├─06 vertex_meta
 │           ├─07 Vertex Inline Properties
 │           ├─08 vertex_property_rowid ───────────────┐
 │           ├─09 OUT Adjacency Descriptor             │
 │           ├─10 IN Adjacency Descriptor              │
 ├─14 Adjacency Directory ─────────────────────┐   │
 └─15 Adjacency Data Page                     │   │
     └─16 Adjacency Entry                     │   │
         ├─ neighbor_vid ─────────────────────┼───┤
         └─ full EID ─────────────────────────┤   │
17 Edge Page                                  │   │
 ├─18 Edge Page Header + PCR ITLs             │   │
 └─19 Canonical Edge Slot <───────────────────┘   │
     ├─20 edge_meta + src_vid + dst_vid           │
     ├─21 EDGE Inline Properties                  │
     └─22 edge_property_rowid ───────────────┐    │
23 Property Data Page <──────────────────────┴────┘
 ├─24 Vertex Property Row <─────────────────────────────08
 └─25 Edge Property Row
26 Target Vertex Page
 └─27 VID Locator <──── 16 neighbor_vid
28 EID Locator <─────── 16 full EID

Phase 1不在 `05` 中物理分配编号 `11`–`13`。它们只保留为未来图级统一Inline Edge扩展：

```text
11 Reserved Inline Edge Area
 ├─12 neighbor_vid
 └─13 full EID或edge_token
```

`26` 不是新页面类型，而是通过 `27` 定位到的另一张 `02 Vertex Page`。`27/28` 是运行时定位结果，不持久化到数据页。
```

新增页面类型：

```text
PAGE_TYPE_GRAPH_SEG_HEAD
PAGE_TYPE_GRAPH_EXTENT_MAP
PAGE_TYPE_GRAPH_VERTEX
PAGE_TYPE_GRAPH_EDGE
PAGE_TYPE_GRAPH_ADJ_DIR
PAGE_TYPE_GRAPH_ADJ_DATA
```

属性页复用 `PAGE_TYPE_PCRH_DATA` 和 LOB。所有结构使用显式字节布局和编译期尺寸断言，不使用依赖编译器的位域。

### 7.1 持久化ABI约定

以下结构是设计约束，不允许实现阶段自行改变字段宽度：

- 持久化结构使用 `#pragma pack(4)` 和固定宽度整数；
- Graph ID本身不编码在VID/EID中，所有定位接口必须同时接收 `graph_handle`；
- generation 0无效，首次分配从1开始；即将回绕时退休对应Slot或对象；
- 保留字段、未使用Inline预算和对齐字节写零；
- 所有页先校验oGRAC `page_head_t` 的Page Type、PCN和Checksum，再校验Graph页身份；
- Graph持久化字段使用明确的Little-Endian读写辅助函数；不允许在不同字节序平台直接强转结构；
- `page_head_t`、`page_tail_t` 和 `pcr_itl_t` 继续复用现有32B、8B和24B格式。

固定槽页必须在格式化时预留不可增长的PCR ITL区域。否则扩展ITL会移动Slot起始偏移，破坏VID/EID的直接计算：

```text
slots_per_page是满足以下不等式的最大整数n：

align8(sizeof(page_head_t)
     + sizeof(graph_page_identity_t)
     + sizeof(type_specific_header)
     + itl_capacity * sizeof(pcr_itl_t)
     + ceil(n / 8))
+ n * slot_size
+ sizeof(page_tail_t)
<= page_size

allocation_bitmap_bytes = ceil(slots_per_page / 8)
slot_area_offset = align8(上述固定头、ITL和Bitmap区域)
```

`itl_capacity`、`slot_area_offset`、`slot_size` 和 `slots_per_page` 在页面生命周期内不可改变。ITL耗尽时等待或复用已清理ITL，不能扩展固定ITL区域。

### 7.2 `01 Graph Partition / Segment Map`

每个 `(graph_id, virtual_partition_id, segment_kind)` 有独立逻辑Segment。第一期Segment Kind为：

```text
GRAPH_SEG_VERTEX
GRAPH_SEG_EDGE
GRAPH_SEG_ADJ_DIRECTORY
GRAPH_SEG_ADJ_DATA
```

Extent Map页的Root和Child都必须携带期望的Epoch与Generation，不能只保存Physical Page ID：

```c
typedef struct st_graph_mapping_page_ref {
    pagid_data_t physical_page_id;  /* 6B */
    uint16 flags;
    uint32 ref_epoch;
    uint16 page_generation;
    uint16 reserved_zero;
} graph_mapping_page_ref_t;         /* 16B */
```

每个Segment选择固定的Logical Extent大小，默认128页且必须是2的幂。Segment Head的Graph扩展部分为60B：

```c
typedef struct st_graph_segment_head {
    uint32 graph_id;
    uint32 segment_id;
    uint16 virtual_partition_id;
    uint8  segment_kind;
    uint8  format_version;
    graph_mapping_page_ref_t extent_map_root;
    uint32 next_logical_page_no;
    uint32 allocated_page_count;
    uint32 retired_page_count;
    uint32 route_epoch;
    uint16 default_slot_size;
    uint16 default_slots_per_page;
    uint16 fixed_itl_capacity;
    uint16 flags;
    uint16 logical_extent_pages;     /* default 128 */
    uint16 reserved_zero;
    uint32 free_layout_directory;
} graph_segment_head_t;             /* 60B */
```

主路径使用Extent Map。每个20B Entry把一个Logical Extent映射到一个连续Physical Extent：

```c
typedef struct st_graph_extent_map_entry {
    pagid_data_t physical_first_page; /* 6B */
    uint16 page_count;
    uint32 extent_epoch;
    uint32 mapping_version;
    uint16 allocation_generation;
    uint16 flags;                   /* PRESENT/RETIRED */
} graph_extent_map_entry_t;         /* 20B */
```

`allocation_generation` 在Physical Extent每次被空间分配器重新分配时递增，并与表空间分配记录及DRC Extent映射资源身份比较，用于阻止旧Extent映射ABA；它不能替代其中每个Logical Page的 `page_generation`。纯物理搬迁必须保留所有Logical Page的Page Generation。

`MAPPING_BUSY` 仅是Logical Extent Mapping X Guard保护下的易失运行态，不写入持久化Entry，避免Crash留下永久阻塞状态。

`PAGE_TYPE_GRAPH_EXTENT_MAP` 每张Leaf保存256个Extent Entry。普通Segment使用Chunked Extent Array；超过单层容量或Logical Extent空间稀疏时，`level>0` 页面使用256个Child Reference按Logical Extent Number组织：

```c
typedef struct st_graph_extent_map_header {
    uint32 graph_id;
    uint32 segment_id;
    uint32 base_logical_extent_no;
    uint16 virtual_partition_id;
    uint8  segment_kind;
    uint8  level;                   /* 0=256个Extent Entry */
    uint16 entry_count;
    uint16 flags;
    uint32 chunk_epoch;
} graph_extent_map_header_t;        /* 24B */

typedef graph_mapping_page_ref_t graph_extent_map_child_t; /* 16B */
```

Extent Map每层固定Fanout 256。Level 0覆盖256个Logical Extent，Level L覆盖 `256^(L+1)` 个Logical Extent；覆盖范围和Base对齐使用64-bit Checked Arithmetic，避免32-bit Root计算溢出。`base_logical_extent_no` 必须按本层覆盖范围对齐，分支下标按Logical Extent Number对应层的8-bit数字计算。

正常页面定位：

```text
logical_extent_no = logical_page_no / logical_extent_pages
page_offset       = logical_page_no % logical_extent_pages
extent_entry      = lookup_extent(logical_extent_no)
physical_page_id  = extent_entry.physical_first_page + page_offset
```

最终解析顺序固定为：

```text
1. 获取Logical Extent的Mapping Read Guard
2. 根据logical_extent_no读取Extent Entry并校验：
      PRESENT=1，RETIRED=0
      page_offset < page_count <= logical_extent_pages
      allocation_generation匹配表空间分配记录及DRC Extent资源身份
3. 使用Checked Page-ID Addition计算physical_first_page + page_offset，
   Physical Extent不得跨Datafile边界
4. 将physical page_id交给DRC定位Page Owner并Pin Buffer
5. 校验Page Identity.mapping_epoch == Extent Entry.extent_epoch
6. 复读Guard保护下的Extent Entry.mapping_version
7. 释放Mapping Read Guard；Buffer Pin继续保护物理页
```

如果步骤5或6发现身份或版本不匹配，必须Unpin已进入的Buffer、丢弃对应Extent翻译缓存、释放Guard并从步骤1重新解析；禁止返回旧Resolution。

Extent Cache Key是 `(graph_id, vpart, segment_kind, logical_extent_no)`，Cache Entry必须同时携带并匹配当前 `mapping_version` 和 `allocation_generation`；DRC Extent映射资源也携带相同Allocation Generation。普通热路径只执行一次Extent Cache查询、代次比较和页内偏移加法。

每个Logical Extent有集群级Mapping S/X Guard，可复用DRC的新映射资源类型：

- Reader持有S Guard直到完成映射解析、Buffer Pin和Page Identity复验；
- Writer持有S Guard直到取得页面DRC X和Page Latch、再次校验mapping_version、完成页面修改并生成Redo；
- Publisher持有X Guard完成最终复制、WAL持久化、缓存失效确认和映射发布；
- X Guard阻止新的Reader取得旧版本，并等待已存在的S Guard退出；
- `graph_page_resolve()` 必须返回或持有Guard，不能只返回一个可能立即过期的Physical Page ID。

Per-Extent Guard只串行化同一Logical Extent，不能代替共享Extent Map页的并发控制。固定Entry更新必须取得Leaf Page DRC X/Page Latch。`graph_mapping_page_ref_t.ref_epoch` 必须等于所引用Extent Map节点的 `chunk_epoch`。

Extent级Radix增长固定使用完整修改路径Copy-on-Write：从待修改Leaf到Root的所有路径节点都创建不可达副本，副本内的Child Reference从下到上组装；每个新节点递增 `chunk_epoch`，其Parent中的新Reference同步设置 `ref_epoch`。不得原地CAS已发布Parent的Child Reference。WAL发布顺序固定为“全部新路径节点内容与Redo持久化 → 以完整旧Root Reference元组 `(physical_page_id, ref_epoch, page_generation)` 对Segment Head中的Root执行唯一CAS → Commit/Flush”。恢复时只允许旧Root完整可达或新Root及其全部子节点完整可达，不能出现已发布Parent指向未持久化Child。Root CAS失败时丢弃整条未发布新路径并从Root重试；CAS失败或Crash时未被已提交Root引用的新节点作为孤儿空间回收。

整Extent搬迁流程：

```text
获取Logical Extent Mapping X Guard，进入易失的MAPPING_BUSY状态
 → 按Logical Page顺序获取所有源页DRC X与Page Latch，冻结写入
 → 复制到连续的新Physical Extent
 → 保留每页page_generation，写入新的extent_epoch
 → 校验页面身份与Checksum
 → 先持久化新页面Redo/数据，再写映射发布Redo
 → 原子发布新Extent Entry：
      mapping_version递增
 → 广播并等待Extent翻译缓存失效确认
 → 失效旧DRC资源后释放写冻结和Mapping X Guard
 → 等待旧Buffer Pin、DCS传输和延迟响应结束
 → 回收旧Extent
```

映射切换对查询SCN透明，不创建两个逻辑版本；发布前旧映射有效，发布后新映射有效。复制期间禁止未捕获的并发写；第一期使用上述阻塞式X Guard和Page Latch，后续才考虑Redo Catch-up或Dual Write。映射变更按Logical Extent串行化；实现也可以对预期 `mapping_version` 使用CAS，失败后重新解析并重试。

WAL顺序必须保证“新页面内容 → 新映射元数据 → Commit/Flush”。恢复后只能得到完整旧映射或完整新映射；未发布的新Extent作为孤儿空间回收。

旧Physical Page复用前必须完成DRC失效确认、DCS传输结束、Buffer Pin归零、Mapping Read Guard退出以及延迟响应隔离。仅依赖Page Identity不构成ABA屏障。

`extent_epoch`、`ref_epoch`、`mapping_version` 和 `chunk_epoch` 不得在陈旧引用仍可能存在时回绕复用。接近回绕时执行集群映射缓存与DRC全局静默并推进Allocation/Page Generation；无法证明安全时退休对应映射身份。

第一期Graph坏页修复复用oGRAC现有ABR的页面获取、Redo回放和原位写回机制，不改变Graph逻辑到物理映射。现有ABR只校验通用PCN、Page ID或Checksum，Graph Page接入时必须增加写回前身份校验：

```text
Standby ABR：
  修复后的页面字节 → 原Buffer及原page_id
  → 写回原Datafile的原Page Offset

Backup + Redo ABR：
  从Backup取得同一page_id
  → 应用Redo到最新状态
  → RESTORE FLUSHPAGE写回原file/page offset
```

ABR请求上下文按Page Type取得期望身份：

- Vertex、Edge、Adjacency数据页：从受Mapping Read Guard保护的Extent Entry和Logical Page Offset取得；
- Segment Head：从Catalog中的Segment入口取得；
- Root Extent Map页：从Segment Head的 `extent_map_root` 取得；
- Child Extent Map页：从受保护的Parent Reference取得。

上下文必须包含期望的Page ID、Page Type，以及适用于该页的Graph、Segment、Virtual Partition、Segment Kind、Logical Page/Base、Mapping Epoch、Page Generation和Layout Level。接收或Redo完成后，必须先校验修复页的Page ID、Graph Page Identity、Page Type和PCN，再计算/验证Checksum；任一项不匹配都必须中止，禁止重算Checksum后写回。原位写回成功后再次校验Buffer中的Graph Page Identity和Checksum。

若原位置永久不可写，由存储层完成坏块重映射，或由数据库执行上述整Extent搬迁；第一期不提供Graph单页映射覆盖。Catalog仅预留 `GRAPH_MAP_FEATURE_PAGE_EXCEPTION` 格式特性位，第一期始终关闭且不分配对应Page Type或持久化结构；未来启用必须增加Mapping Format Version并提供迁移流程。

Segment Head与映射元数据页的公共身份规则为：

```text
Segment Head：
  logical_page_no    = UINT32_MAX
  mapping_epoch      = 0
  page_generation    = 1
  page_layout_version= Segment Format Version

Extent Map Page：
  logical_page_no    = base_logical_extent_no
  mapping_epoch      = Parent Reference.ref_epoch
  page_generation    = Parent Reference.page_generation
  page_layout_version= Extent Map level
```

映射元数据页首次分配generation为1；同一物理页回收并用于新的映射页身份时递增generation，回绕前退休。读取时同时校验Base、Level和Segment ID，因此不同层级或覆盖范围不能互相替代。

数据页映射Key是：

```text
(graph_id, virtual_partition_id, segment_kind, logical_extent_no)
```

Extent Map更新与页面分配使用同一事务和Redo。读取物理页后必须反向校验Graph、Segment、Virtual Partition、Logical Page和Extent Epoch。Page Generation由Adjacency Root Locator等上层对象身份校验，不能与 `allocation_generation` 混用。Vertex Logical Page使用32 bit；Edge EID只允许24-bit Logical Page；Adjacency Locator使用32-bit Logical Page。

### 7.3 Graph页面公共身份

每张Graph页面在现有 `page_head_t` 后保存32B公共身份：

```c
typedef struct st_graph_page_identity {
    uint32 graph_id;
    uint32 segment_id;
    uint32 logical_page_no;
    uint32 mapping_epoch;
    uint16 virtual_partition_id;
    uint16 page_layout_version;
    uint16 page_generation;
    uint16 header_bytes;
    uint8  segment_kind;
    uint8  flags;
    uint16 reserved_zero;
    uint32 identity_crc;
} graph_page_identity_t;            /* 32B */
```

`page_generation` 保护页级Root Locator；`mapping_epoch` 始终取Extent Entry的 `extent_epoch`，防止读取到搬迁前的物理映射。`identity_crc` 计算时把自身字段置零，只覆盖其余Graph身份和类型扩展头。整页Checksum仍由现有Page Tail提供。

Adjacency/Directory Logical Page第一次格式化时 `page_generation=1`。第一期不把已回收的Adjacency/Directory Logical Page Number重新分配给新的页面身份，而是永久退休；Object Slot可以在原页面身份内通过自身generation复用。因此8B `graph_adj_locator_t` 不会发生“Page Generation和Object Generation同时回绕”的ABA。整Extent物理搬迁保持Page Generation不变、只递增Extent Epoch。

### 7.4 Locator分类

持久化邻接Locator使用8B：

```c
typedef struct st_graph_adj_locator {
    uint32 logical_page_no;
    uint16 object_slot;
    uint16 generation;
} graph_adj_locator_t;              /* 8B */
```

语义：

- `logical_page_no=UINT32_MAX && generation=0`：无效Locator；
- `object_slot=UINT16_MAX`：Root是整张页面，generation校验Page Identity；
- 其他值：Root是Slotted Page中的对象，generation校验对象槽；
- Locator省略Graph、Virtual Partition和Segment Kind，它们由当前Graph、Owner VID、方向和Mode确定。

属性仍使用现有8B `rowid_t`，但Property Row必须反向校验Owner ID和generation。VID/EID Locator是运行时结构：

```c
typedef struct st_graph_page_resolution {
    page_id_t physical_page_id;
    uint32 mapping_epoch;
    uint32 mapping_version;
    uint16 page_generation;
    uint16 reserved_zero;
} graph_page_resolution_t;         /* 20B */

typedef struct st_graph_slot_location {
    page_id_t physical_page_id;
    uint32 mapping_epoch;
    uint32 mapping_version;
    uint16 slot_id;
    uint16 slot_size;
} graph_slot_location_t;            /* 20B */
```

`27 VID Locator` 和 `28 EID Locator` 都使用该结构；它们只存在于Buffer/执行器上下文中。Mapping Guard是运行时不透明对象，不进入持久化结构。

`graph_page_resolution_t.page_generation=0` 表示Extent映射层没有Page Generation期望值，调用方必须用VID/EID或Adjacency Root语义完成对象代次校验。

### 7.5 编号结构定义索引

| 编号 | 具体定义 |
|---|---|
| 01 | `graph_segment_head_t`及Extent Map结构 |
| 02–04 | `graph_page_identity_t`、`graph_vertex_page_header_t`、固定ITL/Bitmap/Slot Array布局 |
| 05–10 | 三种 `graph_vertex_slot_*_t`、`graph_vertex_meta_t`、`graph_adj_descriptor_t` |
| 11–13 | Phase 1不持久化；仅保留未来图级Inline Edge格式 |
| 14 | `graph_adj_dir_page_header_t`、`graph_adj_dir_entry_t`、Packed/Shard/Migration对象 |
| 15–16 | `graph_adj_data_page_header_t`、`graph_adj_slot_t`、`graph_adj_entry_base_t`及方向Inline Payload |
| 17–22 | `graph_edge_page_header_t`、四种 `graph_edge_slot_*_t`、`graph_edge_meta_t` |
| 23–25 | 现有PCR Heap Page/Row加 `graph_property_row_header_t` |
| 26 | 与02相同，由目标VID定位，不是新Page Type |
| 27–28 | 运行时 `graph_slot_location_t`，不持久化 |

## 8. 顶点存储

### 8.1 图级固定槽宽

同一个图中的 `05 Vertex Slot` 宽度固定。创建图时从 0B、8B 和 16B 三档 Vertex Inline Property 预算中选择，默认 0B：

| 编号 | 字段 | 大小 | 用途 |
|---|---|---:|---|
| 06 | `vertex_meta` | 16B | 状态、代次、Label Set、Schema、ITL、邻接模式 |
| 07 | `inline_properties` | 0B、8B 或 16B | 定长热顶点属性 |
| 08 | `property_rowid` | 8B | 指向 `24 Vertex Property Row` |
| 09 | `out_adj` | 12B | 出邻接根位置和总出度 |
| 10 | `in_adj` | 12B | 入邻接根位置和总入度 |

三档布局分别为：

| Graph Profile | Inline 预算 | Vertex Slot 总宽度 |
|---|---:|---:|
| `COMPACT` | 0B | 48B |
| `INLINE_8` | 8B | 56B |
| `INLINE_16` | 16B | 64B |

0B 表示不保存 Inline Property 值，但 `vertex_meta`、两位状态位和 `property_rowid` 仍然存在。不同 Vertex Type 可以定义不同的 Inline 表达式，但编码总宽度不得超过图级预算；未使用的预算字节写零。图级固定槽宽使 VID 定位不依赖 Vertex Type 或 `schema_layout_id`。

```c
typedef struct st_graph_vertex_meta {
    uint16 flags;
    uint16 generation;
    uint32 label_set_id;
    uint16 schema_layout_id;
    uint16 inline_property_state_bitmap;
    uint8  itl_id;
    uint8  reserved_zero[2];
    uint8  adjacency_modes;
} graph_vertex_meta_t; /* 16B */
```

`flags` 的持久化位定义为：

```text
bit 0  ALLOCATED
bit 1  DELETED
bit 2  HAS_PROPERTY_ROW
bit 3  OUT_ADJ_PRESENT
bit 4  IN_ADJ_PRESENT
bit 5  SUPERNODE
bit 6  MIGRATING
bit 7  FROZEN
bit 8–15 reserved
```

删除标记受MVCC保护，旧快照结束后才允许回收。读取VID时必须同时满足 `ALLOCATED`、非可见删除且 `VID.generation == vertex_meta.generation`。

`label_set_id` 指向一组 Label。`schema_layout_id` 决定当前图级预算中的热属性如何解码。`inline_property_state_bitmap` 使用两位表示一个 Inline 投影的状态：

```text
00：NULL或属性不存在
01：Inline值有效
10：Inline值失效，必须回读规范原值
11：保留
```

16 bit 最多描述 8 个 Inline 投影。Schema Layout 为每个投影分配稳定的 ordinal 0–7；`bits[2*n, 2*n+1]` 对应 ordinal `n`。不在 Inline Layout 中的冷属性不占 ordinal。`itl_id` 指向 `03` 中的 PCR ITL。

OUT和IN可以采用不同的邻接模式，因此 `adjacency_modes` 拆成两个Nibble：

```text
bits 0–3：OUT mode
bits 4–7：IN mode

0 NO_ADJACENCY
1 DIRECT
2 PACKED_DIRECTORY
3 EXTERNAL_DIRECTORY
4 SUPERNODE_SHARDED
5 MIGRATING
6–15 reserved
```

实现使用掩码和移位读写，不使用C位域。`09 out_adj` 和 `10 in_adj` 使用完全相同的12B结构：

```c
typedef struct st_graph_adj_descriptor {
    graph_adj_locator_t root;       /* 8B */
    uint32 edge_count_hint;         /* 4B */
} graph_adj_descriptor_t;           /* 12B */
```

Descriptor描述整个方向的邻接集合，不描述单条边，因此不保存 `neighbor_vid`。每个邻居的完整64-bit VID保存在 `16 Adjacency Entry` 中。Root按Mode解释：

| Mode | `root` 指向 |
|---|---|
| `NO_ADJACENCY` | 无效Locator |
| `DIRECT` | 单一Edge Type的第一张Adjacency Data Page |
| `PACKED_DIRECTORY` | Packed Directory对象 |
| `EXTERNAL_DIRECTORY` | External Directory Root对象 |
| `SUPERNODE_SHARDED` | Shard Directory对象 |
| `MIGRATING` | 包含old/new root和switch SCN的Migration对象 |

`edge_count_hint` 在 `DIRECT/PACKED` 下是精确值；在 `EXTERNAL/SUPERNODE/MIGRATING` 下是优化器统计提示，`UINT32_MAX` 表示饱和。执行器只能用它估算成本，不能用提示值0直接判定没有边；精确64-bit计数保存在Directory Entry或Shard Directory中。

三个Vertex Slot档位具有明确物理偏移：

```c
typedef struct st_graph_vertex_slot_0 {
    graph_vertex_meta_t meta;       /* offset 0  */
    rowid_t property_rowid;         /* offset 16 */
    graph_adj_descriptor_t out_adj; /* offset 24 */
    graph_adj_descriptor_t in_adj;  /* offset 36 */
} graph_vertex_slot_0_t;            /* 48B */

typedef struct st_graph_vertex_slot_8 {
    graph_vertex_meta_t meta;       /* offset 0  */
    uint8 inline_values[8];         /* offset 16 */
    rowid_t property_rowid;         /* offset 24 */
    graph_adj_descriptor_t out_adj; /* offset 32 */
    graph_adj_descriptor_t in_adj;  /* offset 44 */
} graph_vertex_slot_8_t;            /* 56B */

typedef struct st_graph_vertex_slot_16 {
    graph_vertex_meta_t meta;       /* offset 0  */
    uint8 inline_values[16];        /* offset 16 */
    rowid_t property_rowid;         /* offset 32 */
    graph_adj_descriptor_t out_adj; /* offset 40 */
    graph_adj_descriptor_t in_adj;  /* offset 52 */
} graph_vertex_slot_16_t;           /* 64B */
```

编译期必须断言三个结构的总大小和每个字段偏移。

### 8.2 顶点页

`02 Vertex Page` 从低地址到高地址为：

```text
page_head_t
graph_page_identity_t
graph_vertex_page_header_t
fixed PCR ITL array
allocation bitmap
fixed vertex slots
page_tail_t
```

`03 Vertex Page Header` 的类型扩展部分为24B：

```c
typedef struct st_graph_vertex_page_header {
    uint16 slot_size;               /* 48/56/64 */
    uint16 slots_per_page;
    uint16 used_slots;
    uint16 free_hint;
    uint16 bitmap_offset;
    uint16 bitmap_bytes;
    uint16 itl_offset;
    uint16 slot_area_offset;
    uint8  itl_count;
    uint8  itl_capacity;
    uint16 flags;
    uint32 reserved_zero;
} graph_vertex_page_header_t;       /* 24B */
```

Allocation Bitmap表示Slot是否已物理分配或仍被旧快照保留；可见性仍由Slot Meta和PCR MVCC决定。删除后的Slot只有在所有旧快照结束后才清除Bitmap并进入可复用状态。

`04` 中的槽宽在图级别固定，因此：

```text
logical_page = local_sequence / slots_per_page
slot_id      = local_sequence % slots_per_page
page_id      = graph_page_resolve(VERTEX, logical_page)
```

`slots_per_page` 由页面大小、页头、ITL 和图级槽宽计算，并写入 Graph Catalog、Segment Header 与 Page Header。Vertex Inline预算创建图后不可在线修改，因为改变槽宽会改变所有 `local_sequence` 对应的逻辑页和槽号；需要更大预算时必须重建图或执行未来提供的整图重写流程。

### 8.3 可选 Inline Edge

图级固定 Vertex Slot 后，不能再按 Vertex Type 追加不同大小的 `11 Optional Inline Edge Area`，否则同一图的槽宽将不再固定。因此第一期不在 Vertex Slot 中启用 Inline Edge，编号 `11`–`13` 只作为未来格式扩展点；低度边使用 Packed Small Adjacency Page。

如果后续基准测试证明 Inline Edge 的收益足够大，只能在创建图时增加统一的 `VERTEX_INLINE_ADJACENCY_SIZE`，把固定的 OUT/IN 数组计入该图所有 Vertex Slot，而不能按 Vertex Type 改变槽宽。预留的元素格式为：

```c
typedef struct st_graph_inline_edge_ref {
    graph_vid_t neighbor_vid;       /* 12 */
    graph_edge_token_t edge_token;  /* 13，仅混合 Edge Type 使用 */
    uint8 inline_properties[N];     /* 方向对应的 OUT 或 IN Inline Properties */
} graph_inline_edge_ref_t;
```

未来只评估每个方向 1 或 2 条，最多评估到4条。8条和12条不纳入设计，因为它们显著降低顶点页密度，并让每次增删边都修改顶点页。

普通邻接页保存完整 EID。`edge_token` 只在混合多个 Edge Type 的顶点内嵌数组中使用，其布局为：

```text
edge_local_sequence 32 bit
generation          16 bit
compact_type_code   16 bit
```

它省略分区，换取 Edge Type。出边分区取当前顶点分区，入边分区取 `neighbor_vid` 中的源顶点分区。

## 9. 规范边存储

每条逻辑边有唯一的 `19 Canonical Edge Slot`。边不要求整个图使用同一种槽宽，而是保证单个 Edge Page 内的 Slot 固定。基础 Slot 为 40B，按 `EDGE INLINE PROPERTIES` 的编码预算扩展为 48B、56B 或 64B：

| 编号 | 内容 | 大小 | 用途 |
|---|---|---:|---|
| 20 | `edge_meta + src_vid + dst_vid` | 32B | 类型、代次、Schema、ITL、端点 |
| 21 | EDGE Inline Properties | 0B–24B | 边属性的规范热值 |
| 22 | `property_rowid` | 8B | 指向 `25 Edge Property Row` |

计算公式为：

```text
slot_size = align8(40 + edge_inline_value_size)
```

| Edge Slot Profile | Inline Value 预算 | Slot 宽度 |
|---|---:|---:|
| `COMPACT` | 0B | 40B |
| `INLINE_8` | 1B–8B | 48B |
| `INLINE_16` | 9B–16B | 56B |
| `INLINE_24` | 17B–24B | 64B |

Edge Meta 中的 `inline_property_state_bitmap` 不占用上述 Inline Value 预算。默认 `EDGE_SLOT_PROFILE=AUTO`，系统选择容纳声明表达式的最小档位。用户也可以显式指定 Profile 为后续 Schema 预留空间；实际编码宽度超过显式预算时 DDL 失败，不允许任意字节宽度以避免产生过多页面类别。

```c
typedef struct st_graph_edge_meta {
    uint16 flags;
    uint16 generation;
    uint32 edge_type_id;
    uint16 schema_layout_id;
    uint16 inline_property_state_bitmap;
    uint8  itl_id;
    uint8  property_mode;
    uint8  validity_flags;
    uint8  reserved_zero;
} graph_edge_meta_t;                /* 16B */
```

Canonical Edge Slot的共同前缀和物理偏移为：

```c
typedef struct st_graph_edge_slot_0 {
    graph_edge_meta_t meta;         /* offset 0  */
    graph_vid_t src_vid;            /* offset 16 */
    graph_vid_t dst_vid;            /* offset 24 */
    rowid_t property_rowid;         /* offset 32 */
} graph_edge_slot_0_t;              /* 40B */

typedef struct st_graph_edge_slot_8 {
    graph_edge_slot_0_t base;       /* offset 0  */
    uint8 inline_values[8];         /* offset 40 */
} graph_edge_slot_8_t;              /* 48B */

typedef struct st_graph_edge_slot_16 {
    graph_edge_slot_0_t base;       /* offset 0  */
    uint8 inline_values[16];        /* offset 40 */
} graph_edge_slot_16_t;             /* 56B */

typedef struct st_graph_edge_slot_24 {
    graph_edge_slot_0_t base;       /* offset 0  */
    uint8 inline_values[24];        /* offset 40 */
} graph_edge_slot_24_t;             /* 64B */
```

编号 `21 EDGE Inline Properties` 在逻辑关系图中列于 `22` 之前，但物理上放在40B公共前缀之后，使四个档位共享相同的Meta、端点和Property RowID偏移。

`17 Edge Page` 统一使用 `PAGE_TYPE_GRAPH_EDGE`，不为 40B、48B、56B、64B 分别增加 Page Type。具体页面类别由以下字段共同确定：

```text
graph_id
+ PAGE_TYPE_GRAPH_EDGE
+ edge_type_id
+ schema_layout_id
+ page_layout_version
```

页面头持久化上述全部身份字段以及 `slot_size` 和 `slots_per_page`。同一页面中的所有Slot必须具有相同的Edge Type、Schema Layout和Slot宽度；页面清空后这些字段也不可改变。`edge_type_id` 虽然可以通过 `schema_layout_id` 查询，仍冗余保存在页头中，用于恢复、页面转储和损坏校验。

`17/18` 的完整页面布局为：

```text
page_head_t
graph_page_identity_t
graph_edge_page_header_t
fixed PCR ITL array
allocation bitmap
fixed canonical edge slots
page_tail_t
```

`18 Edge Page Header` 的类型扩展部分为32B：

```c
typedef struct st_graph_edge_page_header {
    uint32 edge_type_id;
    uint16 schema_layout_id;
    uint16 slot_size;               /* 40/48/56/64 */
    uint16 slots_per_page;          /* 必须 <= 256 */
    uint16 used_slots;
    uint16 free_hint;
    uint16 bitmap_offset;
    uint16 bitmap_bytes;
    uint16 itl_offset;
    uint16 slot_area_offset;
    uint8  itl_count;
    uint8  itl_capacity;
    uint16 flags;
    uint8  reserved_zero[6];
} graph_edge_page_header_t;         /* 32B */
```

EID的8-bit `slot_id` 允许0–255，页面格式化时必须验证 `slots_per_page <= 256`。Allocation Bitmap是Slot复用和恢复扫描的物理分配依据；边的查询可见性仍由Meta、ITL和Undo决定。

每个 Virtual Partition 使用一个逻辑 Edge Segment，内部页面可以属于不同 Edge Type 和Layout。Segment按Layout维护空闲页列表。EID中的 `logical_page_no` 通过Extent Map计算物理 `page_id`，再使用页面头中的固定 `slot_size` 计算：

```text
slot_address = slot_area + slot_id * slot_size
```

随后校验 Slot generation。这样不同Edge Type可以使用不同槽宽，同时保持EID直接定位。

规范边是属性事实来源。出邻接和入邻接中的属性是为遍历准备的副本。

`flags` 表示已使用、已删除、存在属性行、存在出邻接副本、存在入邻接副本、时态边、关联向量和迁移状态。`property_mode` 表示属性位于规范槽、规范槽加属性行、仅属性行、LOB 或向量引用。

跨分区边必须满足以下损坏校验规则：

```text
Canonical Edge所在Virtual Partition == src_vid.virtual_partition_id
OUT Adjacency所在Virtual Partition    == src_vid.virtual_partition_id
IN Adjacency所在Virtual Partition     == dst_vid.virtual_partition_id
EID.virtual_partition_id              == src_vid.virtual_partition_id
OUT Entry.neighbor_vid                == dst_vid
IN Entry.neighbor_vid                 == src_vid
```

## 10. 邻接目录

### 10.1 作用

`14 Adjacency Directory` 是顶点级 Edge Type 路由表：

```text
(vertex, direction, edge_type)
    → format、count、schema、sort、shard policy、root
```

它解决以下场景：

- 一个顶点同时拥有 `CLICKED`、`SEARCHED`、`MANAGES` 等多种边；
- OUT 与 IN 的属性宽度、排序和过滤方式不同；
- 优化器需要按 Edge Type 读取度数；
- 超级节点需要保存大量分片入口；
- 在线重建时原子切换根指针；
- 每种边独立维护 Zone Map、Bloom Filter 和压缩格式。

如果所有边共用一个邻接指针，查询 `MANAGES` 时可能扫描大量 `CLICKED` 边。目录让查询直接进入目标 Edge Type。

### 10.2 为什么不放进顶点槽

Edge Type 数量随顶点变化。把完整目录放进固定槽会浪费低度顶点空间，并在增加 Edge Type 时破坏 O(1) 槽定位。顶点槽只保留 12B 的 `09/10` 描述符。

### 10.3 四种物理形态

目录是逻辑对象。`DIRECT` 不分配Directory Page；进入 `PACKED/EXTERNAL/SHARDED` 后，Directory Page由一个 `owner_vid + direction` 独占，但同页保存该方向的多个Edge Type，不为每个Edge Type单独分配页面：

| 模式 | 场景 | 物理形式 |
|---|---|---|
| `DIRECT` | 单一 Edge Type | 描述符直接指向邻接页 |
| `PACKED_DIRECTORY` | 少量 Edge Type | 小目录和邻接项共享页面 |
| `EXTERNAL_DIRECTORY` | 多 Edge Type | 独立目录页 |
| `SUPERNODE_SHARDED` | 超高度数 | 目录指向分片目录 |

外部目录项为 32B：

```c
typedef struct st_graph_adj_dir_entry {
    uint32 edge_type_id;
    uint8  direction;
    uint8  entry_format;
    uint16 shard_count;
    uint64 edge_count;
    graph_adj_locator_t root_locator;
    uint32 flags;
    uint16 schema_layout_id;
    uint8  shard_policy;
    uint8  sort_key_id;
} graph_adj_dir_entry_t;            /* 32B */
```

Directory和Adjacency Data都采用带generation和ITL的8B Side Slot：

```c
typedef struct st_graph_adj_object_slot {
    uint16 object_offset;
    uint16 object_length;
    uint16 generation;
    uint8  itl_id;
    uint8  flags;                    /* ALLOCATED/DELETED/MOVED */
} graph_adj_object_slot_t;           /* 8B */
```

`object_slot` 是稳定数组下标，页面整理只能修改 `object_offset`，不能改变Slot编号和generation。Directory Entry的MVCC归属于对应Side Slot，不在32B Payload中重复保存ITL。

`PAGE_TYPE_GRAPH_ADJ_DIR` 页面由以下部分组成：

```text
page_head_t
graph_page_identity_t
graph_adj_dir_page_header_t
directory/packed/shard/migration object payloads →
free space
← graph_adj_object_slot_t array
← PCR ITL array
page_tail_t
```

类型扩展头为48B：

```c
typedef struct st_graph_adj_dir_page_header {
    graph_vid_t owner_vid;
    uint8  direction;
    uint8  directory_level;
    uint16 object_count;
    uint32 live_object_count;
    uint32 directory_version;
    graph_adj_locator_t next_page;
    graph_adj_locator_t prev_page;
    uint16 free_begin;
    uint16 free_end;
    uint8  itl_count;
    uint8  itl_capacity;
    uint16 flags;                    /* PACKED/EXTERNAL/SHARD/MIGRATION */
    uint32 reserved_zero;
} graph_adj_dir_page_header_t;       /* 48B */
```

Packed模式在同类Slotted Page中保存小目录及其小邻接列表：

```c
typedef struct st_graph_packed_adj_list_header {
    uint32 edge_type_id;
    uint16 schema_layout_id;
    uint16 entry_size;
    uint32 entry_count;
    uint32 flags;
} graph_packed_adj_list_header_t;    /* 16B，随后是Adjacency Entry数组 */
```

Packed Directory Entry的 `root_locator` 指向该对象。对象可以与目录在同页；空间不足或数量超过阈值时，创建External Directory和Adjacency Data Page，再以MVCC版本化方式切换Vertex Descriptor Root。

Packed List是低度场景的单个MVCC对象：对应的 `graph_adj_object_slot_t.itl_id` 管理整个List，增删边时复制并版本化完整小List，不给内部每条Entry分配独立Side Slot。达到大小或更新频率阈值后必须晋升External模式；External Adjacency Data Page中的每条Entry才使用独立 `graph_adj_slot_t` 和ITL。

超级节点的Directory Entry指向Shard Entry数组：

```c
typedef struct st_graph_adj_shard_entry {
    uint16 shard_id;
    uint16 flags;
    uint32 lower_bound;
    uint32 upper_bound;
    uint64 edge_count;
    graph_adj_locator_t root_locator;
    uint32 route_epoch;
} graph_adj_shard_entry_t;           /* 32B */
```

`lower_bound/upper_bound` 按Shard Policy解释为Hash、VID范围或时间桶边界。迁移对象为：

```c
typedef struct st_graph_adj_migration {
    graph_adj_locator_t old_root;
    graph_adj_locator_t new_root;
    knl_scn_t switch_scn;
    uint32 route_epoch;
    uint8 old_mode;
    uint8 new_mode;
    uint16 flags;
} graph_adj_migration_t;             /* 32B */
```

查询SCN早于 `switch_scn` 时走old root，晚于或等于它时走new root。未完成切换时允许双读，但必须按EID去重。

## 11. 邻接项与三类 Inline Properties

### 11.1 邻接项

`16 Adjacency Entry` 的基础部分：

```c
typedef struct st_graph_adj_entry_base {
    graph_vid_t neighbor_vid;
    graph_eid_t eid;
} graph_adj_entry_base_t; /* 16B */
```

随后可追加该方向的 Inline Properties。页面格式记录：

```text
entry_size = 16B + direction_inline_size
```

`direction_inline_size` 当前支持 0B、8B 和 16B。

OUT/IN属性负载使用相同的两位状态编码，但状态位包含在方向预算内。0B档位没有Payload结构；8B和16B使用两个明确结构，避免零长度或负长度数组：

```c
typedef struct st_graph_adj_inline_payload_8 {
    uint16 inline_property_state_bitmap;
    uint8 encoded_values[6];
} graph_adj_inline_payload_8_t;      /* 8B */

typedef struct st_graph_adj_inline_payload_16 {
    uint16 inline_property_state_bitmap;
    uint8 encoded_values[14];
} graph_adj_inline_payload_16_t;     /* 16B */
```

因此 8B 负载实际提供 6B 编码值，16B 负载提供 14B 编码值，且每个方向最多定义 8 个 Inline 投影。Vertex/Edge Slot 的状态 Bitmap 位于 Meta 中，不占用各自 16B/24B Inline Value 区。

邻接页使用与Directory相同的8B Side Slot：

```c
typedef struct st_graph_adj_slot {
    uint16 entry_offset;
    uint16 entry_length;
    uint16 generation;
    uint8  itl_id;
    uint8  flags;
} graph_adj_slot_t;                  /* 8B */
```

它保存项偏移、长度、代次、PCR ITL和删除标志。`entry_length` 必须等于页面声明的 `16 + direction_inline_size`。邻接页只包含一个 `owner_vid + direction + edge_type`，因此项内不重复保存这些字段。

OUT和IN的Entry结构相同，但 `neighbor_vid` 语义不同：

```text
OUT Entry：neighbor_vid = dst_vid
IN Entry： neighbor_vid = src_vid
```

`neighbor_vid` 是完整64-bit VID，包含对方顶点的Virtual Partition；跨分区IN遍历不依赖当前顶点分区推测源点位置。

### 11.2 用户可见的三类属性

| 用户语法 | 编号 | 物理位置 | 用途 |
|---|---|---|---|
| `EDGE INLINE PROPERTIES` | 21 | 规范 Edge Slot | 无损值或可验证派生投影，用于按 EID 点查 |
| `OUT INLINE PROPERTIES` | 16 的出方向副本 | 出邻接项 | 正向遍历过滤 |
| `IN INLINE PROPERTIES` | 16 的入方向副本 | 入邻接项 | 反向遍历过滤 |
| 未声明 Inline | 25 | Edge Property Row | JSON、大对象、低频属性 |

同一逻辑属性定义一次，通过三个存储子句声明物理投影位置。OUT 与 IN 可以不同，例如点击边的正向查询需要权重，反向查询只需要时间和状态。

适合复制的属性是定长、高频过滤、低频更新、可压缩且选择性较高的属性。字符串、JSON、LOB、完整向量和高频计数进入属性行。

更新时，事务同时修改规范值和所有已配置副本：

```text
EDGE INLINE 中存在：更新 19
OUT INLINE 中存在：更新源顶点出邻接项
IN INLINE 中存在：更新目标顶点入邻接项
```

若属性只配置在 OUT 或 IN，或者 Inline 值由任意表达式派生、使用 `HASH32_VERIFY`、或配置 `OVERFLOW=WARNING`，其规范原值必须存放在 `25`。只有可逆、无损且使用 `OVERFLOW=REJECT` 的直接属性转换，才允许由 `21` 单独承担规范值。

### 11.3 页面过滤

`15 Adjacency Data Page` 的页面布局为：

```text
page_head_t
graph_page_identity_t
graph_adj_data_page_header_t
Zone Map / Bloom元数据
Adjacency Entry payloads →
free space
← graph_adj_slot_t array
← PCR ITL array
page_tail_t
```

类型扩展头为64B：

```c
typedef struct st_graph_adj_data_page_header {
    graph_vid_t owner_vid;
    uint32 edge_type_id;
    uint16 schema_layout_id;
    uint8  direction;
    uint8  entry_format;
    uint16 shard_id;
    uint16 shard_count;
    uint16 entry_size;
    uint16 slot_count;
    uint32 live_count;
    uint32 flags;
    graph_adj_locator_t next_page;
    uint32 directory_version;
    uint16 zone_map_offset;
    uint16 zone_map_bytes;
    uint16 bloom_offset;
    uint16 bloom_bytes;
    uint16 invalid_projection_mask;
    uint16 free_begin;
    uint16 free_end;
    uint8  itl_count;
    uint8  itl_capacity;
    uint32 reserved_zero;
} graph_adj_data_page_header_t;      /* 64B */
```

`next_page` 形成同一个 `(owner_vid, direction, edge_type_id, shard_id)` 的页面链。每次进入页面必须校验这些字段、Schema Layout、Directory Version和Page Generation。

查询可以先用页头范围跳过整页，再用邻接项属性过滤，最后只读取命中的规范边和顶点。如果谓词对应的投影在本页存在失效项，则 Zone Map 只能处理有效项，失效项必须进入原值回读队列，不能被当作“不匹配”跳过。

Directory计数、Data Page的 `live_count`、Zone Map和Bloom Filter都是保守摘要，不决定MVCC可见性。摘要落后只能导致额外扫描，不能排除可能可见的数据；恢复时可以根据可见Entry重新计算。每个Vertex、Canonical Edge、Directory Object和External Adjacency Entry分别通过所在页面的Side Slot或Meta引用PCR ITL；Packed List整体使用一个Object Slot。创建一条边时这些记录可以使用不同页内ITL编号，但必须属于同一个数据库事务。

### 11.4 时间 Inline 类型

第一期不提供 `UINT32_DELTA`。只提供 `TIME32` 和 `TIME64`：

```sql
CAST(timestamp AS TIME32 WITH (
    BASE = TIMESTAMP '2020-01-01 00:00:00+00:00',
    UNIT = SECOND,
    OVERFLOW = REJECT
))
```

`TIME32` 保存相对显式 Base 的有符号 32-bit 时间差，`TIME64` 保存有符号 64-bit 时间差。`BASE` 和 `UNIT` 必填：

- `BASE` 必须是带明确时区或已经规范化时区的时间字面量；
- 禁止 `GRAPH_EPOCH`、`now()`、`current_time`、序列值、会话变量等动态来源；
- `UNIT` 是 `SECOND`、`MILLISECOND`、`MICROSECOND` 等明确单位；
- 编码公式是 `CAST(time_expression - base AS int32/int64)`；
- Base 写入 Schema Layout，创建后不可变，修改 Base 创建新 Layout 版本。

动态 Base 会使同一属性在不同写入时刻产生不同语义，并可能在更新后让已有 Inline 值失效，因此禁止使用。

### 11.5 通用数值 Inline 表达式

第一期不提供 `FIXED32_2`、`FIXED64_2`、`UINT8`、`UINT32_DELTA` 等业务语义 Codec。数值 Inline 统一使用扩展 CAST：

```sql
CAST(expression AS INLINE_INT32 WITH (
    BASE = 0,
    OVERFLOW = REJECT
))
```

支持的目标类型包括：

```text
INLINE_INT8 / INLINE_UINT8
INLINE_INT16 / INLINE_UINT16
INLINE_INT32 / INLINE_UINT32
INLINE_INT64 / INLINE_UINT64
INLINE_FP32 / INLINE_FP64
```

`expression` 可以引用当前顶点或边的多个属性并使用现有确定性表达式。编码公式是：

```text
encoded = CAST(expression - base AS inline_numeric_type)
```

`BASE` 可省略，默认是目标类型的零值；指定时必须是不可变数值常量。转换、舍入、NaN 和无穷处理遵循现有 SQL CAST 规则，但超出目标类型范围时使用本节定义的 `OVERFLOW` 策略。

允许任意表达式不代表优化器可以任意改写谓词。Inline 投影保存规范化表达式 AST。只有查询中出现相同规范化表达式，或者优化器能证明转换可逆且保持比较顺序时，才使用该 Inline 投影做过滤；否则回读规范原值。

禁止在 Inline 表达式中使用：

- `now()`、`current_time` 等时间相关函数；
- 随机函数、序列和会话状态；
- 子查询、聚合、窗口函数；
- 外部表或其他顶点、边的属性；
- 未标记为确定性的用户函数。

### 11.6 Overflow 语义与失效值

所有 `TIME32`、`TIME64` 和 `INLINE_*` 数值类型支持：

```text
OVERFLOW = REJECT | WARNING
```

默认是 `REJECT`。

`REJECT` 表示转换溢出或失败时拒绝整条 SQL，事务不得留下部分 Vertex、Edge 或 Adjacency 修改。

`WARNING` 表示规范原值写入成功，但对应 Inline 投影失效：

1. 将 Vertex/Edge Meta 或 OUT/IN 负载中对应投影的两位状态设置为 `10`；
2. 保留或写入 Property Row 中的规范原值；
3. 返回 SQL Warning；
4. 查询命中该项时批量回读规范原值，再执行原始谓词。

状态 `00` 表示逻辑 NULL，不需要回读；状态 `01` 才允许直接解码 Inline 字节。状态 `10` 不能被当作 NULL 或“不匹配”。

OUT/IN `inline_property_state_bitmap` 属于8B或16B方向负载预算，不额外扩大Entry。Page Header的 `invalid_projection_mask` 只记录每个属性是否存在至少一个状态 `10`，不保存精确计数；只要对应bit为1，使用该属性的Zone Map就不能排除这些项。

`WARNING` 保证SQL成功和结果正确，但可能降低查询性能。精确失效计数由后台统计扫描写入Catalog统计，不属于Adjacency Page持久化头；系统视图基于该统计暴露每个Inline投影的失效率，超过阈值时建议修改Base、目标类型或移除该投影。

### 11.7 `HASH32_VERIFY`

提供确定性函数：

```sql
HASH32_VERIFY(expression1, expression2, ...)
```

它将一个或多个表达式映射为 32-bit Hash，适合复合键等值预过滤。函数采用固定版本的持久化 Hash 算法和规范序列化：

- 参数顺序参与 Hash；
- 每个参数包含类型标签、NULL 标记和长度，禁止字符串简单拼接；
- 字符串遵循声明的字符集和 Collation 规范化；
- 时间统一到 Schema 声明的时区和精度；
- Hash 算法版本记录在 Schema Layout 中，跨节点和升级保持一致。

`HASH32_VERIFY` 永远不是规范值。Hash 命中后必须读取原始表达式涉及的属性并执行原谓词验证。它不能单独用于唯一约束、外键、排序或范围过滤。

示例：

```sql
HASH32_VERIFY(tenant_id, campaign_id) AS tenant_campaign_hash
```

查询优化器只有在能保留并执行原始 `(tenant_id, campaign_id)` 等值条件时才能使用该 Hash。用户可见 SQL 类型是 `INTEGER`，物理上按 32 bit 原始位模式保存。

## 12. Inline Edge 的取舍

Inline Edge 能让低度拓扑查询在读到 `05` 后直接获得 `12 neighbor_vid`，减少一次邻接页访问。它不能高效支持复杂边属性过滤；一旦复制较多属性，顶点槽会迅速变大。

100 亿顶点下，图级槽宽对空间影响显著：

| 图级布局 | 顶点槽 | 原始槽空间量级 |
|---|---:|---:|
| `COMPACT` | 48B | 约 480GB |
| `INLINE_8` | 56B | 约 560GB |
| `INLINE_16` | 64B | 约 640GB |

此数量未计入页头、ITL、空洞和副本。更大的槽还会降低顶点页缓存密度。

因此默认值为：

```text
Vertex Inline Size = 0B
Vertex Slot = 48B
Inline Edge Capacity = 0
低度邻接 = Packed Small Adjacency Page
普通邻接 = 带 OUT/IN Inline Properties 的邻接页
超级节点 = 分片邻接段
```

第一期包括 `BELONGS_TO`、`CREATED_BY` 在内的低度边也使用 Packed Small Adjacency Page。热点顶点提前进入普通邻接页，避免所有写操作争抢顶点页 X Owner。

未来若启用 Inline Edge，它没有独立 ITL，其变化记录在顶点槽版本和Undo中；同一顶点槽上的内嵌边更新会串行化。这也是第一期选择Packed Small Adjacency Page的原因之一。

## 13. 属性页

`23 Property Data Page` 复用 PCR Heap：

- `24` 保存 External ID、冷顶点属性、JSON、角色附加属性和向量 ID；
- `25` 保存冷边属性、JSON、审计信息、时态扩展值和向量 ID；
- 大对象继续使用 LOB locator；
- 向量本体存放在独立 Vector Segment，图记录只保存引用、量化摘要或聚类编号。

`24 Vertex Property Row` 和 `25 Edge Property Row` 在现有PCR Heap Row Header及列数据之间增加24B Graph身份头：

```c
typedef struct st_graph_property_row_header {
    uint32 graph_id;
    uint16 schema_layout_id;
    uint8  owner_kind;               /* VERTEX/EDGE */
    uint8  flags;
    uint64 owner_id;                 /* graph-relative VID或EID */
    uint16 owner_generation;
    uint16 payload_format;
    uint32 payload_bytes;
} graph_property_row_header_t;       /* 24B */
```

其后的列值、NULL Bitmap、变长列和LOB Locator继续使用PCR Heap现有Row格式与Catalog列映射，不重新定义一套属性编码。

`08` 和 `22` 使用现有8B `rowid_t`，无属性行时写统一的 `INVALID_ROWID`。RowID只保证物理定位，读取后必须校验：

```text
graph_id
owner_kind
owner_id
owner_generation
schema_layout_id是否可读
PCR MVCC可见性
```

任一身份不匹配都视为陈旧RowID或页面损坏，不能把该行返回给其他Vertex/Edge。业务表物理位置单独缓存，并通过业务主键校验。

## 14. 超级节点

超过配置度数后，目录切换到 `SUPERNODE_SHARDED`。分片策略包括：

- 按邻居 VID 哈希；
- 按邻居 VID 范围；
- 按时间范围；
- 按 Edge Subtype；
- 按 Edge Type 加邻居哈希。

默认按邻居 VID 哈希，使百万级邻接可并行扫描。时间窗口查询为主的边使用时间范围分片。

升级过程：

```text
创建分片目录
 → 新建多个邻接 Extent
 → 后台按策略重分布旧项
 → 新写入进入目标分片
 → 原子切换目录版本
 → 等待旧快照结束
 → 回收旧页
```

分片分散到多个页面和 DRC bucket。超级节点不使用单链溢出页。

## 15. 多 Label

`label_set_id` 指向复用的 Label 集合：

```text
42 → {User, Person, Employee, Manager}
43 → {User, Person, Employee}
```

Catalog：

```text
SYS_GRAPH_LABEL
SYS_GRAPH_LABEL_SET
SYS_GRAPH_LABEL_SET_MEMBER
```

Dictionary Cache 保存 `label_set_id → bitmap`。从已知 VID 判断多个 Label 时执行一次位图包含判断。

从 Label 扫描全图时使用按分区组织的倒排索引：

```text
(graph_id, label_id, virtual_partition_id) → compressed VID list
```

添加或删除 Tag 时创建或复用新的 Label Set，事务内更新顶点槽和倒排索引。VID、分区和顶点页位置保持不变。Tag 附加属性默认进入 `24`，不改变图级固定槽宽。

## 16. 复杂查询的遍历流程

以第 2.3 节的组合查询为例。

### 16.1 用户入口

```text
(graph_id, User namespace, external_id)
 → ID Map
 → user VID
 → 27 VID Locator
 → 02/04/05 User Vertex Slot
```

读取 `06.label_set_id`，确认包含 `User`。

### 16.2 第一跳 CLICKED

```text
以query_scn构造Vertex Slot CR版本
 → 校验VID generation、删除状态和Label
 → 读取adjacency_modes低4-bit及09 OUT Descriptor
 → 将Descriptor复制到执行器并释放Vertex Page latch
 → 按Mode解析Root
 → 定位CLICKED OUT
 → 页头Zone Map/Bloom跳过不符合时间范围的页
 → 16中按PCR可见性读取neighbor_vid、EID及OUT Inline属性
 → 得到seed VID和click EID
```

按Mode解析Root：

```text
NO_ADJACENCY
 → 返回空集

DIRECT
 → Root按GRAPH_SEG_ADJ_DATA解析
 → 校验页面owner_vid、OUT、edge_type_id=CLICKED

PACKED_DIRECTORY
 → Root按GRAPH_SEG_ADJ_DIRECTORY解析
 → 在Packed Directory查找CLICKED对象

EXTERNAL_DIRECTORY
 → 读取Directory Root
 → 按edge_type_id查找graph_adj_dir_entry_t

SUPERNODE_SHARDED
 → 读取Shard Directory
 → 根据邻居Hash、VID范围或时间谓词裁剪Shard

MIGRATING
 → 读取Migration对象
 → 按query_scn和switch_scn选择old/new root
 → 需要双读时按EID去重
```

进入每张Adjacency Page时必须校验：

```text
page_type和graph_page_identity
owner_vid == 当前顶点VID
direction == OUT
edge_type_id == CLICKED
schema_layout_id可读
directory_version和root generation匹配
```

扫描Entry时：

1. 通过Side Slot的 `itl_id` 和Undo判断 `query_scn` 可见性；
2. 使用OUT Inline状态和值执行 `timestamp/status/weight` 谓词；
3. 状态 `10`、属性未Inline、需要返回完整边或执行 `HASH32_VERIFY` 时，把EID加入回读队列；
4. 回读队列按 `EID.virtual_partition_id + logical_page_no` 排序并批量通过 `28` 读取 `19/25`；
5. 其余命中项直接输出完整 `neighbor_vid`，不读取Canonical Edge。

Vertex Page latch不能跨Adjacency Page访问持有，Adjacency Page latch也不能跨远程Frontier发送持有，避免跨页和跨节点Latch链。

### 16.3 后续 SIMILAR 跳

每一层执行：

```text
当前 VID Frontier
 → 去重
 → 按 Virtual Partition 分桶
 → 批量读取本地顶点与邻接
 → 用 SIMILAR 的 score 过滤
 → 生成下一层 Frontier
```

远程页数量少时批量拉取页面；某一分区 Frontier 很大时，把该桶发送到目标 Preferred Instance 继续执行。

IN遍历采用相同流程，但读取 `adjacency_modes` 高4-bit及 `10 in_adj`。IN Entry中：

```text
neighbor_vid = src_vid
EID.virtual_partition_id = src_vid.virtual_partition_id
```

因此目标顶点的IN邻接页位于目标分区，而返回的源顶点可以位于任意分区。执行器先扫描本地IN邻接集合，再按完整 `neighbor_vid.virtual_partition_id` 对源顶点Frontier分桶；不能假设源点与当前目标顶点同分区。

### 16.4 结果物化

最终候选执行：

```text
candidate VID → 26/05 → 07 热属性 → 必要时 08/24
```

中间顶点不读取完整属性。向量候选集先按 VID 建立过滤集合，再与图扩展结果相交。

### 16.5 写一条边

创建 `A -[CLICKED]-> B`：

```text
1. 从 A 的 Virtual Partition 分配 EID
2. 写 19，保存 20、21、22
3. 在 A 的分区写 16 的 OUT 项：neighbor_vid=B、eid=新EID
4. 在 B 的分区写 16 的 IN 项：neighbor_vid=A、eid=同一EID
5. 必要时创建或扩展 14、15
6. 写 Undo/Redo
7. 同一事务提交
```

任一页面修改失败，事务整体回滚。

第一次创建邻接或发生 `DIRECT→PACKED→EXTERNAL→SHARDED` 转换时，先完整构造新Root，再在Vertex Slot的PCR ITL保护下原子更新对应方向的Mode Nibble和Descriptor Root。旧查询通过CR Vertex Slot读取旧Root；只有旧快照结束后才回收旧对象。Mode和Root不能分两次无版本写入。

### 16.6 删除边和顶点

删除一条边在同一事务中标记：

```text
Canonical Edge Slot
源顶点OUT Adjacency Entry
目标顶点IN Adjacency Entry
必要的Directory计数和摘要失效标记
```

摘要允许延迟重算，但三个可见记录必须原子删除。只有早于删除SCN的查询全部结束后，Adjacency Object Slot和Canonical Edge Slot才允许递增generation并复用。

第一期普通 `DELETE vertex` 采用 `RESTRICT`：只要任一方向存在可见边就拒绝。`DETACH DELETE` 同步删除所有入边、出边和Canonical Edge后再删除Vertex；不提供可能留下悬空 `neighbor_vid` 的异步清理。Vertex Slot只有在全部邻接清理且旧快照结束后才能复用。

## 17. 用户接口

以下 DDL 是 oGRAC 的 OpenCypher 扩展。

### 17.1 创建图

```sql
CREATE GRAPH ad_graph
OPTIONS (
    DEFAULT_CONSISTENCY = 'STATEMENT',
    PAGE_SIZE = '8K',
    VIRTUAL_PARTITIONS = 65536,
    PLACEMENT_PARTITIONS = 1024,
    VERTEX_INLINE_SIZE = 16,
    EID_LAYOUT = 'VP16_PAGE24_SLOT8_GEN16'
);
```

`VIRTUAL_PARTITIONS` 是 VID 编码空间。物理 Segment 按实际出现的数据延迟创建。`PLACEMENT_PARTITIONS` 是当前迁移和调度粒度。

`VERTEX_INLINE_SIZE` 只能是 0、8 或 16，省略时默认为 0，对应 48B、56B、64B Vertex Slot。该参数在图创建后不可在线修改。`EID_LAYOUT` 第一期只允许 `VP16_PAGE24_SLOT8_GEN16`；显式记录布局版本是为了后续增加更大单分区地址空间，而不是在现有格式中借用 generation。

会话可覆盖一致性：

```sql
SET GRAPH READ CONSISTENCY = 'TRANSACTION';
SET GRAPH READ CONSISTENCY = 'EVENTUAL';
```

### 17.2 顶点类型

```sql
CREATE VERTEX TYPE User IN GRAPH ad_graph (
    external_id      VARCHAR(64) NOT NULL,
    tenant_id        BIGINT NOT NULL,
    status           SMALLINT,
    region_id        INTEGER,
    user_level       INTEGER,
    last_active_time TIMESTAMP,
    profile          JSON,
    embedding_id     BIGINT,

    EXTERNAL KEY (external_id)
)
ID NAMESPACE User
STORAGE (
    INLINE PROPERTIES (
        CAST(status AS INLINE_INT8 WITH (
            OVERFLOW = REJECT
        )) AS status,
        CAST(region_id AS INLINE_UINT16 WITH (
            OVERFLOW = REJECT
        )) AS region_id,
        CAST(user_level AS INLINE_INT32 WITH (
            OVERFLOW = REJECT
        )) AS user_level,
        CAST(last_active_time AS TIME64 WITH (
            BASE = TIMESTAMP '1970-01-01 00:00:00+00:00',
            UNIT = MICROSECOND,
            OVERFLOW = REJECT
        )) AS last_active_time
    )
)
PARTITION BY HASH (tenant_id, external_id);
```

DDL 根据图的 `VERTEX_INLINE_SIZE` 校验编码宽度，且 Inline 投影不超过 8 个；图级预算为0时不允许声明 Vertex Inline Properties。变长属性自动进入 `24`。Vertex/Edge 的两位状态使用Meta中已有的 `inline_property_state_bitmap`，不占用图级Inline预算。

第一期不提供按 Vertex Type 配置的 `VERTEX INLINE ADJACENCY`，因为它会破坏同一图Vertex Slot宽度固定的约束。低度边由Packed Small Adjacency Page承载。

### 17.3 Tag

```sql
CREATE VERTEX TAG Person FOR TYPE User IN GRAPH ad_graph;

CREATE VERTEX TAG Employee FOR TYPE User IN GRAPH ad_graph (
    employee_no VARCHAR(32),
    department_id BIGINT
);

CREATE VERTEX TAG Manager FOR TYPE User IN GRAPH ad_graph (
    management_level SMALLINT
);
```

Tag 属性默认进入属性行。

```cypher
MATCH (u:User {external_id: $id})
SET u:Employee:Manager;

MATCH (u:User {external_id: $id})
REMOVE u:Manager;
```

### 17.4 边类型

```sql
CREATE EDGE TYPE CLICKED IN GRAPH ad_graph
FROM User TO Advertisement (
    timestamp   TIMESTAMP NOT NULL,
    weight      FLOAT,
    status      SMALLINT,
    device_type SMALLINT,
    detail      JSON
)
STORAGE (
    EDGE_SLOT_PROFILE = AUTO,
    EDGE INLINE PROPERTIES (
        CAST(timestamp AS TIME64 WITH (
            BASE = TIMESTAMP '1970-01-01 00:00:00+00:00',
            UNIT = MICROSECOND,
            OVERFLOW = REJECT
        )) AS timestamp,
        CAST(weight AS INLINE_FP32 WITH (
            OVERFLOW = REJECT
        )) AS weight,
        CAST(status AS INLINE_INT16 WITH (
            OVERFLOW = REJECT
        )) AS status,
        CAST(device_type AS INLINE_INT16 WITH (
            OVERFLOW = REJECT
        )) AS device_type
    ),
    OUT INLINE PROPERTIES (
        CAST(timestamp AS TIME32 WITH (
            BASE = TIMESTAMP '2026-01-01 00:00:00+00:00',
            UNIT = SECOND,
            OVERFLOW = WARNING
        )) AS timestamp,
        CAST(weight AS INLINE_FP32 WITH (
            OVERFLOW = REJECT
        )) AS weight,
        CAST(status AS INLINE_INT8 WITH (
            OVERFLOW = REJECT
        )) AS status,
        CAST(device_type AS INLINE_INT8 WITH (
            OVERFLOW = REJECT
        )) AS device_type
    ),
    IN INLINE PROPERTIES (
        CAST(timestamp AS TIME32 WITH (
            BASE = TIMESTAMP '2026-01-01 00:00:00+00:00',
            UNIT = SECOND,
            OVERFLOW = WARNING
        )) AS timestamp,
        CAST(status AS INLINE_INT8 WITH (
            OVERFLOW = REJECT
        )) AS status
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

`detail` 未列入 Inline 子句，进入 `25`。例子中的EDGE INLINE编码值为16B，因此AUTO选择56B Canonical Edge Slot。显式配置支持 `COMPACT`、`INLINE_8`、`INLINE_16` 和 `INLINE_24`；编码宽度分别校验：EDGE INLINE不超过所选预算且投影不超过8个；OUT与IN各自不超过所选的8B或16B，其中2B `inline_property_state_bitmap` 计入方向预算。

例子中的 OUT/IN `timestamp` 使用 `WARNING`，因此完整时间戳必须保留在规范 Edge Slot 或 `25`。超出2026 Base的TIME32秒范围时，边写入仍成功，但邻接项对应状态设置为`10`，查询批量回读规范时间戳。默认策略仍是 `REJECT`。

### 17.5 超级节点策略

```sql
ALTER EDGE TYPE CLICKED IN GRAPH ad_graph
SET SUPER NODE POLICY (
    PROMOTE_AT_DEGREE = 100000,
    SHARD BY HASH(neighbor_vid),
    SHARDS = AUTO,
    MIN_SHARDS = 16,
    MAX_SHARDS = 256,
    TARGET_EDGES_PER_SHARD = 65536
);
```

时间范围策略写为 `SHARD BY RANGE(timestamp)`。

### 17.6 查询接口

第一阶段支持：

- 定点 `MATCH`；
- 指定 Edge Type 的一跳和 `*1..6` 有限变长路径；
- Label、顶点属性和边属性过滤；
- `WHERE`、`RETURN`、聚合、`ORDER BY`、`LIMIT`；
- 参数绑定；
- `id(vertex)` 返回 VID，`id(edge)` 返回 EID；
- `EXPLAIN` 显示 Expand、分区交换和预取计划。

后续增加：

- 向量起点过程；
- 边向量过滤；
- 更完整的 OpenCypher 写语句和最短路径。

写入接口包括按 External ID 创建顶点、按 Edge Type 创建边、更新属性和删除对象。所有结构修改都在数据库事务中执行。

## 18. Schema 变更

### 18.1 增加冷属性

```sql
ALTER VERTEX TYPE User ADD PROPERTY risk_level INTEGER;
```

只发布新的 Catalog 版本。旧记录缺少该属性时返回 NULL 或默认值，不重写顶点页。

### 18.2 改变内嵌布局

```sql
ALTER VERTEX TYPE User
SET INLINE PROPERTIES (status, region_id, user_level, risk_level);
```

流程：

```text
PREPARE   校验宽度并创建 schema_layout_id
PUBLISH   广播 Schema Epoch，失效旧执行计划
DUAL READ 按记录中的 schema_layout_id 解码
BACKFILL  按页转换，写 Undo/Redo，记录进度
SWITCH    新布局成为默认写入格式
GC        等待旧快照结束后回收
```

Vertex Inline表达式只能在图级预算内变更，因此Vertex Slot宽度不变，可以在原Segment中按Layout版本双读和回填。超过图级预算的变更必须拒绝，用户需要重建图；第一期不提供改变 `VERTEX_INLINE_SIZE` 的在线迁移。

Edge Inline表达式变更会创建新的 `schema_layout_id`。`EDGE_SLOT_PROFILE=AUTO` 对每个新Layout重新计算最小可用档位，因此新Layout可以比旧Layout更窄、同宽或更宽；显式Profile则继续使用用户指定的预算。新写入进入新Layout的Edge Page，旧边保留在旧Layout页面并继续按页头解码，不强制迁移且EID不变。更新旧边时，新Layout才有的值先写入Property Row；后台迁移Canonical Edge不属于第一期范围。

### 18.3 删除和类型变更

删除先逻辑隐藏，等待旧查询结束后物理清理。

安全扩展可以延迟转换，例如更窄整数扩展到更宽整数。不兼容转换要求显式转换表达式、后台校验和错误处理策略。

## 19. 分区变更

### 19.1 在线移动

```sql
ALTER GRAPH ad_graph
REBALANCE PLACEMENT PARTITIONS
TO INSTANCES (0, 1, 2, 3);
```

```text
选择一组 Virtual Partition
 → 标记 MIGRATING
 → 预热目标节点
 → 更新 route_epoch
 → 新请求进入目标节点
 → 等待旧请求结束
 → 标记 ONLINE
```

VID 不变。Preferred Instance 是性能提示，节点故障后由 DRC 保证可访问性，并更新路由。

### 19.2 调整 Placement 数量

```sql
ALTER GRAPH ad_graph
SET PLACEMENT PARTITIONS 2048
ONLINE;
```

该操作重组 Virtual Partition 到 Placement Partition 的映射，不修改 VID 和边引用。

### 19.3 修改分区键

```sql
ALTER GRAPH ad_graph
REPARTITION BY HASH(region_id, external_id)
MODE REBUILD;
```

分区键决定 Virtual Partition，进而影响 VID。已有数据默认通过重建新布局并切换读流量来完成。

第一期不提供Forwarding模式，因为修改VID还需要同步重写Canonical Edge端点、OUT/IN `neighbor_vid`、ID Map和源分区EID，单个转发指针不足以保证一致性。修改分区键只能执行完整Graph Rebuild并切换读流量。单个顶点不因短期热点改写 VID；热点通过副本、Preferred Instance 和邻接分片处理。

## 20. 向量能力预留

第一期数据结构保留以下接入点：

- 顶点或边属性中的 `vector_id`；
- 邻接项中的可选量化摘要；
- `VectorSeed` 输出 VID Frontier；
- `EdgeVectorFilter` 对一批 EID 批量过滤。

向量索引可使用 oGRAC 页面和分区路由，但其存储格式独立于邻接项。图执行器把向量结果当作普通 Frontier，继续复用分区批量扩展。

## 21. 模块划分与代码落点

| 模块 | 路径 | 内容 |
|---|---|---|
| 持久化结构 | `pkg/src/upgrade_check/knl_graph_persistent.h` | 页面、槽、目录和尺寸断言 |
| 图类型与接口 | `pkg/src/kernel/graph/` | ID、布局、定义校验、访问接口 |
| 页面类型 | `pkg/src/kernel/buffer/knl_page.h` | 新 `PAGE_TYPE_GRAPH_*` |
| Catalog | `pkg/src/kernel/catalog/` | 图、Label Set、ID Map、分区 |
| 集群 | `pkg/src/cluster/` | 图页 CR、批量预取、分区路由 |
| Redo/Undo | `pkg/src/kernel/persist/`、`pkg/src/kernel/xact/` | 图日志和回滚 |
| SQL 前端 | `pkg/src/ogsql/cypher/` | 独立 OpenCypher Parser、Binder、Lowering |
| 优化与执行 | `pkg/src/ogsql/optimizer/`、`executor/` | Expand、Path、分区交换 |
| 运维 | `pkg/src/ogbox/` | 页面转储和校验 |
| 测试 | `pkg/test/unit_test/ut/kernel/` | 布局和 ID 单元测试 |

现有 `cbo_join_graph_t` 是关系 Join 图，不复用为属性图存储或执行结构。

内核接口的第一阶段形态：

```c
status_t graph_validate_vertex_def(const graph_vertex_def_t *def);
status_t graph_validate_edge_def(const graph_edge_def_t *def);

status_t graph_make_vid(uint16 vpart, uint32 sequence, uint16 generation,
    graph_vid_t *vid);
status_t graph_vertex_locate(graph_handle_t *handle,
    graph_vid_t vid, uint8 access_mode, graph_mapping_guard_t *guard,
    graph_slot_location_t *location);
status_t graph_make_eid(const graph_id_layout_t *id_layout,
    uint16 vpart, uint32 logical_page_no, uint8 slot_id, uint16 generation,
    graph_eid_t *eid);
status_t graph_decode_eid(const graph_id_layout_t *id_layout,
    graph_eid_t eid, graph_eid_fields_t *fields);
status_t graph_edge_locate(graph_handle_t *handle,
    graph_eid_t eid, uint8 access_mode, graph_mapping_guard_t *guard,
    graph_slot_location_t *location);
status_t graph_page_resolve(graph_handle_t *handle, uint16 vpart,
    uint8 segment_kind, uint32 logical_page_no, uint8 access_mode,
    graph_mapping_guard_t *guard, graph_page_resolution_t *resolution);
void graph_mapping_guard_release(graph_mapping_guard_t *guard);
status_t graph_adj_resolve(graph_handle_t *handle, graph_vid_t owner_vid,
    uint8 direction, uint8 mode, graph_adj_locator_t locator,
    knl_scn_t query_scn, graph_adj_object_view_t *object);

status_t graph_get_vertex(graph_handle_t *handle, graph_vid_t vid,
    knl_scn_t query_scn, graph_vertex_view_t *vertex);
status_t graph_get_edge(graph_handle_t *handle, graph_eid_t eid,
    knl_scn_t query_scn, graph_edge_view_t *edge);
status_t graph_adj_open(graph_handle_t *handle, graph_vid_t owner_vid,
    uint8 direction, uint32 edge_type_id, knl_scn_t query_scn,
    graph_adj_cursor_t *cursor);
status_t graph_adj_fetch(graph_adj_cursor_t *cursor,
    graph_adj_entry_view_t *entry, bool32 *eof);
void graph_adj_close(graph_adj_cursor_t *cursor);
status_t graph_expand(graph_handle_t *handle,
    const graph_expand_req_t *request, graph_expand_result_t *result);
```

`access_mode` 固定为 `GRAPH_ACCESS_READ` 或 `GRAPH_ACCESS_WRITE`。所有定位API显式接收Graph Handle，因为相同的64-bit VID/EID值在不同Graph中可以合法存在。Reader成功定位后，在Mapping Guard保护下进入并Pin Buffer、校验Page Identity和 `mapping_version`，然后可以释放Guard。Writer必须继续持有Guard直到取得DRC X和Page Latch、再次校验Version、完成修改并生成Redo，之后才能调用 `graph_mapping_guard_release()`。`graph_adj_open()` 在内部复制Vertex Descriptor后释放Vertex Page latch，再解析Directory/Data Page；`graph_adj_close()` 必须释放当前Buffer、CR Page、Mapping Guard和远程预取资源。

ID构造函数返回 `status_t`，在写入输出参数前校验generation非0、保留值、Virtual Partition范围以及 `logical_page_no <= 0xFFFFFF`；禁止静默截断超出持久化位宽的输入。`graph_adj_resolve()` 使用 `query_scn` 构造Directory Object的CR版本并决定Migration old/new root。

未实现的存储修改和遍历返回明确错误，不能伪造成功。

## 22. 分期实施

### 第一期

- 持久化结构、ID、布局校验和页面容量计算；
- Catalog、ID Map、Label Set；
- 顶点、规范边、DIRECT/PACKED 邻接读写；
- EDGE/OUT/IN Inline Properties；
- 基本 OpenCypher 读查询和有限变长路径；
- 语句级一致读；
- 单写节点和多读节点；
- 超级节点哈希分片；
- 页面转储、恢复和回归测试。

### 第二期

- Placement 在线迁移；
- 多写路由和跨分区事务压测；
- 事务快照与最终一致策略完善；
- Schema 在线 Backfill；
- 向量起点和边过滤；
- 基于统计信息的移动计算选择。

### 后续

- 更完整的 OpenCypher；
- 分区键重建工具；
- 自动切图和社区分区；
- 更丰富的压缩邻接格式。

## 23. 错误处理与测试

校验失败包括：

- ID 字段越界或保留值冲突；
- 槽宽、属性宽度或页面容量超限；
- 不支持的编码、邻接模式或分片策略；
- generation 不匹配；
- Graph、Segment、Virtual Partition、Logical Page、Mapping Epoch、Page Generation、页面类型或布局版本不匹配；
- Extent不是PRESENT、已经RETIRED、页偏移越界或Physical Extent跨Datafile边界；
- 悬空 VID、EID 或 RowID；
- OUT/IN Mode与Descriptor Root类型不匹配；
- Adjacency owner、方向、Edge Type、Directory Version或Entry长度不匹配；
- 未指定重建模式就修改分区键；
- OUT/IN 属性无法从规范值编码或校验；
- TIME Base 不是明确时间字面量，或使用动态时间、会话状态；
- Inline 表达式包含非确定性函数、子查询或跨对象引用；
- `OVERFLOW=WARNING` 但不存在可回读的规范原值；
- `HASH32_VERIFY` 无法保留和执行原始验证谓词。

测试覆盖：

- VID/EID 边界值和往返；
- 24-bit Edge逻辑页号、8-bit Slot和16-bit generation边界；
- generation 不匹配；
- 所有持久化结构的大小和偏移；
- `graph_segment_head_t`、Extent Map和公共Page Identity的编码、恢复及反向校验；
- Extent主路径直接计算、Extent级Radix跨层查找和陈旧Mapping Epoch拒绝；
- Logical Page边界 `0/0xFFFFFF/0xFFFFFFFF` 的64-bit覆盖范围计算；
- Extent Entry的PRESENT/RETIRED、页偏移、Page Count、Extent Epoch和Allocation Generation校验；
- Extent Cache中 `mapping_version` 相同但 `allocation_generation` 不同的Entry必须失效；
- DRC Extent资源代次不匹配，以及Physical Extent回收重分配后的旧缓存和延迟响应必须被拒绝；
- 两个Extent并发修改同一已存在Extent Map Leaf时必须通过Page Latch串行化；
- 并发扩展Extent级Radix时，完整路径COW、唯一Root Reference CAS和失败重试；
- 复制期间并发写冻结、Resolve与Publish竞争及Mapping Guard读写互斥；
- Writer持有Mapping S Guard直到DRC X、Page Latch、页面修改和Redo生成全部完成；
- Extent翻译缓存失效确认和 `mapping_version` 校验；
- Extent级Radix增长时新路径节点先持久化、Root作为唯一发布点，以及Crash后旧树或新树原子可达；
- 新页Redo、映射发布Redo、Commit/Flush各Crash Point恢复到完整旧映射或完整新映射；
- 进入易失MAPPING_BUSY后Crash不会在持久化Entry中留下该状态；
- 两个整Extent搬迁并发时按Logical Extent串行化或CAS重试；
- DRC延迟响应、DCS传输、Buffer Pin和Mapping Guard未结束前禁止复用旧Physical Page；
- Partial Extent、Checked Page-ID Addition和Datafile边界；
- extent/ref/chunk Epoch及mapping_version回绕静默或退休；
- Standby ABR将Graph坏页修复到原Buffer与原 `page_id`，不更新Extent Map；
- Backup + Redo ABR只接受同一 `page_id`，并由 `RESTORE FLUSHPAGE` 写回原file/page offset；
- 数据页、Segment Head、Root/Child Extent Map页分别从Extent Entry、Catalog、Segment Head Root、Parent Reference构造ABR预期身份；
- Graph各Page Type在Checksum重算和任何写回前，按ABR请求上下文校验完整Page Identity、Page Type、PCN和Page ID；不匹配时拒绝写回；
- Graph各Page Type完成ABR原位写回后再次通过Page Identity与Checksum校验；
- 原位置永久不可写时由存储层坏块重映射或整Extent搬迁，第一期不存在单页映射覆盖；
- Phase 1拒绝启用 `GRAPH_MAP_FEATURE_PAGE_EXCEPTION`，不注册或分配对应Page Type及持久化结构；
- 8KB 页面容量；
- 固定槽页ITL区域耗尽时不移动Slot Area；
- sequence 到页和槽的定位；
- 图级0/8/16B Vertex Inline预算及48/56/64B Slot定位；
- OUT/IN Mode Nibble独立变化及两个12B Descriptor的结构一致性；
- Descriptor Root Locator无效值、Page Root、Object Root、generation不匹配和回绕退休；
- Adjacency/Directory Logical Page回收后永久退休，Object Slot仅在原页面身份内复用；
- 40/48/56/64B Edge Page的页内固定槽定位；
- Edge Page Header与Graph、Page Type、Edge Type、Schema Layout、Page Layout Version和Slot宽度交叉校验；
- Edge逻辑页清空后不允许更换逻辑身份或Layout；保持Slot内容、Page Generation和Layout时允许整Extent物理搬迁，并要求Extent Epoch原子递增；Slot generation回绕时退休；
- Inline Edge字段保持零值，低度边进入Packed Small Adjacency Page；
- 0/8/16B 方向属性预算；
- DIRECT、PACKED、EXTERNAL、SHARDED和MIGRATING Root解析；
- Packed对象晋升External Directory时旧快照读取旧Root、新快照读取新Root；
- Directory/Adjacency Side Slot整理不改变object_slot和generation；
- OUT Entry保存dst VID、IN Entry保存src VID，并按对方VID跨分区分桶；
- Directory计数、Zone Map和Bloom摘要陈旧时只能增加扫描、不能漏结果；
- Property Row反向校验Graph、Owner ID、generation和Schema Layout；
- 多 Label Set；
- TIME32/TIME64 的边界、时区、单位和不可变 Base；
- 所有 INLINE 数字目标类型的现有 CAST 语义、REJECT 原子回滚和两位状态转换；
- `00` NULL、`01` 有效 Inline、`10` 规范值回读三种状态的读写；
- WARNING 状态 `10` 不能被 Zone Map 错误排除，并能批量回读规范值；
- 任意确定性表达式的规范化匹配，以及非确定性表达式拒绝；
- `HASH32_VERIFY` 的多列、NULL、类型、Collation、参数顺序和碰撞后原值验证；
- Hash 算法版本在不同节点和恢复后的稳定性；
- 跨页创建边的原子性；
- 邻接属性过滤不读取未命中的 Edge Slot；
- 同一查询 SCN 下的多跳可见性；
- 恢复后 ID、目录和属性一致。

## 24. 已吸收的设计修正

| 早期想法 | 当前设计 | 原因 |
|---|---|---|
| VID 使用 16+40+8 | 16+32+16 | 为高频删除提供足够 generation |
| VID和EID使用相同的16+32+16解释 | VID为16+32+16；EID为16+24+8+16 | EID需要同时编码逻辑页与页内槽 |
| 固定 128B 槽和 4–8 条内嵌边作为默认 | 图级Vertex Inline为0/8/16B，默认0B；第一期不在Vertex Slot中保存Inline Edge | 保证同一图槽宽固定，并控制百亿顶点空间和顶点页争用 |
| 所有Canonical Edge固定64B | Page内固定，按Layout使用40/48/56/64B | 保持直接定位，同时提高无属性边和窄边的页面密度 |
| 邻接项使用 edge_token | 普通邻接项使用完整 EID | 降低重建和恢复复杂度 |
| 用户配置 Covering Properties | 用户配置 EDGE/OUT/IN INLINE | 直接表达三个物理副本 |
| 每个顶点独立 Directory Page | DIRECT、PACKED、EXTERNAL、SHARDED四种形态 | 低度顶点避免额外页面，超级节点可独立分片 |
| 图分区绑定一个 DRC 分区 | 两类分区正交 | DRC Master 可分散，本地性由 Page Owner 保证 |
| 单 Label | Label Set | 支持一人多角色 |
| ID Map 按 Label | ID Map 按 ID Namespace | Label 可变，业务身份不变 |
| 业务 ID 直接乘槽宽定位 | External ID 映射到 VID | 业务 ID 稀疏且需要稳定代次 |
| `UINT32_DELTA`、`FIXED32_2` 等专用 Codec | TIME32/TIME64 和通用扩展 CAST | 复用 SQL 转换语义，避免大量专用编码 |
| 动态时间 Base 或 Page Base | Schema 中的明确不可变时间字面量 | 更新和迁移不会改变 Inline 语义 |
| Hash 值直接参与等值判断 | `HASH32_VERIFY` 预过滤后校验原值 | 处理 32-bit Hash 碰撞 |
| 一个 Hot Property Bitmap 位 | 每个投影两位的 Inline Property State | 区分 NULL、有效值和 WARNING 回读 |
| 每个Logical Page维护独立映射Entry | Extent Map主路径 + 现有原位ABR；只允许整Extent重映射 | 普通访问只做Extent翻译，避免为极低概率坏页维护第二套映射、缓存和恢复协议 |

## 25. 总体确认清单

确认本文表示同意以下基线：

1. 原生图页面、复用 oGRAC 事务和 DRC。
2. VID使用16-bit虚拟分区、32-bit序号和16-bit代次；EID使用16-bit虚拟分区、24-bit逻辑页、8-bit页内槽和16-bit代次。
3. 图虚拟分区、Placement、Preferred Instance、DRC 分区和 Page Owner 互不替代。
4. 同一图的Vertex Slot固定；图创建时选择0B、8B或16B Inline预算，默认0B，对应48B、56B或64B Slot；第一期Inline Edge关闭，低度边使用Packed Small Adjacency Page。
5. 每条边有唯一Canonical Edge Slot；Edge Page内槽宽固定，不同Layout可使用40B、48B、56B或64B。
6. OUT/IN各使用相同的12B Descriptor但独立Mode；目录按Edge Type路由，并选择DIRECT、PACKED、EXTERNAL或SHARDED。
7. 用户通过 EDGE、OUT、IN INLINE PROPERTIES 控制三处属性副本。
8. 多 Label 使用 Label Set 和 ID Namespace。
9. 超级节点按邻居哈希等方式分片。
10. 第一期单写多读，多写通过分区亲和扩展。
11. 分区键变更默认重建；Placement 和 Preferred Instance 可在线调整。
12. 向量能力先预留 Frontier 接口，不进入第一期存储关键路径。
13. 第一期时间编码只提供显式 Base 的 TIME32/TIME64；数值使用通用 Inline CAST；Hash 使用 `HASH32_VERIFY` 并强制原值校验。
14. Vertex、Edge 和 OUT/IN Inline 投影使用两位状态编码，严格区分 NULL、有效值和规范原值回读。
15. Segment只使用Chunked Extent Map，超大或稀疏Segment可用Extent级Radix容器；坏页复用现有原位ABR，原位置永久不可写时使用存储层重映射或整Extent搬迁；所有Graph Page保存并反向校验公共身份。
16. 固定Vertex/Edge Slot Page预留不可增长的ITL区域；Adjacency/Directory使用带generation和ITL的8B Side Slot。
17. OUT Entry保存完整dst VID，IN Entry保存完整src VID；跨分区Frontier根据Entry中的VID分桶。
18. Property Row复用PCR Heap，但反向保存并校验Graph、Owner ID、generation和Schema Layout。

以下参数保留为上线前基准测试项，不改变上述结构：

- 实际 Placement Partition 数量；
- 业务分区键和边切割率；
- 各 Edge Type 的 OUT/IN 属性编码；
- 超级节点晋升阈值；
- 拉页与移动计算的切换阈值；
- Logical Extent页数；
- 最终一致模式允许的陈旧窗口。
