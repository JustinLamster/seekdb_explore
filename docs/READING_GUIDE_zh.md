# seekdb 源码阅读指南（面向初学者）

> 这份文档是写给"想读懂 seekdb 源码、但对数据库系统了解有限"的读者的。它不替代官方文档，而是把零散的背景知识、术语、阅读路径串成一条主线，让你知道**先读什么、为什么这样读、读到什么程度可以转下一节**。
>
> 文档分四个部分：
> 1. **背景知识**：你需要先理解的"数据库是什么"以及"seekdb 是什么"。
> 2. **架构总览**：seekdb 的大模块、它们如何协作。
> 3. **源码阅读路线图**：推荐的阅读顺序、每条主线的关键文件。
> 4. **专题深入**：内存 / 日志 / 多租户 / SQL 执行 / 存储 / 向量与全文 等模块的导读。
>
> 阅读这份文档时，**强烈建议**你打开一个能跳转代码的 IDE（VSCode + clangd，或 CLion），这样可以一边读一边追代码。

---

## 目录

- [Part 1 — 背景知识](#part-1--背景知识)
  - [1.1 什么是数据库系统](#11-什么是数据库系统)
  - [1.2 关系数据库（OLTP / OLAP）的核心概念](#12-关系数据库oltp--olap的核心概念)
  - [1.3 向量数据库与 ANN 检索](#13-向量数据库与-ann-检索)
  - [1.4 全文检索（Full-Text Search）](#14-全文检索full-text-search)
  - [1.5 OceanBase 与 seekdb 的关系](#15-oceanbase-与-seekdb-的关系)
  - [1.6 seekdb 的产品定位与"AI-Native"含义](#16-seekdb-的产品定位与ai-native含义)
- [Part 2 — 架构总览](#part-2--架构总览)
  - [2.1 仓库的物理布局](#21-仓库的物理布局)
  - [2.2 进程模型与线程模型](#22-进程模型与线程模型)
  - [2.3 多租户（Multi-Tenant）模型](#23-多租户multi-tenant模型)
  - [2.4 一条 SQL 的"生命旅程"](#24-一条-sql-的生命旅程)
  - [2.5 一次写入是如何持久化的（LSM 视角）](#25-一次写入是如何持久化的lsm-视角)
- [Part 3 — 源码阅读路线图](#part-3--源码阅读路线图)
  - [3.1 阅读前必须先掌握的"代码方言"](#31-阅读前必须先掌握的代码方言)
  - [3.2 推荐的阅读顺序](#32-推荐的阅读顺序)
  - [3.3 让你少走弯路的几个忠告](#33-让你少走弯路的几个忠告)
- [Part 4 — 模块专题导读](#part-4--模块专题导读)
  - [4.1 oblib：基础设施层](#41-oblib基础设施层)
  - [4.2 内存管理](#42-内存管理)
  - [4.3 日志系统](#43-日志系统)
  - [4.4 observer：进程入口与网络层](#44-observer进程入口与网络层)
  - [4.5 sql：查询编译与执行](#45-sql查询编译与执行)
  - [4.6 storage：存储引擎](#46-storage存储引擎)
  - [4.7 transaction & logservice：事务与日志复制](#47-transaction--logservice事务与日志复制)
  - [4.8 rootserver：集群元信息中心](#48-rootserver集群元信息中心)
  - [4.9 向量索引（vector_index / vsag / hnsw）](#49-向量索引vector_index--vsag--hnsw)
  - [4.10 全文索引（fts）](#410-全文索引fts)
- [Part 5 — 工具：编译、运行、调试、测试](#part-5--工具编译运行调试测试)
- [Part 6 — 术语表](#part-6--术语表)
- [Part 7 — 进一步学习资源](#part-7--进一步学习资源)

---

## Part 1 — 背景知识

### 1.1 什么是数据库系统

**一句话定义**：数据库系统（DBMS）是一类软件，它把"数据的存取"包装成一个高层接口（通常是 SQL），同时承担**持久化、并发、容错、查询优化、安全**这些麻烦事。

把"读懂源码"拆开来看，你其实需要理解 DBMS 至少要解决以下几类问题：

| 子问题 | 它在 seekdb 里大致对应哪里 |
|---|---|
| 客户端怎么连进来？协议长什么样？ | `src/observer/mysql/`、`src/observer/net/` |
| SQL 字符串怎么变成可执行的计划？ | `src/sql/parser` → `resolver` → `optimizer` → `code_generator` |
| 计划怎么真的跑起来？ | `src/sql/engine`、`src/sql/executor` |
| 数据放在磁盘上是什么格式？怎么读得快？ | `src/storage/blocksstable`、`src/storage/access` |
| 多个事务并发改同一条记录怎么办？ | `src/storage/tx`、`src/storage/concurrency_control` |
| 进程崩了重启能恢复吗？ | `src/storage/slog`、`src/logservice` |
| 集群里多副本怎么同步？ | `src/logservice`（PALF / clog）、`src/rootserver` |
| 多个用户/租户共享一台机器怎么隔离？ | `src/observer/omt`、`deps/oblib` 的 allocator 体系 |

你可以把上面这张表当作"读源码时的脑图"，每读懂一块，就在脑子里把对应的格子涂满。

### 1.2 关系数据库（OLTP / OLAP）的核心概念

读 seekdb 之前，你最少需要知道下面这些词的含义。**如果一时记不住没关系**，先有个印象，后面看到代码再回来查。

- **关系（Relation） / 表（Table）**：行与列组成的二维数据结构，列有类型，行是一条记录。
- **schema（模式）**：表的元信息（哪些列、什么类型、哪些索引、约束……）。在 seekdb 里 schema 的内存结构定义在 `src/share/schema/` 下。
- **SQL**：标准的查询语言。seekdb 兼容 MySQL 方言。
- **DDL（Data Definition Language）**：`CREATE TABLE` / `ALTER TABLE` 这类改 schema 的语句。
- **DML（Data Manipulation Language）**：`INSERT` / `UPDATE` / `DELETE` / `SELECT` 这类操作数据的语句。
- **事务（Transaction）**：一组操作要么全做、要么全不做（**ACID**：原子、一致、隔离、持久）。
- **隔离级别（Isolation Level）**：决定并发事务之间能不能看到对方未提交的数据。常见有 RC（Read Committed）、RR（Repeatable Read）、SI（Snapshot Isolation）等。
- **MVCC（Multi-Version Concurrency Control）**：通过保存多个数据版本，让读不阻塞写、写不阻塞读。OceanBase / seekdb 是 MVCC 实现。
- **WAL（Write-Ahead Log）**：先写日志、再改数据，这样即使崩溃也能用日志恢复。在 seekdb 里这条日志叫 **clog / PALF**。
- **LSM-Tree（Log-Structured Merge Tree）**：一种存储结构，把写先放在内存里（**MemTable**），写满后刷成磁盘上的不可变文件（**SSTable**），后台再做**Compaction（合并）**清理旧版本。OceanBase 的存储引擎就是 LSM 风格。
- **OLTP**（在线事务处理）：高并发、小事务，比如电商下单。
- **OLAP**（在线分析处理）：大数据量扫描分析，比如报表。
- **HTAP**：同时支持 OLTP 和 OLAP。OceanBase / seekdb 都属于这一类。
- **执行计划（Plan）**：优化器把 SQL 翻译出来的"算子树"，形如 `TableScan → Filter → Join → Project`。
- **算子（Operator）**：执行计划里的一个节点，每种算子是一个 C++ 类（在 seekdb 里都继承自 `ObOperator`，见 `src/sql/engine/ob_operator.h`）。

> ⚠️ 数据库术语很多，但**不需要一上来都搞懂**。你只要能在脑中区分"逻辑层（SQL/计划）"和"物理层（行/磁盘）"就够了，剩下的边读边查。

### 1.3 向量数据库与 ANN 检索

seekdb 把向量检索做成"内置能力"，所以你需要稍微了解一下背景。

- **嵌入向量（Embedding）**：把一段文字、一张图片用一个高维浮点数组（比如 384/768/1536 维）表示，使得"语义相近的内容向量也相近"。这些向量通常由神经网络模型（如 BERT、CLIP）产出。
- **向量相似度**：常用 L2 距离、内积、余弦相似度。
- **kNN（k-Nearest Neighbor）**：给一个查询向量，返回最相似的 k 个。
- **ANN（Approximate Nearest Neighbor）**：精确 kNN 在大数据下太慢，所以业界用近似算法换速度。
- **HNSW / IVF / PQ**：常见的 ANN 索引结构。
  - **HNSW**（Hierarchical Navigable Small World）：分层小世界图，目前最主流的图索引。
  - **IVF**（Inverted File）：先聚类，再在簇内搜索。
  - **PQ**（Product Quantization）：把高维向量压缩，加速距离计算。
- **混合检索（Hybrid Search）**：把向量检索和传统的全文检索/SQL 过滤合在一起。这是 seekdb 的核心卖点。

在 seekdb 中：
- 向量列由 `VECTOR(N)` 类型定义（你能在 `tools/deploy/mysqltest/` 的 case 里看到）。
- 向量索引底层接的是 [VSAG](https://github.com/antgroup/vsag)（蚂蚁集团开源的向量检索库），代码在 `src/storage/vector_index/` 与 `src/storage/retrieval/` 等处。
- DDL 语法形如：
  ```sql
  VECTOR INDEX idx_vec (embedding) WITH(DISTANCE=l2, TYPE=hnsw, LIB=vsag)
  ```

### 1.4 全文检索（Full-Text Search）

- **倒排索引（Inverted Index）**：把"文档→词"翻成"词→文档列表"，是全文检索的核心数据结构。
- **分词器（Tokenizer / Analyzer）**：把一段文本切成词，比如英文按空格切、中文用 ik / ngram 切。
- **打分（Scoring）**：典型如 BM25，根据词频和文档频率给每个匹配文档打分。

seekdb 把全文检索能力放在 `src/storage/fts/` 下，你能看到多个内置分词器（`ob_ngram_ft_parser`、`ob_ik_ft_parser`、`ob_beng_ft_parser` 等），DDL 形如：
```sql
FULLTEXT INDEX idx_fts(content) WITH PARSER ik
```

### 1.5 OceanBase 与 seekdb 的关系

这是理解整份代码的**第一关键**。

- **OceanBase** 是蚂蚁集团/阿里出品的分布式关系数据库，已经发展了十年以上、有数百万行 C++ 代码，特点是高可用、强一致、MySQL 兼容、HTAP。
- **seekdb 是 OceanBase 的"AI 化派生版"**。它复用了 OceanBase 内核里几乎所有东西（存储引擎、SQL 引擎、事务、日志服务、rootserver、多租户、内存分配器……），但**裁剪**和**新增**了一些能力，让它更像一个面向 AI 场景的搜索数据库。
- 因此你在源码里看到的：
  - 命名规范（`ob_*`、`Ob*`）
  - 错误码（`OB_SUCCESS`、`OB_FAIL`）
  - 目录结构（`observer/sql/storage/rootserver/share/...`）
  - 工具链（`build.sh`、`obd.sh`、`mysqltest`）

  ……都是从 OceanBase 继承来的。这意味着**OceanBase 的开源资料对你 100% 有用**：你在 [open.oceanbase.com](https://open.oceanbase.com) 上看到的内核分析文章、博客、PPT，几乎都可以直接套到 seekdb 上。

> 实战建议：当你在 seekdb 里看不懂一段代码时，可以直接搜对应的 OceanBase 内核博客（往往比看代码快十倍）。

seekdb 在 OceanBase 之上做的主要增量大致是：
- **嵌入式（Embedded）模式**：可以像 SQLite 一样把数据库嵌入到进程里跑，详见 `src/observer/embed/`。
- **向量检索 / 全文检索**深度集成（混合搜索、统一执行计划）。
- **AI Inside**：内置 embedding / reranking / LLM 推理（这部分功能多在 plugin 与上层 SDK，例如 [pyseekdb](https://github.com/oceanbase/pyseekdb)）。
- **裁剪了部分分布式能力**，主打"单机/嵌入式 + AI 搜索"。

### 1.6 seekdb 的产品定位与"AI-Native"含义

简单说："**MySQL 兼容 + 向量 + 全文 + JSON + GIS 全在一个引擎里，并且原生支持 AI 工作流**"。它的目标用户是想做 RAG / 语义搜索 / Agent 记忆系统的开发者，不希望把数据拆到向量库 + 关系库 + 搜索库三套系统里。

知道这点对读源码的意义在于：当你看到一个让你觉得"分布式数据库为什么要这么麻烦"的设计，请记住——seekdb 继承了一个分布式数据库的全部血统，但**它当前的主用例是单机/嵌入式**。这能解释很多"过度设计"的疑惑。

---

## Part 2 — 架构总览

### 2.1 仓库的物理布局

打开 seekdb 根目录，你会看到：

```
seekdb/
├── src/                # 内核源代码（你 80% 的时间会花在这里）
│   ├── observer/       # 服务器进程入口、MySQL 协议、RPC 派发、租户管理
│   ├── sql/            # SQL 编译执行栈：parser→resolver→optimizer→codegen→engine
│   ├── storage/        # 存储引擎：LSM、事务、向量索引、全文索引……
│   ├── rootserver/     # 集群元信息、DDL、负载均衡（单机模式下作用变小）
│   ├── logservice/     # PALF / clog 日志服务，支撑复制和恢复
│   ├── share/          # schema 缓存、配置、系统表、错误码 等共享代码
│   ├── pl/             # PL/SQL 存储过程引擎
│   ├── libtable/       # 表 API（KV / 客户端 SDK 用的）
│   ├── plugin/         # 可插拔扩展
│   ├── objit/          # 表达式 JIT
│   └── diagnose/       # 诊断/可观测性
│
├── deps/
│   ├── oblib/          # 基础设施库（容器、内存、IO、RPC、日志……）
│   ├── easy/           # 老一代 RPC 网络库
│   └── 3rd/            # 第三方依赖（由 build.sh --init 安装到 deps/3rd/usr/local/）
│
├── unittest/           # 内核单元测试，目录结构与 src/ 一一对应
├── deps/oblib/unittest # oblib 的单元测试
├── mittest/            # 多实例集成测试（simple_server / multi_replica / palf_cluster ...）
│
├── tools/
│   ├── deploy/         # obd.sh + 单机部署 yaml + mysqltest 测试用例
│   ├── ob_admin/       # 离线工具，用来 dump/检查数据文件
│   └── ob_error/       # 错误码查询工具
│
├── docs/
│   └── developer-guide/{en,zh}/   # 官方开发者文档（强烈推荐通读 en 目录）
│
├── build.sh            # 唯一推荐的编译入口
└── CMakeLists.txt
```

**记忆口诀**：`src/` 装内核，`deps/oblib/` 装基础库，`unittest/` 跟着 `src/` 走，`tools/deploy/` 是部署和回归测试的家，`docs/developer-guide/` 是官方说明书。

### 2.2 进程模型与线程模型

seekdb 主进程的可执行文件是 `build_<type>/src/observer/seekdb`（编译产物），它是一个**多线程单进程**程序。理解它的线程模型对读源码非常重要。

- 进程入口：`src/observer/main.cpp` → `src/observer/ob_server.cpp`（`ObServer::start()`）。
- **网络层**：基于 libeasy / pkt-nio 等，监听 MySQL 端口（默认 2881 / 部署时常见 10000），接收 MySQL 协议包后投递给请求队列。
- **请求分发**：`src/observer/ob_srv_xlator*` 把网络包翻译成内部 RPC / SQL 请求。
- **执行线程池（OMT）**：`src/observer/omt/` 是 **O**ceanbase **M**ulti **T**enancy 的缩写，每个租户拥有自己的线程池（worker pool）和资源配额。一条 SQL 实际是被某个租户的某个 worker 线程跑出来的。
- **后台线程**：合并/迁移/日志/统计等都各自有独立线程或定时器。

你在日志里看到 `[T1003][CompactionMergeT...]` 这种字段，前面的 `T1003` 是租户 ID，后面是线程名。

### 2.3 多租户（Multi-Tenant）模型

OceanBase 的核心设计之一是"在一个进程里跑多个数据库实例"，每个叫一个**租户（tenant）**：

- 租户 ID 是一个 `uint64_t`（特殊值：`OB_SYS_TENANT_ID = 1` 是系统租户，`500` 是 server 级 default 租户）。
- 每个租户有独立的：内存配额、worker 线程池、schema、临时表、缓存。
- 内存分配通过 `ob_malloc(size, ObMemAttr{tenant_id, ctx_id, label})` 记账，能分到具体租户头上。
- 嵌入式模式下"租户"概念依然存在，只是默认就一个用户租户。

为什么这点对读源码重要？——你会经常在函数签名里看到 `tenant_id` 参数，看到 `MTL(ObXXX*)`（**M**ulti-**T**enant **L**ocal）这种宏，看到 `ObTenantBase` / `ObTenantCtxAllocator`。请把它们理解为"在当前线程的租户上下文里取一个对象"。

`MTL` 宏的典型用法：
```cpp
ObVectorIndexService *svc = MTL(ObVectorIndexService*);  // 获取当前租户的向量索引服务
```

### 2.4 一条 SQL 的"生命旅程"

这是把所有模块串起来的最好方式。当你执行：
```sql
SELECT id, content
FROM articles
WHERE MATCH(content) AGAINST('hello' IN NATURAL LANGUAGE MODE)
ORDER BY l2_distance(embedding, '[0.1, 0.2, ...]') APPROXIMATE
LIMIT 10;
```

大致会发生以下事情（每一步括号里是对应的代码位置）：

1. **网络接入**：客户端用 MySQL 协议连接 → `src/observer/mysql/` 解析握手包 → 建立 session。
2. **请求派发**：MySQL `COM_QUERY` 包到达 → `ObMPQuery::process()`（`src/observer/mysql/obmp_query.cpp`）。
3. **进入 SQL 入口**：`ObSql::stmt_query()`（`src/sql/ob_sql.cpp`）—— 这是整个 SQL 子系统的总入口，**强烈建议先把这一个函数读懂**。
4. **解析（Parse）**：lex+yacc 写的 parser 把 SQL 字符串变成抽象语法树（`ParseNode` 树），代码在 `src/sql/parser/`。
5. **解析消歧（Resolve）**：`src/sql/resolver/` 把语法树绑定到 schema 上，校验列名、推断类型、生成"逻辑算子树（`ObDMLStmt`）"。
6. **重写（Rewrite）**：`src/sql/rewrite/` 做等价改写（谓词下推、子查询展开、视图合并……），优化器友好的形式。
7. **优化（Optimizer）**：`src/sql/optimizer/` 基于代价模型枚举执行计划，挑出最便宜的一个，输出"物理算子树（`ObLogicalOperator` → `ObOpSpec`）"。
8. **代码生成（Code Generator）**：`src/sql/code_generator/` 把物理算子翻译成可执行的 `ObOperator` 树和表达式（`ObExpr`）。
9. **执行（Engine）**：`ObResultSet::open() → ObOperator::get_next_row()`（`src/sql/engine/ob_operator.cpp`）按 Volcano 模型逐行/逐 batch 拉数据。
10. **存储访问（DAS / Storage）**：算子最底层是 `TableScan`、`IndexScan`，它通过 **DAS**（Data Access Service，`src/sql/das/`）调用存储引擎接口 `src/storage/access/` 读 MemTable + SSTable。
11. **混合检索（向量 + 全文）**：当 SQL 涉及 `VECTOR INDEX` 或 `FULLTEXT INDEX`，DAS 会走到向量索引扫描算子（`src/storage/vector_index/`、`src/storage/retrieval/`）或全文倒排扫描算子（`src/storage/fts/`），它们返回候选行 ID 后再回表。
12. **结果回写**：行被一路向上 `get_next_row` 到 `ObResultSet`，再被序列化成 MySQL 结果集协议包发回客户端。

**学习建议**：第一遍读源码时，**只跟一条 `SELECT * FROM t WHERE id = 1` 的极简路径**走一遍，把上面 12 步的每一步都看一眼即可。先求"宽度"，再求"深度"。

### 2.5 一次写入是如何持久化的（LSM 视角）

`INSERT INTO t VALUES (...)` 的简化路径：

1. SQL 层把 insert 算子翻译好后，调用存储 API 写入。
2. 写操作进入 **MemTable**（`src/storage/memtable/`）：内存中的有序表，使用 MVCC 多版本。
3. 同时把这条修改写到 **clog**（commit log，由 `src/logservice/` 提供的 PALF 复制日志），保证重启可恢复。
4. 事务提交时，事务模块（`src/storage/tx/`）协调可见性版本号（snapshot version）。
5. MemTable 写满后会被冻结，由 compaction 线程（`src/storage/compaction/`）刷成磁盘上的 **SSTable**（`src/storage/blocksstable/`，物理存储是定长的 macro/micro block）。
6. 后台 compaction 周期性合并多层 SSTable，回收过期版本。

上面每个名词都对应一个目录，你不需要一上来全部看完。等你对 SQL 路径熟悉以后，再回头读 storage 才会有收获。

---

## Part 3 — 源码阅读路线图

### 3.1 阅读前必须先掌握的"代码方言"

seekdb 是一个十年的 C++ 代码库，它有自己的"方言"。**不掌握这些约定，你看一行代码就要懵一次。** 这一节是你必须先记住的内容。

#### (a) 命名

- 文件名：`ob_xxx.cpp / ob_xxx.h`，全小写、下划线分隔。
- 类名：`ObXxx`，PascalCase。
- 函数名 & 普通变量：`lower_snake_case`。
- **类成员变量末尾必带下划线**：`int64_t schema_version_;`。
- 头文件 include guard：`OCEANBASE_<MODULE>_<FILE>_`。

#### (b) 错误码与"单入口单出口"

这是你最早需要适应的事。**所有函数返回 `int`，0（`OB_SUCCESS`）代表成功，其它代表错误**。错误码定义在 `deps/oblib/src/lib/ob_errno.h`（约几千个）。

**铁律**：禁止函数中途 `return`，禁止 `goto`/`exit`。每个函数只在最后 `return ret;`。为了遵守这个规矩，代码大量出现 `if / else if` 链：

```cpp
int ObSomething::do_work()
{
  int ret = OB_SUCCESS;
  if (OB_ISNULL(req_)) {
    ret = OB_INVALID_ARGUMENT;
    LOG_WARN("invalid", K(ret), KP(req_));
  } else if (OB_FAIL(step_one())) {
    LOG_WARN("step one failed", K(ret));
  } else if (FALSE_IT(flag_ = false)) {        // 用 FALSE_IT 包一行带副作用的语句
    // 这里永远走不到
  } else if (OB_FAIL(step_two())) {
    LOG_WARN("step two failed", K(ret));
  } else {
    // 成功路径
  }
  return ret;
}
```

常用错误处理宏：
| 宏 | 含义 |
|---|---|
| `OB_SUCC(ret)` | `ret == OB_SUCCESS` |
| `OB_FAIL(expr)` | 把 `expr` 的返回值赋给 `ret` 并判断不为 0 |
| `OB_ISNULL(p)` | `p == nullptr` |
| `OB_NOT_NULL(p)` | `p != nullptr` |
| `FALSE_IT(stmt)` | 把任意一句副作用语句包成一个永远求值为 false 的表达式，方便嵌入到 if-else 链里减少嵌套 |
| `OB_UNLIKELY(x)` / `OB_LIKELY(x)` | 编译器分支预测提示 |

#### (c) 不准用 STL，要用 ObXxx 容器

为了多租户内存记账，**禁止使用 `std::vector / std::map / std::unordered_map / std::string` 等 STL 容器**。对应的替代品（在 `deps/oblib/src/lib/container/` 与 `deps/oblib/src/lib/hash/`）：

| STL | seekdb 对应 |
|---|---|
| `std::string` | `ObString`（注意：**不管理内存、不以 `\0` 结尾**） |
| `std::vector` | `ObArray` / `ObSEArray`（带本地小数组的版本）/ `ObFixedArray` |
| `std::list` | `ObList` / `ObDList` |
| `std::unordered_map` | `ObHashMap`（需要 `create()` 后才能用）/ `ObLinkHashMap` |
| `std::unordered_set` | `ObHashSet` |
| `std::map`（红黑树） | `ObRbTree` |
| `std::queue` | `ObFixedQueue` / `ObLinkQueue` / `ObLightyQueue` |

容器接口风格：函数都返回 `int`，需要你检查 `ret`。文档详见 `docs/developer-guide/en/container.md`。

#### (d) 内存分配方式

- **不要直接 `new` / `malloc`**。统一走 `ob_malloc` / `OB_NEW` / `ObArenaAllocator` 等接口（详见后文专题 4.2）。
- 大量代码看起来"申请了内存却没释放"——别紧张，那通常是用了 `ObArenaAllocator`，它在请求结束时一把回收。

#### (e) 现代 C++ 用得很克制

- 不鼓励 `auto`、智能指针、移动语义、range-for、lambda（**内核代码里**）。
- 鼓励 `override`、`final`、`constexpr`。
- 模板大量使用，但避免过度复杂的元编程。

#### (f) 日志

- 用 `LOG_INFO / LOG_WARN / LOG_ERROR / LOG_DEBUG / LOG_TRACE`。
- 不用 printf 风格的格式串，而是 `K(var)` 把变量以 `key=value` 形式塞进去。

```cpp
LOG_WARN("failed to do x", K(ret), K(tenant_id), KP(ptr), KPC(session));
```

| 宏 | 输出 |
|---|---|
| `K(x)` | `"x", x` |
| `K_(x)` | `"x", x_`（成员变量） |
| `KP(p)` | 指针的十六进制 |
| `KPC(p)` | 调用 `*p` 的 `to_string()`；为 NULL 则输出 "NULL" |
| `KR(ret)` | 同时输出错误码及其名称（仅非 lib 代码可用） |

文档：`docs/developer-guide/en/logging.md`。

#### (g) `TO_STRING_KV` 宏

任何想被日志打印的对象，都要在类里加：
```cpp
TO_STRING_KV(K_(field1), K_(field2), KP_(ptr));
```
它会自动生成把对象序列化成 `field1=..., field2=..., ptr=0x...` 的方法。

---

掌握了这 7 条，你就能"看懂"绝大部分 seekdb 代码的形态了。

### 3.2 推荐的阅读顺序

下面的顺序假设你**对数据库不熟**，想从零看到能做小修改的程度。每一步给一个**最小的产出目标**——达成它就可以进下一步，不要被庞大的代码量吓退。

#### Step 0 — 跑起来 + 编译过

> 目标：成功把 seekdb 编出来，并用 obd.sh 起一个本地实例，用 mysql client 连进去执行 `select 1;`。

参考：`docs/developer-guide/en/build-and-run.md` 与本仓库根目录 `CLAUDE.md`。

#### Step 1 — 阅读官方开发者文档

> 目标：通读 `docs/developer-guide/en/` 下面除 toolchain 之外的所有文档，特别是：
> - `coding-convention.md` / `coding-standard.md`（约定）
> - `container.md`（容器）
> - `memory.md`（内存）
> - `logging.md`（日志）
> - `unittest.md`（测试）
> - `debug.md`（调试）

读这些文档比直接读代码事半功倍。它们已经是被高度浓缩过的知识。

#### Step 2 — 找一段最小的 SQL 路径，从入口开始追

> 目标：能描述出 `SELECT 1;` 从网络包到结果包的完整调用链。

具体怎么做：
1. 找到 `src/observer/main.cpp`，看一下进程怎么启动（不需要全懂）。
2. 找到 `src/observer/mysql/obmp_query.cpp` 里的 `ObMPQuery::process()`，这是 MySQL `COM_QUERY` 包的入口。
3. 跟着进入 `ObSql::stmt_query()`（`src/sql/ob_sql.cpp`）。
4. 用 grep 跟一下 parse → resolve → optimize → codegen → execute 五步，每步看 2~3 个文件即可。
5. 跟到 `ObResultSet::open()` 与 `ObOperator::get_next_row()`，理解 Volcano 拉模型。

**强烈推荐**：使用 gdb attach 到 seekdb 进程，在上面这些函数下断点，自己跑一条简单 SQL，看真实的栈是怎么走的。十次断点比十小时静态阅读更有效。`docs/developer-guide/en/debug.md` 里教了怎么用。

#### Step 3 — 跟一条写入路径

> 目标：理解一次 `INSERT` 是怎么落到 MemTable 和 clog 的。

读 `src/sql/engine/dml/`（DML 算子）→ `src/storage/access/`（存储访问层接口）→ `src/storage/memtable/`（内存表）→ `src/storage/tx/`（事务上下文）。一开始不需要看 SSTable 部分。

#### Step 4 — 读一遍 oblib 关键基础设施

> 目标：以后看到 `ObSEArray`、`ObHashMap`、`ObArenaAllocator`、`ob_malloc` 不再要查文档。

重点文件：
- `deps/oblib/src/lib/container/ob_se_array.h`
- `deps/oblib/src/lib/hash/ob_hashmap.h`
- `deps/oblib/src/lib/allocator/page_arena.h`
- `deps/oblib/src/lib/allocator/ob_malloc.h`
- `deps/oblib/src/lib/oblog/ob_log.h`

#### Step 5 — 选一个你最感兴趣的子系统深入

到这一步你已经具备"自驱"读源码的能力了。挑一个让你兴奋的方向继续：

- 想搞懂 SQL 优化器 → `src/sql/optimizer/`、`src/sql/rewrite/`
- 想搞懂 LSM 存储 → `src/storage/blocksstable/`、`src/storage/compaction/`
- 想搞懂事务 → `src/storage/tx/`、`src/storage/concurrency_control/`
- 想搞懂日志复制 → `src/logservice/`
- 想搞懂向量检索 → `src/storage/vector_index/`、`src/storage/retrieval/`、第三方 vsag
- 想搞懂全文检索 → `src/storage/fts/`
- 想搞懂多租户 → `src/observer/omt/`、`deps/oblib/src/lib/alloc/`

#### Step 6 — 写一个单元测试 / 改一个小 bug

> 目标：在 `unittest/` 里加一个 `test_xxx.cpp`，跑通；或在 `tools/deploy/mysqltest/test_suite/` 里加一个 case。

到这一步，你就真正"读懂"这份代码了。

### 3.3 让你少走弯路的几个忠告

1. **别试图一遍读懂所有代码**。OceanBase 内核是百万行级别的，按"先宽后深"的顺序、配合 gdb 单步是唯一可行的方法。
2. **优先看 `ob_xxx.h`，再看 `ob_xxx.cpp`**。头文件里有类的整体面貌，cpp 是细节。先理解类的责任再读实现。
3. **对长函数，先读 if-else 链的"骨架"，跳过日志和小细节**。seekdb 的函数往往很长，但骨架其实简单。
4. **看不懂时直接搜 OceanBase 的中文博客**。绝大多数模块在 [open.oceanbase.com](https://open.oceanbase.com/blog) 都有解读文章。
5. **不要被错误处理的样板代码淹没**。脑中"过滤掉" `if (OB_FAIL(...)) { LOG_WARN(...); }` 这类样板，只关注 happy path。
6. **常备 `ob_errno.h`**：你查每一个错误码的含义都需要它。
7. **遇到不熟的宏先 grep 它的定义**。`MTL`、`FALSE_IT`、`OB_INNER_TABLE_DEFAULT_VALUE` 之类的宏到处都是，搞清楚一次，受益一辈子。
8. **不要写 STL**，否则你提交代码时会被 review 拒掉。

---

## Part 4 — 模块专题导读

> 这一节按"先底层、后上层"的顺序介绍每个核心模块。每个模块给你 (1) 它做什么 (2) 入口文件 (3) 推荐先读的几个文件。

### 4.1 oblib：基础设施层

**位置**：`deps/oblib/src/lib/`

**做什么**：提供整个 seekdb 都依赖的基础设施。可以类比为 STL + Boost + glog + 内存分配器 + RPC。下面这些目录是最常用的：

| 子目录 | 内容 |
|---|---|
| `lib/container/` | `ObArray`、`ObSEArray`、`ObFixedArray`、`ObList`、`ObDList`、`ObBitmap` 等 |
| `lib/hash/` | `ObHashMap`、`ObHashSet`、`ObLinkHashMap`、`ObLinearHashMap` |
| `lib/alloc/`、`lib/allocator/` | `ob_malloc`、`ObTenantCtxAllocator`、`ObArenaAllocator`、`ObConcurrentFIFOAllocator` |
| `lib/string/` | `ObString` 与字符串工具 |
| `lib/oblog/` | 日志系统 (`OB_LOGGER`) |
| `lib/thread/`、`lib/coro/` | 线程、协程基础 |
| `lib/lock/` | 各种锁（latch、spinlock、futex 等） |
| `lib/queue/` | 各种队列 |
| `lib/charset/` | 字符集与 collation |
| `lib/compress/` | 压缩算法封装（zstd、lz4 等） |
| `lib/encrypt/`、`lib/ssl/` | 加密、TLS |
| `lib/json/`、`lib/json_type/` | JSON 解析与类型 |
| `lib/geo/` | GIS 几何 |
| `lib/number/` | 高精度数值（DECIMAL） |
| `lib/timezone/` | 时区 |
| `lib/wide_integer/` | 大整数 |
| `lib/roaringbitmap/` | RoaringBitmap |
| `lib/stat/`、`lib/wait_event/`、`lib/trace/` | 性能统计、等待事件、链路追踪 |

**进入 oblib 时的小心智模型**：你可以把它想成一座地基，地基上盖了 src/ 这栋大楼。你不需要把地基的每一块砖都看完——只在用到的时候去查。

### 4.2 内存管理

**位置**：`deps/oblib/src/lib/alloc/`、`deps/oblib/src/lib/allocator/`

**核心 API**：
```cpp
// libc 风格
void *ob_malloc(int64_t nbyte, const ObMemAttr &attr = default_memattr);
void  ob_free(void *ptr);
void *ob_realloc(void *ptr, int64_t nbyte, const ObMemAttr &attr);

// C++ 风格（会调用构造/析构）
OB_NEW(T, label, ...);
OB_DELETE(T, label, ptr);

// 池子风格
OB_NEWx(T, pool, ...);
OB_DELETEx(T, pool, ptr);
```

每次申请都要带上 `ObMemAttr`：
```cpp
struct ObMemAttr {
  uint64_t    tenant_id_;   // 算到哪个租户头上
  ObLabel     label_;       // 字符串标签，用于分类统计
  uint64_t    ctx_id_;      // 见 alloc_struct.h，每个 (tenant,ctx) 一个 allocator
  ObAllocPrio prio_;        // Normal / High，决定能否吃 reserved memory
};
```

**几种常见 allocator**：

| Allocator | 使用场景 |
|---|---|
| **ob_malloc / ob_free** | 通用、按租户记账。最常用。 |
| **ObArenaAllocator** | "只申请、批量释放"。SQL 一次请求里申请的小内存通常都用 arena，请求结束 reset 一把回收。**这就是为什么你常常看不到 free 在哪里。** |
| **ObConcurrentFIFOAllocator** | 多线程 FIFO 分配。 |
| **ObFIFOAllocator** | 单线程 FIFO。 |
| **PageArena** | arena 的底层实现，按 page 拿。 |

**调试与诊断**：seekdb 周期性会把所有租户 / ctx 的内存使用量打到日志里（搜 `[MEMORY]`），出问题时这是排查的第一手资料。

详见 `docs/developer-guide/en/memory.md`。

### 4.3 日志系统

**位置**：`deps/oblib/src/lib/oblog/`

**关键概念**：
- 日志默认输出到 `<install_dir>/log/seekdb.log`。
- 日志级别：`DEBUG / TRACE / INFO / WDIAG / EDIAG / WARN(DBA) / ERROR(DBA)`。
- **WDIAG / EDIAG**：开发自查日志，**等同于** `LOG_WARN` / `LOG_ERROR`，但和给 DBA 看的 `LOG_DBA_WARN` / `LOG_DBA_ERROR` 严格区分。读源码时注意。
- 每条日志带：时间 / 级别 / 模块 / 函数 / 文件:行 / 线程 ID 与名 / 租户 / **TraceID** / 上一次日志耗时。
- TraceID 是排查问题的金钥匙：拿到 TraceID 后用 `grep` 一查就能拉出一整条请求的所有日志。
- 用 `K(...)` 输出变量。详见上面 [3.1 节](#31-阅读前必须先掌握的代码方言)。

**实操技巧**：
```sql
-- 临时调高某模块日志级别（只对当前请求生效）
SELECT /*+ log_level("SQL.OPT:DEBUG") */ * FROM t WHERE id = 1;

-- 全局调日志级别
ALTER SYSTEM SET syslog_level = 'DEBUG';
```

详见 `docs/developer-guide/en/logging.md`。

### 4.4 observer：进程入口与网络层

**位置**：`src/observer/`

**关键文件**：
- `main.cpp` —— 进程入口。
- `ob_server.cpp` / `ob_server.h` —— `ObServer` 单例，负责生命周期、初始化各子系统。
- `mysql/` —— MySQL 协议层。`obmp_query.cpp / obmp_stmt_*.cpp` 是各种 MySQL 命令的 handler。
- `net/` —— 网络层封装。
- `ob_srv_xlator.cpp / ob_srv_xlator_*.cpp` —— 把进来的 RPC 包路由到对应的处理函数。
- `omt/` —— 多租户线程池实现（`ObMultiTenant`、`ObTenant`）。
- `embed/` —— 嵌入式模式入口，让 seekdb 能像 SQLite 一样被链接进进程。
- `table/`、`table_load/` —— 表 API（KV / 直接导入路径）。
- `virtual_table/` —— 系统视图实现。

**第一次读建议**：
1. `main.cpp`（很短）
2. `ObServer::start()` 的骨架（只看顶层调用，不进细节）
3. `obmp_query.cpp::ObMPQuery::process()`（这是你 SQL 之旅的起点）

### 4.5 sql：查询编译与执行

**位置**：`src/sql/`

**子目录速览**：

| 子目录 | 角色 |
|---|---|
| `parser/` | lex+yacc 把 SQL 字符串变成 `ParseNode` 树 |
| `resolver/` | 把语法树绑定 schema，输出 `ObDMLStmt` 等"逻辑表示" |
| `rewrite/` | 等价改写：谓词下推、子查询消除、视图合并、外连接消除…… |
| `optimizer/` | 基于代价的优化器，输出 `ObLogicalOperator` 树 |
| `code_generator/` | 把逻辑算子翻译成物理算子规格 (`ObOpSpec`) 与表达式 (`ObExpr`) |
| `engine/` | **物理执行**：所有 `ObOperator` 子类都在这里。再细分 `aggregate / join / sort / window_function / px / dml / table / set / subquery / expr ...` |
| `executor/` | 一条 SQL 的总执行调度（`ObExecutor`） |
| `das/` | **D**ata **A**ccess **S**ervice：算子向存储层发请求的统一中间层 |
| `dtl/` | **D**ata **T**ransfer **L**ayer：分布式执行时算子之间的数据通道 |
| `plan_cache/` | 计划缓存，把同一个 SQL 文本的优化结果缓存复用 |
| `session/` | session 上下文 |
| `monitor/` | SQL 监控与统计视图 |
| `printer/` | 反解 SQL（把 stmt 还原成字符串） |
| `privilege_check/` | 权限检查 |
| `udr/` | 用户自定义规则 |

**核心入口**：
- `ObSql::stmt_query()`（`ob_sql.cpp`）—— 你的第一个断点。
- `ObResultSet`（`ob_result_set.cpp`）—— 一条 SQL 一个 `ObResultSet`，封装计划与执行上下文。
- `ObOperator::get_next_row()`（`engine/ob_operator.cpp`）—— 算子拉模型的接口。
- `ObExecContext`（`engine/ob_exec_context.cpp`）—— 执行期上下文，挂着所有 per-query 资源（算子、表达式 ctx、错误信息……）。

**学习顺序建议**：
1. 把 `ObOperator` 类层次先看明白：每个算子提供 `inner_open / inner_get_next_row / inner_close`，由基类编排生命周期。
2. 找最简单的算子读：`engine/basic/` 下面的 `ObValuesOp`、`ObExprValuesOp`。
3. 然后读 `engine/table/`（表扫描算子），理解如何调 DAS 拿数据。
4. 之后再读 `engine/join/`（hash join、merge join）感受经典算法在工业代码里的样子。
5. 最后再啃 `optimizer/`——这里的代码最难，但你已经理解执行模型后会容易很多。

### 4.6 storage：存储引擎

**位置**：`src/storage/`

存储是整份代码最庞大也最深的子系统。建议**最后再深入**。

**子目录速览**：

| 子目录 | 角色 |
|---|---|
| `memtable/` | 内存中的有序写入表（MVCC） |
| `blocksstable/` | 磁盘上的不可变 SSTable（macro/micro block 格式） |
| `blockstore/` | 块存储抽象 |
| `compaction/` | 后台合并（mini / minor / major compaction） |
| `access/` | 存储访问层接口（行迭代器、谓词下推、列裁剪等） |
| `tx/` | 事务实现：事务管理器、二阶段提交、xid |
| `tx_storage/` | 事务相关的存储辅助 |
| `tx_table/` | 事务状态表 |
| `concurrency_control/` | 并发控制（行锁、版本可见性） |
| `tablet/` | tablet（分片）抽象 |
| `tablelock/` | 表/行锁 |
| `ls/` | log stream，复制单元 |
| `meta_mem/`、`meta_store/` | 元数据缓存与持久化 |
| `column_store/` | 列存表 |
| `lob/` | 大对象（CLOB / BLOB） |
| `ddl/` | DDL 物理执行（建索引、加列等） |
| `direct_load/` | 旁路导入 |
| `mview/` | 物化视图 |
| `multi_data_source/` | 多数据源事务支持 |
| `slog/`、`slog_ckpt/` | 存储元数据日志、checkpoint |
| `tmp_file/` | 临时文件（用于排序/外存连接） |
| `backup/`、`restore/`、`high_availability/` | 备份恢复、HA |
| `vector_index/` | **向量索引** —— 见 4.9 |
| `fts/` | **全文索引** —— 见 4.10 |
| `retrieval/` | 与向量索引相关的检索算子 |
| `truncate_info/` | TRUNCATE 元信息 |
| `checkpoint/` | 检查点 |

**第一次读建议**：先读 `access/` 下的接口（`ObStoreRowIterator`、`ObTableScan`），把"存储对外暴露什么"搞清楚；再读一点 `memtable/`；其它先放一放。

### 4.7 transaction & logservice：事务与日志复制

**事务**：`src/storage/tx/`、`src/storage/concurrency_control/`、`src/storage/tx_table/`

事务模型是 OceanBase 的核心创新点之一（Snapshot Isolation + Paxos 提交）。这部分理解曲线陡，建议看完 SQL/存储再来啃。

**日志服务**：`src/logservice/`

提供 PALF（Paxos-replicated Log）和 clog 服务，用来：
- WAL：每次写都先落 clog 才能算成功
- 复制：多副本之间通过 PALF 一致同步
- 恢复：宕机重启后用 clog 重放
- 实现"日志即存储"的设计

单机模式下复制能力被简化，但代码框架还在。

### 4.8 rootserver：集群元信息中心

**位置**：`src/rootserver/`

负责 schema 管理、DDL 调度、负载均衡、leader 选举、tablet 分布、扩缩容等"集群级"动作。在 seekdb 单机/嵌入式模式下，rootserver 仍然存在，但很多功能不会真正用上。当你看 schema、DDL 相关代码时会绕到这里。

### 4.9 向量索引（vector_index / vsag / hnsw）

**位置**：
- `src/storage/vector_index/`
- `src/storage/retrieval/`
- 表达式与函数：`src/sql/engine/expr/` 下与 `vector` / `embedding` 相关的文件

**底层依赖**：[VSAG](https://github.com/antgroup/vsag)，蚂蚁集团开源的向量检索库，封装了 HNSW 等算法。集成代码会通过 `LIB=vsag` 的 DDL 选项触发。

**怎么读**：
1. 先在 `tools/deploy/mysqltest/` 找一个 vector 相关的 case，看看 SQL 表达。
2. 跟着 DDL 路径理解 `CREATE VECTOR INDEX` 是怎么变成存储侧索引创建动作的。
3. 跟着 SELECT 路径理解 `ORDER BY l2_distance(...) APPROXIMATE LIMIT k` 是怎么走向 ANN 检索算子的。

向量检索与 SQL 优化器的交互（比如能不能下推过滤、能不能与全文检索组合）是 seekdb 最有特色的部分。

### 4.10 全文索引（fts）

**位置**：`src/storage/fts/`

主要内容：
- `ob_*ft_parser.cpp` —— 内置分词器：`ngram`、`ngram2`、`ik`（中文）、`whitespace`、`beng`（英文）。
- `ob_fts_doc_word_iterator.cpp` —— 倒排迭代器。
- `ob_fts_plugin_helper.cpp` —— 分词器插件接口。
- `dict/`、`ik/` —— 词典与 ik 分词资源。

**怎么读**：先理解 `MATCH(...) AGAINST(...)` 这套 SQL 语法在 parser/resolver 里如何被识别，再跟到 fts 这边的倒排扫描。

---

## Part 5 — 工具：编译、运行、调试、测试

> 这一节是把根目录 `CLAUDE.md` 里的内容用中文讲一遍 + 加注释。如果你已经看过 `CLAUDE.md`，可以略过。

### 编译

**只用 `build.sh`**，不要直接 cmake。它会自动加载 `deps/3rd/usr/local/oceanbase/devtools` 里的工具链。

```bash
bash build.sh debug   --init --make            # debug 模式，第一次必须加 --init
bash build.sh release --init --make            # release 模式
bash build.sh debug --make -j24                # 第二次起可省 --init
bash build.sh clean                            # 清理
```

构建产物在 `build_<type>/`，主二进制是 `build_<type>/src/observer/seekdb`。

### 部署运行

```bash
./tools/deploy/obd.sh prepare -p /tmp/obtest
./tools/deploy/obd.sh deploy  -c ./tools/deploy/single.yaml
mysql -uroot -h127.0.0.1 -P10000          # 默认 10000 端口（root 部署时）
./tools/deploy/obd.sh destroy --rm -n single
```

`tools/deploy/single.yaml` 是单机部署模板，你可以改里面的内存大小、CPU 数等参数做实验。

### 调试

```bash
# 找进程
pidof seekdb

# 用 gdb / lldb attach
gdb seekdb <pid>
```

debug 模式编出来的二进制带符号，可以打断点、看变量。**强烈建议第一次读 SQL 路径时就开 gdb**。

实用断点位置（按你阅读的顺序）：
1. `ObMPQuery::process` —— MySQL 命令入口
2. `ObSql::stmt_query` —— SQL 总入口
3. `ObResultSet::open` —— 计划开始执行
4. `ObOperator::get_next_row` —— 算子拉数据
5. `ObTableScan::inner_get_next_row` —— 真正读存储

### 测试

#### 单元测试（gtest）

单元测试**默认不会编**，需要进 `unittest/` 子目录手动 make：

```bash
bash build.sh debug --init --make
cd build_debug/unittest
make -j                   # 编全部
./run_tests.sh            # 跑全部
```

跑单个测试（**注意要在 `build_debug` 而不是 `build_debug/unittest` 下 make**）：

```bash
cd build_debug
make -j test_chunk_row_store
./unittest/sql/engine/basic/test_chunk_row_store
```

新增测试：在 `unittest/<对应模块>/` 下新建 `test_xxx.cpp`，并在同目录的 `CMakeLists.txt` 中注册。

#### mysqltest（SQL 回归测试）

case 在 `tools/deploy/mysqltest/test_suite/`，通过 obd.sh 跑：

```bash
cd tools/deploy
./obd.sh mysqltest -n test --all                                   # 跑全部
./obd.sh mysqltest -n test --suite acs                             # 跑一个 suite
./obd.sh mysqltest -n test \
  --test-dir ./mysql_test/test_suite/alter/t \
  --result-dir ./mysql_test/test_suite/alter/r \
  --test-set alter_log_archive_option                              # 跑单个 case
```

mysqltest case 是你"读懂功能"的第二好资源（仅次于看代码本身）：每个新功能通常都有 case，能直接告诉你"它的 SQL 长什么样、期望输出是什么"。

---

## Part 6 — 术语表

> 你在源码 / 注释 / 文档里反复见到的缩写。

| 缩写 | 全称 | 含义 |
|---|---|---|
| **OB** | OceanBase | seekdb 的母体项目；前缀 `Ob` 也来自这里 |
| **MTL** | Multi-Tenant Local | 在当前线程的租户上下文里取对象的宏 |
| **OMT** | OceanBase Multi Tenant | 多租户线程池模块 (`src/observer/omt`) |
| **DAS** | Data Access Service | SQL 访问存储的中间层 (`src/sql/das`) |
| **DTL** | Data Transfer Layer | 算子之间数据传输 (`src/sql/dtl`) |
| **PX** | Parallel eXecution | 并行执行框架 (`src/sql/engine/px`) |
| **PDML** | Parallel DML | 并行 DML (`src/sql/engine/pdml`) |
| **PL** | Procedural Language | PL/SQL 存储过程 (`src/pl`) |
| **LSM** | Log-Structured Merge | LSM-Tree 存储 |
| **MemTable** | | LSM 内存表 |
| **SSTable** | Sorted String Table | LSM 磁盘表 |
| **clog** | Commit Log | WAL；多副本同步日志 |
| **PALF** | Paxos Append-only Log File | OceanBase 的 Paxos 日志实现 (`src/logservice/palf`) |
| **slog** | Storage Log | 存储元数据日志 (`src/storage/slog`) |
| **Tablet** | | 数据分片单位 |
| **LS / Log Stream** | | 一组共享日志的 tablet 集合，复制和 HA 单元 |
| **rs / RootServer** | | 集群元信息中心 |
| **MVCC** | Multi-Version Concurrency Control | 多版本并发控制 |
| **SI** | Snapshot Isolation | 快照隔离 |
| **WDIAG/EDIAG** | Warning/Error Diagnosis | 给开发看的 WARN/ERROR 日志，等同于 LOG_WARN/LOG_ERROR |
| **VSAG** | Vector Search Algorithm Library | 蚂蚁开源的向量检索库 |
| **HNSW** | Hierarchical Navigable Small World | 主流图向量索引 |
| **IVF** | Inverted File index | 倒排式向量索引 |
| **PQ** | Product Quantization | 向量量化压缩 |
| **FTS** | Full-Text Search | 全文检索 |
| **BM25** | | 经典全文检索打分函数 |
| **DDL** | Data Definition Language | 表结构变更 SQL |
| **DML** | Data Manipulation Language | 数据变更 SQL |
| **HTAP** | Hybrid Transactional/Analytical Processing | 同时跑 OLTP 和 OLAP |

---

## Part 7 — 进一步学习资源

### seekdb / OceanBase 官方资料

- **官方开发者指南**（必读）：`docs/developer-guide/en/` 与 `docs/developer-guide/zh/`
- OceanBase 中文官博：<https://open.oceanbase.com/blog>
  - 上面有大量内核解读文章（事务、存储、SQL、PALF、内存等），seekdb 几乎全部适用。
- OceanBase 论文（VLDB / SIGMOD）：搜索 "OceanBase VLDB"
- DeepWiki 自动文档：<https://deepwiki.com/oceanbase/seekdb>
- pyseekdb（Python SDK）：<https://github.com/oceanbase/pyseekdb>
- VSAG（向量库）：<https://github.com/antgroup/vsag>

### 数据库系统通识

如果你完全没学过数据库系统课，下面这些是性价比最高的入门材料：

- **CMU 15-445 / 15-721**（Andy Pavlo）：YouTube 上有完整视频，业内公认的最好数据库系统入门课。
  - 15-445：本科入门版，覆盖存储/索引/事务/查询/恢复。
  - 15-721：研究生进阶版，专攻现代内存/列存/优化器/分布式。
- 教材：《Database System Concepts》（Silberschatz）/ 《Database Management Systems》（Ramakrishnan）
- 论文：**Architecture of a Database System**（Hellerstein, Stonebraker, Hamilton）—— 50 页综述，把 DBMS 的全貌讲清楚。

### 向量检索 & 全文检索

- HNSW 原始论文：Malkov & Yashunin, *Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs*, 2016
- *Information Retrieval: Implementing and Evaluating Search Engines*（Büttcher, Clarke, Cormack） —— 全文检索经典书

### C++

由于 seekdb 风格非常"老派 C++"（避开现代特性），你不需要看现代 C++ 教程。建议熟悉：
- 类继承与虚函数
- 模板基本用法
- RAII / 构造析构
- 函数指针 / 静态多态
- 简单的 lock / atomic 概念

---

## 结语

读这种规模的工业代码是一场马拉松，不是冲刺。你不需要"理解全部"，只需要保持**一边读一边写**的节奏：

> 看 → 跑 → 调 → 改 → 写测试 → 再看下一段

每完成一次这样的循环，你对系统的理解都会前进一格。两三个月后回头看 Part 1，你会发现当时困惑你的每一个词都已经成了直觉。

祝阅读愉快。
