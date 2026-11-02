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
 → Segment 与 Extent Map
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

```text
63                    48 47                    32 32                 0
+-----------------------+------------------------+--------------------+
| virtual_partition_id  | generation             | local_sequence     |
| 16 bit                | 16 bit                 | 32 bit             |
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
 │           └─11 Optional Inline Edge Area            │
 │               ├─12 neighbor_vid ───────────────┐    │
 │               └─13 EID 或 edge_token ──────┐   │    │
 ├─14 Adjacency Directory                      │   │    │
 └─15 Adjacency Data Page                      │   │    │
     └─16 Adjacency Entry                      │   │    │
         ├─ neighbor_vid ──────────────────────┼───┤    │
         └─ full EID ──────────────────────────┤   │    │
17 Edge Page                                   │   │    │
 ├─18 Edge Page Header + PCR ITLs              │   │    │
 └─19 Canonical Edge Slot <────────────────────┘   │    │
     ├─20 edge_meta + src_vid + dst_vid            │    │
     ├─21 EDGE Inline Properties                   │    │
     └─22 edge_property_rowid ───────────────┐     │    │
23 Property Data Page <──────────────────────┴─────┘    │
 ├─24 Vertex Property Row <─────────────────────────────┘
 └─25 Edge Property Row
26 Target Vertex Page
 └─27 VID Locator <──── 12/16 neighbor_vid
28 EID Locator <─────── 13/16 EID
```

新增页面类型：

```text
PAGE_TYPE_GRAPH_SEG_HEAD
PAGE_TYPE_GRAPH_VERTEX
PAGE_TYPE_GRAPH_EDGE
PAGE_TYPE_GRAPH_ADJ_DIR
PAGE_TYPE_GRAPH_ADJ_DATA
```

属性页复用 `PAGE_TYPE_PCRH_DATA` 和 LOB。所有结构使用显式字节布局和编译期尺寸断言，不使用依赖编译器的位域。

## 8. 顶点存储

### 8.1 固定基础槽

`05 Vertex Slot` 的基础部分固定为 64B：

| 编号 | 字段 | 大小 | 用途 |
|---|---|---:|---|
| 06 | `vertex_meta` | 16B | 状态、代次、Label Set、Schema、ITL、邻接模式 |
| 07 | `inline_properties` | 16B | 定长热顶点属性 |
| 08 | `property_rowid` | 8B | 指向 `24 Vertex Property Row` |
| 09 | `out_adj` | 12B | 出邻接根位置和总出度 |
| 10 | `in_adj` | 12B | 入邻接根位置和总入度 |

```c
typedef struct st_graph_vertex_meta {
    uint16 flags;
    uint16 generation;
    uint32 label_set_id;
    uint16 schema_layout_id;
    uint16 inline_property_state_bitmap;
    uint8  itl_id;
    uint8  inline_out_count;
    uint8  inline_in_count;
    uint8  adjacency_mode;
} graph_vertex_meta_t; /* 16B */
```

`flags` 表示已分配、已删除、存在属性行、出邻接溢出、入邻接溢出、超级节点、迁移中和冻结。删除标记受 MVCC 保护，旧快照结束后才允许回收。

`label_set_id` 指向一组 Label。`schema_layout_id` 决定 16B 热属性如何解码。`inline_property_state_bitmap` 使用两位表示一个 Inline 投影的状态：

```text
00：NULL或属性不存在
01：Inline值有效
10：Inline值失效，必须回读规范原值
11：保留
```

16 bit 最多描述 8 个 Inline 投影。Schema Layout 为每个投影分配稳定的 ordinal 0–7；`bits[2*n, 2*n+1]` 对应 ordinal `n`。不在 Inline Layout 中的冷属性不占 ordinal。`itl_id` 指向 `03` 中的 PCR ITL。

`adjacency_mode` 取值为：

```text
NO_ADJACENCY
DIRECT
PACKED_DIRECTORY
EXTERNAL_DIRECTORY
SUPERNODE_SHARDED
MIGRATING
```

### 8.2 顶点页

`02 Vertex Page` 从低地址到高地址为：

```text
page_head_t
graph vertex page header
PCR ITL array
fixed vertex slots
page_tail_t
```

`04` 中的槽宽固定，因此：

```text
logical_page = local_sequence / slots_per_page
slot_id      = local_sequence % slots_per_page
page_id      = extent_map[logical_page]
```

`slots_per_page` 由页面大小、页头、ITL 和槽宽计算，并写入 Segment Header 与 Page Header。

### 8.3 可选 Inline Edge

`11` 默认不存在。启用后，出边数组和入边数组物理分离，因为 OUT 与 IN 的属性宽度可以不同：

```c
typedef struct st_graph_inline_edge_ref {
    graph_vid_t neighbor_vid;       /* 12 */
    graph_edge_token_t edge_token;  /* 13，仅混合 Edge Type 使用 */
    uint8 inline_properties[N];     /* 方向对应的 OUT 或 IN Inline Properties */
} graph_inline_edge_ref_t;
```

推荐容量是每个方向 0、1 或 2 条，最多评估到 4 条。8 条和 12 条不作为默认值，因为它们显著降低顶点页密度，并让每次增删边都修改顶点页。

普通邻接页保存完整 EID。`edge_token` 只在混合多个 Edge Type 的顶点内嵌数组中使用，其布局为：

```text
edge_local_sequence 32 bit
generation          16 bit
compact_type_code   16 bit
```

它省略分区，换取 Edge Type。出边分区取当前顶点分区，入边分区取 `neighbor_vid` 中的源顶点分区。

## 9. 规范边存储

每条逻辑边有唯一的 `19 Canonical Edge Slot`，固定 64B：

| 编号 | 内容 | 大小 | 用途 |
|---|---|---:|---|
| 20 | `edge_meta + src_vid + dst_vid` | 32B | 类型、代次、Schema、ITL、端点 |
| 21 | EDGE Inline Properties | 24B | 边属性的规范热值 |
| 22 | `property_rowid` | 8B | 指向 `25 Edge Property Row` |

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
} graph_edge_meta_t;
```

`17 Edge Page` 使用与顶点页相同的固定槽定位方式。EID 的 `local_sequence` 通过 `28 EID Locator` 得到逻辑页和槽号。

规范边是属性事实来源。出邻接和入邻接中的属性是为遍历准备的副本。

`flags` 表示已使用、已删除、存在属性行、存在出邻接副本、存在入邻接副本、时态边、关联向量和迁移状态。`property_mode` 表示属性位于规范槽、规范槽加属性行、仅属性行、LOB 或向量引用。

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

### 10.3 三种物理形态

目录是逻辑对象，不要求每顶点一张独立页面：

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
    rowid_t root_locator;
    uint32 flags;
    uint16 property_schema_version;
    uint8  shard_policy;
    uint8  sort_key_id;
} graph_adj_dir_entry_t;
```

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

OUT/IN 属性负载使用相同的两位状态编码，但状态位包含在方向预算内：

```c
typedef struct st_graph_adj_inline_payload {
    uint16 inline_property_state_bitmap;
    uint8 encoded_values[direction_inline_size - sizeof(uint16)];
} graph_adj_inline_payload_t;
```

因此 8B 负载实际提供 6B 编码值，16B 负载提供 14B 编码值，且每个方向最多定义 8 个 Inline 投影。Vertex/Edge Slot 的状态 Bitmap 位于 Meta 中，不占用各自 16B/24B Inline Value 区。

邻接页的 4B Side Directory：

```c
typedef struct st_graph_adj_slot {
    uint16 entry_offset;
    uint8  itl_id;
    uint8  flags;
} graph_adj_slot_t;
```

它保存项偏移、PCR ITL 和删除标志。邻接页只包含一个 `owner_vid + direction + edge_type`，因此项内不重复保存这些字段。

### 11.2 用户可见的三类属性

| 用户语法 | 编号 | 物理位置 | 用途 |
|---|---|---|---|
| `EDGE INLINE PROPERTIES` | 21 | 规范 Edge Slot | 无损值或可验证派生投影，用于按 EID 点查 |
| `OUT INLINE PROPERTIES` | 16 的出方向副本 | 出邻接项或出 Inline Edge | 正向遍历过滤 |
| `IN INLINE PROPERTIES` | 16 的入方向副本 | 入邻接项或入 Inline Edge | 反向遍历过滤 |
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

`15 Adjacency Data Page` 的页头维护：

- owner、方向、Edge Type；
- entry format 和 schema；
- shard、计数、空闲空间、下一页；
- 热属性最小值和最大值；
- 每个 Inline 投影是否完整的 Page Bitmap 或失效计数；
- 可选 Bloom Filter；
- PCR ITL。

查询可以先用页头范围跳过整页，再用邻接项属性过滤，最后只读取命中的规范边和顶点。如果谓词对应的投影在本页存在失效项，则 Zone Map 只能处理有效项，失效项必须进入原值回读队列，不能被当作“不匹配”跳过。

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

OUT/IN `inline_property_state_bitmap` 属于 8B 或 16B 方向负载预算，不额外扩大 Entry。Page Header 维护属性级失效计数；只要某属性存在状态 `10` 的项，使用该属性的 Zone Map 就不能排除这些项。

`WARNING` 保证 SQL 成功和结果正确，但可能降低查询性能。系统视图需要暴露每个 Inline 投影的失效率，超过阈值时建议修改 Base、目标类型或移除该投影。

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

100 亿顶点下，槽宽对空间影响显著：

| 基础布局 | 顶点槽 | 原始槽空间量级 |
|---|---:|---:|
| 无 Inline Edge | 64B | 约 640GB |
| 4 条 Compact | 128B | 约 1.28TB |
| 8 条 Compact | 192B | 约 1.92TB |

此数量未计入页头、ITL、空洞和副本。更大的槽还会降低顶点页缓存密度。

因此默认值为：

```text
Vertex Slot = 64B
Inline Edge Capacity = 0
低度邻接 = Packed Small Adjacency Page
普通邻接 = 带 OUT/IN Inline Properties 的邻接页
超级节点 = 分片邻接段
```

只对 `BELONGS_TO`、`CREATED_BY` 等低度、低更新、过滤字段很少的边按 Vertex Type 启用 Inline Edge。热点顶点即使未达到容量，也提前进入邻接页，避免所有写操作争抢顶点页 X Owner。

Inline Edge 没有独立 ITL，其变化记录在顶点槽版本和 Undo 中。同一顶点槽上的内嵌边更新会串行化。

## 13. 属性页

`23 Property Data Page` 复用 PCR Heap：

- `24` 保存 External ID、冷顶点属性、JSON、角色附加属性和向量 ID；
- `25` 保存冷边属性、JSON、审计信息、时态扩展值和向量 ID；
- 大对象继续使用 LOB locator；
- 向量本体存放在独立 Vector Segment，图记录只保存引用、量化摘要或聚类编号。

`08` 和 `22` 使用图引擎自己的 `rowid_t`。业务表物理位置单独缓存，并通过业务主键校验。

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

添加或删除 Tag 时创建或复用新的 Label Set，事务内更新顶点槽和倒排索引。VID、分区和顶点页位置保持不变。Tag 附加属性默认进入 `24`，不改变 64B 基础槽。

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
10 IN/09 OUT descriptor
 → 根据 adjacency_mode 进入 14 或直接进入 15
 → 定位 CLICKED OUT
 → 页头 min/max 跳过不符合时间范围的页
 → 16 中的 timestamp、status、weight 过滤
 → 得到 seed VID 和 click EID
```

只有需要返回完整边或访问 JSON 时才通过 `28` 读取 `19/25`。

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
3. 按 A 的模式写 11 或 16 的 OUT 项
4. 按 B 的模式写 11 或 16 的 IN 项
5. 必要时创建或扩展 14、15
6. 写 Undo/Redo
7. 同一事务提交
```

任一页面修改失败，事务整体回滚。

## 17. 用户接口

以下 DDL 是 oGRAC 的 OpenCypher 扩展。

### 17.1 创建图

```sql
CREATE GRAPH ad_graph
OPTIONS (
    DEFAULT_CONSISTENCY = 'STATEMENT',
    PAGE_SIZE = '8K',
    VIRTUAL_PARTITIONS = 65536,
    PLACEMENT_PARTITIONS = 1024
);
```

`VIRTUAL_PARTITIONS` 是 VID 编码空间。物理 Segment 按实际出现的数据延迟创建。`PLACEMENT_PARTITIONS` 是当前迁移和调度粒度。

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
    VERTEX_SLOT_SIZE = 64,
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

DDL 校验内嵌属性编码后不超过 16B，且 Inline 投影不超过 8 个；变长属性自动进入 `24`。Vertex/Edge 的两位状态使用 Meta 中已有的 `inline_property_state_bitmap`，不占用这 16B。

可选低度边内嵌：

```sql
VERTEX INLINE ADJACENCY (
    EDGE TYPES (BELONGS_TO, CREATED_BY),
    OUT CAPACITY 2,
    IN CAPACITY 1,
    OUT FORMAT 'INLINE_8',
    IN FORMAT 'INLINE_8'
);
```

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

`detail` 未列入 Inline 子句，进入 `25`。编码宽度分别校验：EDGE INLINE 不超过 24B且投影不超过8个；OUT 与 IN 各自不超过所选的8B或16B，其中2B `inline_property_state_bitmap` 计入方向预算。

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

槽宽变化必须创建新 Segment，再迁移 VID 对应的数据。

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

Forwarding 模式可暂时保留旧 VID 到新 VID 的指针，但会增加一次跳转，只作为兼容选项。单个顶点不因短期热点改写 VID；热点通过副本、Preferred Instance 和邻接分片处理。

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

graph_vid_t graph_make_vid(uint16 vpart, uint32 sequence, uint16 generation);
status_t graph_vertex_locate(const graph_segment_layout_t *layout,
    graph_vid_t vid, graph_slot_location_t *location);
status_t graph_edge_locate(const graph_segment_layout_t *layout,
    graph_eid_t eid, graph_slot_location_t *location);

status_t graph_get_vertex(graph_handle_t *handle, graph_vid_t vid,
    knl_scn_t query_scn, graph_vertex_view_t *vertex);
status_t graph_get_edge(graph_handle_t *handle, graph_eid_t eid,
    knl_scn_t query_scn, graph_edge_view_t *edge);
status_t graph_expand(graph_handle_t *handle,
    const graph_expand_req_t *request, graph_expand_result_t *result);
```

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
- 页面类型或布局版本不匹配；
- 悬空 VID、EID 或 RowID；
- 未指定重建模式就修改分区键；
- OUT/IN 属性无法从规范值编码或校验；
- TIME Base 不是明确时间字面量，或使用动态时间、会话状态；
- Inline 表达式包含非确定性函数、子查询或跨对象引用；
- `OVERFLOW=WARNING` 但不存在可回读的规范原值；
- `HASH32_VERIFY` 无法保留和执行原始验证谓词。

测试覆盖：

- VID/EID 边界值和往返；
- generation 不匹配；
- 所有持久化结构的大小和偏移；
- 8KB 页面容量；
- sequence 到页和槽的定位；
- 0/2/4 Inline Edge 配置；
- 0/8/16B 方向属性预算；
- DIRECT、PACKED、EXTERNAL 和超级节点策略；
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
| 固定 128B 槽和 4–8 条内嵌边作为默认 | 默认 64B，内嵌边按类型启用 | 控制百亿顶点空间和顶点页争用 |
| 邻接项使用 edge_token | 普通邻接项使用完整 EID | 降低重建和恢复复杂度 |
| 用户配置 Covering Properties | 用户配置 EDGE/OUT/IN INLINE | 直接表达三个物理副本 |
| 每个顶点独立 Directory Page | DIRECT、PACKED、EXTERNAL 三级 | 低度顶点避免额外页面 |
| 图分区绑定一个 DRC 分区 | 两类分区正交 | DRC Master 可分散，本地性由 Page Owner 保证 |
| 单 Label | Label Set | 支持一人多角色 |
| ID Map 按 Label | ID Map 按 ID Namespace | Label 可变，业务身份不变 |
| 业务 ID 直接乘槽宽定位 | External ID 映射到 VID | 业务 ID 稀疏且需要稳定代次 |
| `UINT32_DELTA`、`FIXED32_2` 等专用 Codec | TIME32/TIME64 和通用扩展 CAST | 复用 SQL 转换语义，避免大量专用编码 |
| 动态时间 Base 或 Page Base | Schema 中的明确不可变时间字面量 | 更新和迁移不会改变 Inline 语义 |
| Hash 值直接参与等值判断 | `HASH32_VERIFY` 预过滤后校验原值 | 处理 32-bit Hash 碰撞 |
| 一个 Hot Property Bitmap 位 | 每个投影两位的 Inline Property State | 区分 NULL、有效值和 WARNING 回读 |

## 25. 总体确认清单

确认本文表示同意以下基线：

1. 原生图页面、复用 oGRAC 事务和 DRC。
2. VID/EID 使用 16 bit 虚拟分区、32 bit 序号、16 bit 代次。
3. 图虚拟分区、Placement、Preferred Instance、DRC 分区和 Page Owner 互不替代。
4. 顶点基础槽 64B，Inline Edge 默认关闭并可按边类型启用。
5. 每条边有唯一 64B 规范槽。
6. 邻接目录按 Edge Type 路由，并根据数量选择 DIRECT、PACKED 或 EXTERNAL。
7. 用户通过 EDGE、OUT、IN INLINE PROPERTIES 控制三处属性副本。
8. 多 Label 使用 Label Set 和 ID Namespace。
9. 超级节点按邻居哈希等方式分片。
10. 第一期单写多读，多写通过分区亲和扩展。
11. 分区键变更默认重建；Placement 和 Preferred Instance 可在线调整。
12. 向量能力先预留 Frontier 接口，不进入第一期存储关键路径。
13. 第一期时间编码只提供显式 Base 的 TIME32/TIME64；数值使用通用 Inline CAST；Hash 使用 `HASH32_VERIFY` 并强制原值校验。
14. Vertex、Edge 和 OUT/IN Inline 投影使用两位状态编码，严格区分 NULL、有效值和规范原值回读。

以下参数保留为上线前基准测试项，不改变上述结构：

- 实际 Placement Partition 数量；
- 业务分区键和边切割率；
- 各 Edge Type 的 OUT/IN 属性编码；
- 超级节点晋升阈值；
- 拉页与移动计算的切换阈值；
- 最终一致模式允许的陈旧窗口。
