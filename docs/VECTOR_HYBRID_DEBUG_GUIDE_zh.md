# seekdb 向量 / 混合检索：机制讲解 + 断点验证

> 这份文档不是"实验手册"，也不是"架构概念集"。它是**两者的合体**：
>
> - **每个章节先讲清楚一段机制**：seekdb 内部是怎么组织的、为什么这么组织、关键抽象是什么
> - **然后给一个最小实验**：跑一段 SQL、在指定位置设断点，**亲眼看到讲解里的概念在代码里真实地动起来**
>
> 你可以只读"讲解"建立心智模型；也可以只跑"实验"获取手感；最大收益是两者交替——讲解告诉你"应该看到什么"，实验让你确认"确实是这样"。

---

## 目录

- [Part 0 — 必须先建立的心智模型](#part-0--必须先建立的心智模型)
- [Part 1 — 环境准备 + 常见构建坑](#part-1--环境准备--常见构建坑)
- [Part 2 — Debug 准备：让 observer 可被 attach](#part-2--debug-准备让-observer-可被-attach)
- [Part 3 — 一条 SQL 的生命周期：OMT、Plan Cache、Volcano](#part-3--一条-sql-的生命周期omtplan-cachevolcano)
- [Part 4 — 向量索引的"5 表架构"与异步 DDL](#part-4--向量索引的5-表架构与异步-ddl)
- [Part 5 — HNSW ANN 查询：Adapter 模式 + Filter 下推](#part-5--hnsw-ann-查询adapter-模式--filter-下推)
- [Part 6 — IVF 系列：聚类、量化、缓存](#part-6--ivf-系列聚类量化缓存)
- [Part 7 — 全文检索：分词器插件 + 倒排扫描](#part-7--全文检索分词器插件--倒排扫描)
- [Part 8 — Hybrid Search：优化器视角 + 选择性驱动](#part-8--hybrid-search优化器视角--选择性驱动)
- [Part 9 — Hybrid Vector Index：DB 内 embedding 的异步刷新](#part-9--hybrid-vector-indexdb-内-embedding-的异步刷新)
- [Part 10 — 调试技巧汇总](#part-10--调试技巧汇总)
- [Part 11 — 7 天断点学习路线](#part-11--7-天断点学习路线)
- [附录 A — 关键代码地图](#附录-a--关键代码地图)
- [附录 B — 自助找断点位置的方法](#附录-b--自助找断点位置的方法)

---

## Part 0 — 必须先建立的心智模型

读后续任何章节前，先把下面这些"基础事实"刻在脑子里。少一项，后面追代码会迷路。

### 0.1 seekdb 是什么、不是什么

seekdb 是 **OceanBase 内核的派生**，**不是**一个轻量级的"嵌入式向量库"。它继承了 OceanBase 的全套机制：

- **多租户**：所有内存、CPU、IO 都按租户计费，没有"全局变量"那种东西。`ALTER SYSTEM SET ob_vector_memory_limit_percentage = 30` 是按**当前租户**生效。
- **LSM 存储**：写入先进 MemTable，定期 dump/compact 成 SSTable。**向量索引也遵守这套规则**——这是后面"5 表架构"的根因。
- **PALF Paxos 复制**：日志多副本（单节点部署时退化成单副本，但代码路径完全一样）。
- **DDL 异步任务化**：`CREATE INDEX` 不是同步执行的，rootserver 接收后丢给 task framework 后台跑。

理解这点很重要——你看到的所有"复杂"都不是为了向量本身，而是为了**让向量融入 LSM + 多租户 + 分布式这套既有体系**。

### 0.2 "Hybrid" 这个词被用在两件不同的事上

| 术语 | 含义 | DDL 触发 |
|------|------|----------|
| **Hybrid Vector Index** | 索引建在**文本列**上，DB 内部自动调 embedding 模型把文本转向量 | `VECTOR INDEX(text_col) WITH(model=ob_embed, ...)` + 先注册 AI 模型 |
| **Hybrid Search** | 同一条 SQL 里**同时用向量距离 + 全文打分** | `WHERE MATCH(c) AGAINST(...) ORDER BY l2_distance(...) APPROXIMATE` |

两者**正交**：hybrid search 既能查"普通向量索引"（你预先算好 embedding），也能查"hybrid vector index"。本文 Part 5-8 是 hybrid search，Part 9 才是 hybrid vector index。

### 0.3 向量索引内部的关键抽象（必看）

下面三个设计决定了你后续看到的几乎所有现象：

**(a) 一个向量索引 = 5 张内部表**（不是一张！）

`src/share/vector_index/ob_plugin_vector_index_adaptor.h` 里 `ObVectorIndexInfo` 直接列出了：

```
rowkey_vid_table       ← 用户 rowkey → 内部 vid 映射
vid_rowkey_table       ← 反向：vid → rowkey
inc_index_table        ← 增量数据：刚写入但还没合到主索引的向量
snapshot_index_table   ← 主 HNSW 索引（合并后的快照）
vbitmap_table          ← 删除位图：标记哪些 vid 已被删除
```

外加一张 `data_table`（用户原表）。所以一个 `CREATE VECTOR INDEX` 之后 `oceanbase.__all_table` 里会冒出 4-5 行新记录，对应这些 hidden table。

**为什么这么设计**：seekdb 的存储是 LSM。HNSW 索引一旦构建好不能频繁 in-place 更新（图结构会退化）。所以新写入先进 `inc_index_table`（小、可增量、查询时 brute-force），定期通过 **refresh task** 合并到 `snapshot_index_table`（大、HNSW 图、查询时走 ANN）。删除则只在 `vbitmap_table` 里标记，查询时通过 filter 跳过。

**这就解释了**：
- 为什么有 `dbms_vector.refresh_index` 这种存储过程
- 为什么 `EXPLAIN` 里能看到向量扫描有"两阶段"
- 为什么刚 INSERT 完查询性能可能突然下降（增量太多）
- 为什么 `ObPluginVectorIndexAdaptor` 里有 `PVQ_LACK_SCN`、`PVQ_REFRESH` 这些状态——查询要等增量索引刷新到一个一致的版本

**(b) Adapter 模式：算法实现可插拔**

`ObPluginVectorIndexAdaptor` 是统一接口；底下挂 vsag（HNSW 系列）、native IVF（IVF_FLAT/SQ8/PQ）等不同后端。`WITH(type=hnsw, lib=vsag)` 的 `lib` 参数选后端，`type` 选算法。

`deps/oblib/src/lib/vector/ob_vsag_adaptor.h` 里看 vsag 当前支持的类型：`HNSW`、`HNSW_SQ`（标量量化）、`HNSW_BQ`（二值量化）、`HGRAPH`、`IPIVF`。IVF_FLAT/SQ8/PQ 是 seekdb **自己实现**的（在 `ob_vector_kmeans_ctx.cpp` 等），不在 vsag 里。

**(c) Filter 下推到 HNSW 图遍历**

这是 hybrid search 的关键。常规做法是"先 ANN 取 top-N，再用 WHERE 过滤"，问题是过滤后可能不够 N 行。seekdb 的做法是把 WHERE 子句构造成 `ObHnswBitmapFilter`（基于 roaring bitmap），传给 vsag，**vsag 在图遍历时就跳过不满足 filter 的节点**。这样既保证了 top-N 数量，又不用反复 retry。

`src/share/vector_index/ob_plugin_vector_index_adaptor.h:144` `ObHnswBitmapFilter` 实现 `vsag::FilterInterface`。

### 0.4 全文检索的关键抽象

- **倒排索引**：把文档拆成词，建立"词 → 文档列表"映射。查询时按词查倒排表，再合并打分。
- **分词器是插件**：`ob_*_ft_parser.{h,cpp}`，包括 `whitespace`（按空白切）、`ngram`（n 元切，常用 2-gram）、`ik`（中文专用）、`beng`（英文专用）。`CREATE FULLTEXT INDEX ... WITH PARSER ik` 的 `ik` 就是选哪个分词器。
- **打分**：`MATCH(c) AGAINST('q')` 返回相关性分数（BM25 类）。

### 0.5 SQL 执行的 Volcano 模型

OceanBase 的执行引擎是经典 Volcano（也叫 Iterator）模型：

```
LIMIT (top)
  └─ SORT
      └─ FILTER
          └─ SCAN (leaf)
```

每个算子实现 `get_next_row()`。父算子调子算子的 `get_next_row()`，子算子返回一行就交给父算子处理。**整个执行就是一棵算子树的递归 next 调用**。这是后面所有 "在算子上下断点" 的基础。

ANN 扫描在这个模型里就是一个特殊的 SCAN 算子（叫法不一，可能是 `ObTableScanWithVecIndex` / `ObVectorIndexScan` / `PHY_VEC_IDX_SCAN`，以你 `EXPLAIN` 看到的为准）。

---

## Part 1 — 环境准备 + 常见构建坑

### 1.1 Debug 构建

```bash
cd ~/seekdb_explore                              # 你的实际路径
bash build.sh debug --init --make -j$(nproc)
```

构建类型对比：

| 类型 | 用途 | 优化 | 调试信息 | 速度 |
|------|------|------|----------|------|
| `debug` | 开发 / gdb | -O0 | 完整 | 慢 |
| `release` | 生产 | -O2 + LTO | 部分 | 快 |
| `errsim` | 错误注入测试 | 中 | 完整 | 中 |
| `dissearray` | 离散数组优化 | 类 release | 部分 | 快 |
| `rpm` | 打包 | 类 release | minimal | 快 |

**永远用 `bash build.sh` 不要直接 `cmake`**——它会注入正确的 toolchain（`deps/3rd/usr/local/oceanbase/devtools/`）和 ASAN/LLD/BOLT 等编译选项。

### 1.2 你大概率会撞到的几个坑

**坑 1：`rpm2cpio: command not found`（Ubuntu/Debian 必中）**

seekdb 的 deps 是 RPM 包，Ubuntu 没自带 RPM 解包工具：

```bash
sudo apt-get install -y git wget rpm rpm2cpio cpio make build-essential \
                        binutils m4 file python3 libaio-dev
```

`libaio-dev` observer 链接时要用，提前装。

**坑 2：`libaio1t64` 缺失（仅 Ubuntu 24.04+ / Debian 13+）**

```bash
sudo apt-get install -y libaio1t64
```

**坑 3：deps 下载失败 / 慢**

阿里云的 mirror 偶尔抽风，错误形如 `Failed to download <xxx.rpm>`。**直接重跑 `bash build.sh debug --init --make`**，`--init` 是幂等的，会从中断点继续。

如果反复失败，看一眼当前选的 mirror：

```bash
grep "repo:" deps/init/oceanbase.el9.x86_64.deps   # 你的发行版改对应文件
```

可以临时换成你公司内网镜像。

**坑 4：链接阶段 OOM**

debug 构建链接 observer 时一个进程要吃 8-15GB 内存。如果机器只有 16GB 你又开了 `-j$(nproc)`，链接器会被 OOM Killer 干掉。降并发：

```bash
bash build.sh debug --make -j4    # 链接阶段单独限制
```

或者用 `--ninja` 让 Ninja 自己管资源调度。

**坑 5：磁盘空间**

debug 构建产物 + deps 大约 **30-50 GB**。开始前 `df -h .` 看一眼。

**坑 6：编译产物在哪**

```
build_debug/
├── compile_commands.json          ← 给 clangd / VS Code 用
├── src/observer/seekdb            ← 主二进制（几个 GB）
├── unittest/                      ← 单元测试（make 默认不编译，需 cd 进去再 make）
└── ...
```

把 `compile_commands.json` 软链到仓库根，给 IDE 用：

```bash
ln -sf build_debug/compile_commands.json compile_commands.json
```

### 1.3 部署 + 连接

```bash
cd tools/deploy
./obd.sh prepare -p /tmp/obtest
./obd.sh deploy -c ./single.yaml

mysql -uroot -h127.0.0.1 -P10000

# 出问题就：
./obd.sh destroy --rm -n single
```

**确认 deploy 用的是你刚 build 的 debug 版**：

```bash
ls -l /tmp/obtest/single/bin/seekdb
md5sum /tmp/obtest/single/bin/seekdb build_debug/src/observer/seekdb
# 不一致就手动覆盖：
cp build_debug/src/observer/seekdb /tmp/obtest/single/bin/seekdb
# 然后重启
```

连上后初始化：

```sql
USE oceanbase;
ALTER SYSTEM SET ob_vector_memory_limit_percentage = 30;
ALTER SYSTEM SET enable_sql_audit = true;
CREATE DATABASE vec_test;
USE vec_test;
```

---

## Part 2 — Debug 准备：让 observer 可被 attach

### 2.1 找进程

```bash
pgrep -a observer                  # 记下 PID 为 $OBPID
```

### 2.2 关闭可能干扰的自检（强烈建议）

observer 默认有看门狗 / 健康检查线程。被 gdb 暂停几秒就可能被认为"僵死"自杀，或者副本被切走。

```sql
ALTER SYSTEM SET _enable_check_diagnose_info = false;
ALTER SYSTEM SET enable_perf_event = false;
ALTER SYSTEM SET rpc_timeout = 3600000000;       -- 1 小时
```

### 2.3 attach

**命令行：**

```bash
sudo gdb -p $OBPID
(gdb) set pagination off
(gdb) set print pretty on
(gdb) handle SIGPIPE nostop noprint
(gdb) handle SIG34 SIG35 SIG36 nostop noprint
```

> ⚠️ attach 时**整个 observer 进程冻结**——所有线程都停。客户端的连接也会卡住。专用调试机，别 attach 生产实例。

**VS Code Remote SSH：**

仓库根 `.vscode/launch.json`：

```jsonc
{
  "version": "0.2.0",
  "configurations": [{
    "name": "attach to observer",
    "type": "cppdbg",
    "request": "attach",
    "program": "/tmp/obtest/single/bin/seekdb",
    "processId": "${command:pickProcess}",
    "MIMode": "gdb",
    "setupCommands": [
      { "text": "set pagination off" },
      { "text": "handle SIGPIPE nostop noprint" },
      { "text": "handle SIG34 SIG35 SIG36 nostop noprint" }
    ]
  }]
}
```

按 F5 → 选 observer 进程 → 在编辑器里点行号下断点。

### 2.4 日志补刀

很多路径走异步 task 线程，断点不一定能蹲到。打开 DEBUG 日志补全：

```sql
ALTER SYSTEM SET syslog_level = 'INFO,STORAGE.VECTOR:DEBUG';
```

```bash
tail -F /tmp/obtest/single/log/observer.log | grep -i 'trace_id=YXxxx'
```

trace_id 怎么拿：`SELECT TRACE_ID FROM oceanbase.GV$OB_SQL_AUDIT ORDER BY REQUEST_TIME DESC LIMIT 1;`

---

## Part 3 — 一条 SQL 的生命周期：OMT、Plan Cache、Volcano

### 3.1 机制讲解

#### 网络层 → MySQL 协议 → SQL 主入口

观察者起一组 **网络 IO 线程** 监听 MySQL 端口（默认 10000）。一个客户端连接绑定到一个 worker 线程；worker 线程上下文里包含**当前租户**（OMT，OceanBase Multi-Tenant）。所有内存分配都从租户的 allocator 走，不会跨租户串。

MySQL 协议层（`src/observer/mysql/`）解析 COM_QUERY 命令，把 SQL 字符串交给 `ObSql::stmt_query`。

#### Plan Cache：第一次 vs 之后

`ObSql::stmt_query` 第一步是查 **Plan Cache**：用 SQL 的"参数化签名"（把字面量替换成 `?`）作为 key 查缓存。

- **命中**：直接拿到 `ObPhysicalPlan`，跳过 parse/resolve/optimize/codegen
- **未命中**：走完整流程：parser → resolver → optimizer → code generator → 生成 `ObPhysicalPlan` → 插入 plan cache

**这就解释了为什么有时候你在 `generate_physical_plan` 上设断点不会触发**——SQL 模式之前已经被缓存了。要让 plan cache miss：

```sql
ALTER SYSTEM FLUSH PLAN CACHE GLOBAL;
```

或在 SQL 前加 hint：`SELECT /*+ NO_USE_PLAN_CACHE */ ...`。

#### Volcano 执行模型

物理 plan 是一棵 **算子树**。每个算子继承 `ObOperator`，实现 `inner_open` / `inner_get_next_row` / `inner_close`。

```
ObResultSet::open()                ← 顶层入口
  → root_op->open()                ← 递归 open 整棵树
ObResultSet::get_next_row()
  → root_op->get_next_row()        ← 父调子，子返回一行往上传
ObResultSet::close()
  → root_op->close()
```

整个执行就是反复调用 `get_next_row`。算子之间通过**值传递**（一行行）或**批传递**（向量化执行）。看 `EXPLAIN` 的输出，每一行就是一个算子。

#### 多租户内存的影响

为什么文档反复强调"租户内存"？因为：

- 你 `ALTER SYSTEM SET ob_vector_memory_limit_percentage = 30` 给的是**当前租户内存的 30%**
- 向量索引（HNSW 图、IVF 倒排表）都从这个池子分配
- 内存吃光会触发 OOM，但**不是杀进程，是这个查询失败**——错误码 `-4013 OB_ALLOCATE_MEMORY_FAILED`

### 3.2 实验验证

**SQL：**

```sql
USE vec_test;
ALTER SYSTEM FLUSH PLAN CACHE GLOBAL;
SELECT 1;
```

**断点（按调用顺序）：**

| # | 函数 | 位置 | 你应该看到 |
|---|------|------|-----------|
| 1 | `ObMPQuery::process` | `src/observer/mysql/obmp_query.cpp` | `sql_` 字符串就是 `SELECT 1` |
| 2 | `ObSql::stmt_query` | `src/sql/ob_sql.cpp` | 这里有 plan cache 查询逻辑 |
| 3 | `ObSql::generate_physical_plan` | `src/sql/ob_sql.cpp` | 因为 plan cache 被 flush 了，会进 |
| 4 | `ObResultSet::open` | `src/sql/ob_result_set.cpp` | 整棵算子树开始 open |
| 5 | `ObOperator::get_next_row` | `src/sql/engine/ob_operator.cpp` | 通用基类，所有算子的 next 都从这里进 |

**操作：**

```gdb
(gdb) break ObMPQuery::process
(gdb) break ObResultSet::open
(gdb) continue
```

发 SQL，停在 `process`：

```gdb
(gdb) p sql_                       # 看到 "SELECT 1"
(gdb) bt 30                        # 看上层栈：网络层 → RPC → 协议
(gdb) c                            # 跳到 ObResultSet::open
(gdb) bt 30                        # 看 plan 是怎么被运行起来的
```

**第二次发同样 SQL（不 flush plan cache）**：观察 `generate_physical_plan` 是否触发——应该不触发，证明走了 plan cache。

### 3.3 小结

跑完这一节你应该能回答：

1. 一条 SQL 从 TCP 连接到返回结果，**至少经过几个层次的抽象**？（答：网络/IO 线程 → MySQL 协议 → SQL 主入口 → optimizer/codegen → 算子树执行）
2. 为什么 OB 的算子代码里到处都是 `inner_get_next_row`？（答：Volcano 模型，父算子驱动子算子）
3. 为什么调试时同一条 SQL 跑两次行为不一样？（答：第二次走 plan cache，跳过了 parse/resolve/optimize）

---

## Part 4 — 向量索引的"5 表架构"与异步 DDL

### 4.1 机制讲解

#### 为什么是 5 张表

回顾 Part 0.3：HNSW 图结构不能频繁 in-place 更新。所以 seekdb 的设计是：

```
┌─────────────────────────────────────────────────────────┐
│ 用户表  data_table                                     │
│   ├─ id: 1, embedding: [0.1, 0.2, ...]                 │
│   └─ ...                                                │
└─────────────────────────────────────────────────────────┘
            │
            │ CREATE VECTOR INDEX 后衍生：
            ▼
┌─────────────────────────────────────────────────────────┐
│ rowkey_vid_table   (id=1) → vid=42                      │
│ vid_rowkey_table   vid=42 → (id=1)                      │
│                                                          │
│ snapshot_index_table   ← HNSW 图（主索引，构建好的快照）│
│   持久化的 vsag 序列化数据                              │
│                                                          │
│ inc_index_table        ← 增量索引（新写入的向量）       │
│   小、行式存储，查询时 brute-force 扫                   │
│                                                          │
│ vbitmap_table          ← 删除位图                       │
│   roaring bitmap，标记被删除的 vid                      │
└─────────────────────────────────────────────────────────┘
```

**写入流程**：

```
INSERT INTO docs VALUES (5, [0.5, ...])
  → 写 data_table（用户表）
  → 分配 vid=46
  → 写 rowkey_vid_table、vid_rowkey_table
  → 把向量追加到 inc_index_table
  → snapshot_index_table 不动
```

**查询流程**：

```
SELECT ... ORDER BY l2_distance(...) APPROXIMATE LIMIT 10
  → 在 snapshot 上做 HNSW ANN，拿 top-K 候选
  → 在 inc 上 brute-force 算距离，拿 top-K 候选
  → 用 vbitmap 过滤掉已删除的 vid
  → 合并 + rerank → 返回最终 top-10
```

**Refresh 流程**（异步）：

```
后台 task 周期性触发：
  → 读 inc_index_table 里的所有向量
  → 调用 vsag 把它们 add 到 snapshot HNSW 图
  → 更新 snapshot_index_table 的序列化数据
  → 清空 inc_index_table
```

**对应到代码**：
- 写入：`src/storage/access/ob_vector_store.cpp`、`src/share/vector_index/ob_plugin_vector_index_*`
- Refresh：`src/storage/vector_index/ob_vector_index_refresh.cpp`、`ob_vector_index_sched_job_utils.cpp`、`src/share/vector_index/ob_vector_index_async_task_util.cpp`
- 状态枚举：`PluginVectorQueryResStatus { PVQ_START, PVQ_WAIT, PVQ_LACK_SCN, PVQ_OK, PVQ_REFRESH, ... }` 在 `ob_plugin_vector_index_adaptor.h`

`PVQ_LACK_SCN` 解读：查询请求带了一个 SCN（一致性版本号），如果 snapshot 还没 refresh 到那么新，就要先等 refresh 完成。

#### CREATE VECTOR INDEX 的异步 DDL

OceanBase 的 DDL 框架是 **task-based 状态机**：

```
client → ObDDLService::create_index_table     ← 收到请求，立刻返回
            → 创建 ObDDLTask 记录写入 __all_ddl_task_status
            → ObVecIndexBuildTask::process    ← 后台 task 调度，多次进入
                ├─ state = WAIT_TRANS_END     ← 等待相关事务结束
                ├─ state = REDEFINITION       ← 创建 5 张内部表
                ├─ state = COPY_TABLE_DEPENDENT_OBJECTS
                ├─ state = TAKE_EFFECT        ← 完成
                └─ state = SUCCESS / FAIL
```

**这就解释了**：
- `CREATE VECTOR INDEX` 立刻返回但实际还没建好——你紧跟着查可能拿到空结果
- `dbms_vector.refresh_index('idx', ...)` 是手动触发 refresh task（默认是 lazy 的）
- 中间状态可以查：`SELECT * FROM oceanbase.__all_virtual_ddl_task_status;`

### 4.2 实验验证

**SQL：**

```sql
USE vec_test;
DROP TABLE IF EXISTS docs;
CREATE TABLE docs (
  id INT PRIMARY KEY,
  embedding VECTOR(10)
);

CREATE VECTOR INDEX idx_vec ON docs(embedding) WITH (distance=l2, type=hnsw, lib=vsag);
```

**断点：**

| # | 函数 | 文件 | 你将看到 |
|---|------|------|---------|
| 1 | `ObCreateIndexResolver::resolve` | `src/sql/resolver/ddl/` | DDL 语法被解析 |
| 2 | `ObDDLService::create_index_table` | `src/rootserver/ob_ddl_service.cpp` | rootserver 接收 |
| 3 | `ObVecIndexBuildTask::process` | `src/rootserver/ddl_task/ob_vec_index_build_task.cpp` | **状态机被反复调用** |
| 4 | `ObPluginVectorIndexAdaptor::init` 或 `::create_index` | `src/share/vector_index/ob_plugin_vector_index_adaptor.cpp` | 索引插件初始化 |
| 5 | `ObVsagAdaptor::build_index` | `deps/oblib/src/lib/vector/ob_vsag_adaptor.cpp` | 真正调 vsag 库 |

**操作：**

```gdb
(gdb) break ObVecIndexBuildTask::process
(gdb) break ObVsagAdaptor::build_index
(gdb) continue
```

发 CREATE INDEX。task 调度有延迟，几秒到几十秒后会停在 `process`：

```gdb
(gdb) p task_status_                # 当前 state，每次进 process 应该不同
(gdb) p task_id_, table_id_, index_id_
(gdb) c                             # 让它继续到下一个 state，反复几次
```

最终（或者中间某次）会进 `ObVsagAdaptor::build_index`：

```gdb
(gdb) p dim_                        # 维度，应该是 10
(gdb) p max_elements_, ef_construction_, M_   # HNSW 参数
(gdb) bt                            # 看从 task → adaptor → vsag 的完整栈
```

**验证 5 表架构：**

```sql
SELECT table_name FROM oceanbase.__all_table
WHERE data_table_id = (SELECT table_id FROM oceanbase.__all_table WHERE table_name='docs');
-- 应该看到 docs + idx_vec 系列内部表（rowkey_vid / vid_rowkey / inc / snapshot / vbitmap）

SELECT * FROM oceanbase.__all_virtual_ddl_task_status\G
-- 看 DDL 任务的当前状态
```

### 4.3 小结

1. 一个向量索引在底层是**几张表**？为什么这么设计？（5 张，因为 LSM 不能 in-place 更新 HNSW 图）
2. `CREATE VECTOR INDEX` 是同步还是异步？（异步，task framework 调度）
3. 为什么会有 `dbms_vector.refresh_index` 这个东西？（手动触发增量合并到主 HNSW）

---

## Part 5 — HNSW ANN 查询：Adapter 模式 + Filter 下推

### 5.1 机制讲解

#### HNSW 算法 30 秒速通

HNSW (Hierarchical Navigable Small World) 是一个**多层图**：

```
Layer 2:    A ─────────── F        ← 稀疏，长跳
Layer 1:    A ──── C ──── F ──── H ← 中等
Layer 0:    A ─ B ─ C ─ D ─ E ─ F ─ G ─ H  ← 密集，所有点
            ↑ 入口点
```

- 每个向量是一个节点。建索引时按概率随机决定该点出现在前几层（高层稀疏）
- 节点之间根据距离建边（NN graph），每层节点最多有 `M` 个邻居
- **查询**：从顶层入口出发，每层贪心走"距离 query 最近的邻居"，到达局部最优后下一层。`ef_search` 控制候选队列大小（越大越准越慢）

**关键参数**（建索引时由 `WITH(...)` 设定，存在 `ObVectorIndexParam`）：
- `M`：每节点最大邻居数（典型 16-64）
- `ef_construction`：构建时候选队列大小（典型 200-500）
- `ef_search`：查询时候选队列大小（运行时可调，hint 或 system var）

#### Adapter 模式：vsag 是后端之一

```
ObPluginVectorIndexAdaptor (统一接口)
       │
       ├─ ObVsagAdaptor  → libvsag.so   (HNSW / HNSW_SQ / HNSW_BQ / HGRAPH / IPIVF)
       └─ (native IVF impl)              (IVF_FLAT / IVF_SQ8 / IVF_PQ)
```

`WITH(type=hnsw, lib=vsag)` 选哪个。

`deps/oblib/src/lib/vector/ob_vsag_adaptor.cpp` 顶部可以看到 `#include "vsag/vsag.h"` 等头文件——这就是把 vsag 当作第三方库链接进来。条件编译宏 `OB_BUILD_CDC_DISABLE_VSAG` 可以禁用 vsag（用于 CDC 等不需要向量的构建）。

#### Filter 下推：hybrid filter 的精髓

考虑这条 SQL：

```sql
SELECT id FROM docs
WHERE category = 'AI'
ORDER BY l2_distance(embedding, '[...]') APPROXIMATE LIMIT 10;
```

朴素做法 ①：先 ANN 取 top-100，再用 `category='AI'` 过滤 → 可能不到 10 行；
朴素做法 ②：先 `category='AI'` 过滤拿 1M 行，再 brute-force 算距离取 top-10 → 慢；

**seekdb 的做法**：把 `category='AI'` 的结果集（一组 vid）做成 roaring bitmap，包成 `ObHnswBitmapFilter`（实现 `vsag::FilterInterface`），传给 vsag。**vsag 在图遍历时调用 `filter.test(vid)`，不满足的节点直接跳过、不参与候选队列**。这样既保证 top-10 数量，又只遍历一遍图。

代码路径：
- `src/share/vector_index/ob_plugin_vector_index_adaptor.h:144` `ObHnswBitmapFilter`
- 实现 `obvsag::FilterInterface::test(int64_t id)`
- 传给 `ObVsagAdaptor::knn_search` 的 filter 参数

#### 查询时为什么要扫两次（snapshot + inc）

因为 4.1 讲的 5 表架构。`ObPluginVectorIndexAdaptor::query` 内部会：

1. 在 `snapshot_index_table` 上调 `ObVsagAdaptor::knn_search` 拿 K 个候选
2. 在 `inc_index_table` 上 brute-force 算所有距离拿 K 个候选
3. 用 `vbitmap_table` 过滤已删除
4. 合并 + 按距离 rerank → 返回 top-K

**所以你看 `EXPLAIN` 时，向量索引的扫描节点会涉及多个 tablet**。

#### `APPROXIMATE` 关键字的语义

- 有 `APPROXIMATE`：optimizer 走向量索引算子，调用 vsag knn_search → 近似 top-K，快
- 无 `APPROXIMATE`：optimizer 走普通 table scan + sort，对所有行精确算距离 → 精确 top-K，慢但精确

这是用户**显式声明**精度/速度 trade-off 的方式，optimizer 不会替你猜。

### 5.2 实验验证

**准备数据：**

```sql
INSERT INTO docs VALUES
  (1, '[0.20,0.21,0.88,0.82,0.62,0.50,0.98,0.87,0.25,0.54]'),
  (2, '[0.74,0.67,0.90,0.45,0.23,0.66,0.77,0.23,0.58,0.93]'),
  (3, '[0.33,0.05,0.08,0.39,0.97,0.37,0.18,0.94,0.01,0.63]'),
  (4, '[0.14,0.87,0.02,0.32,0.04,0.14,0.71,0.44,0.63,0.63]'),
  (5, '[0.33,0.85,0.88,0.66,0.98,0.41,0.20,0.19,0.95,0.79]');

-- 让增量先合并到 snapshot，避免现在数据全在 inc 里看不出 HNSW 路径
CALL dbms_vector.refresh_index('idx_vec', 'docs', 'embedding', 1, 'FAST');
```

**实验 SQL：**

```sql
EXPLAIN
SELECT id
FROM docs
ORDER BY l2_distance(embedding, '[0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5]') APPROXIMATE
LIMIT 3;
```

记下 EXPLAIN 输出里的算子名（`PHY_VEC_IDX_SCAN` 类似的）。

**断点：**

```gdb
(gdb) break ObVsagAdaptor::knn_search
(gdb) continue
```

发 SELECT，停下后：

```gdb
(gdb) bt 25                          # 看 ObResultSet → 算子 → adaptor → vsag 的完整栈
(gdb) p k                            # top-k = 3
(gdb) p dim_                         # 10
(gdb) p ef_search_                   # 默认值
(gdb) finish                         # 看返回的 distance / vid
```

**对比实验：去掉 `APPROXIMATE`：**

```sql
EXPLAIN
SELECT id FROM docs
ORDER BY l2_distance(embedding, '[0.5,...]')
LIMIT 3;
```

`EXPLAIN` 应该完全不同——没有向量索引算子。`ObVsagAdaptor::knn_search` 的断点也不会触发。这就证明了 `APPROXIMATE` 的开关作用。

**Filter 下推实验：**

```sql
ALTER TABLE docs ADD COLUMN category VARCHAR(16);
UPDATE docs SET category = IF(id <= 2, 'AI', 'DB');

SELECT id FROM docs
WHERE category = 'AI'
ORDER BY l2_distance(embedding, '[0.5,...]') APPROXIMATE LIMIT 5;
```

下断点 `ObHnswBitmapFilter::test`，看到每个候选 vid 都会调一次 `test`——这就是过滤下推到图遍历内部。

### 5.3 小结

1. HNSW 大概怎么搜的？（多层贪心图遍历）
2. seekdb 怎么把 SQL 的 WHERE 子句和 ANN 结合？（构造 roaring bitmap → 包成 vsag::FilterInterface → 在图遍历时 test）
3. `APPROXIMATE` 关键字的作用？（开关：要不要走向量索引算子）
4. 为什么一次查询会触发两次"扫描"逻辑？（snapshot HNSW + inc brute-force，最后合并）

---

## Part 6 — IVF 系列：聚类、量化、缓存

### 6.1 机制讲解

#### IVF 的核心思想

IVF (Inverted File Index) 把向量空间划成 `nlist` 个簇（k-means 聚类），查询时只扫"离 query 最近的 `nprobe` 个簇"，跳过其他簇。

```
向量空间被划成 N 个簇：
   ┌───┬───┬───┐
   │ • │   │ ░ │   • 是 query
   ├───┼───┼───┤   ░ 是 query 落入的簇 + 邻近簇
   │   │ ░ │   │   只扫这些簇里的向量
   ├───┼───┼───┤
   │   │   │   │
   └───┴───┴───┘
```

参数：
- `nlist`：簇数（典型 sqrt(N)，N 是数据量）
- `nprobe`：查询扫几个簇（小则快，大则准）

#### 三种 IVF 变种的区别

| 类型 | 簇内向量怎么存 | 内存 | 精度 |
|------|---------------|------|------|
| `IVF_FLAT` | 原 float32 向量 | 大 | 高 |
| `IVF_SQ8` | int8 量化（每维 1 字节） | 1/4 | 中 |
| `IVF_PQ` | 乘积量化（拆成 m 段，每段独立量化） | 极小 | 中低 |

#### IVF Cache：为什么需要专门的缓存

簇中心（centroids）查询时每次都要用，又不大，但放在用户表里太慢。所以 seekdb 有 `ObVectorIndexIvfCacheMgr`（`src/share/vector_index/ob_vector_index_ivf_cache_mgr.cpp`）专门缓存：

- centroids（聚类中心）
- 簇 → 向量列表的反向映射

第一次查询会触发 cache load；之后的查询直接命中缓存。

#### 与 HNSW 的取舍

| 维度 | HNSW | IVF |
|------|------|-----|
| 召回率 | 高 | 中 |
| QPS | 高 | 中 |
| 内存 | 大（图） | 中-小 |
| 构建速度 | 慢 | 快 |
| 增量友好 | 不友好（需 refresh 重建） | 较友好 |
| 适合场景 | 静态数据、追求性能 | 大数据量、内存受限 |

### 6.2 实验验证

```sql
DROP TABLE IF EXISTS docs_ivf;
CREATE TABLE docs_ivf (id INT PRIMARY KEY, embedding VECTOR(10));
CREATE VECTOR INDEX idx_ivf ON docs_ivf(embedding) WITH (distance=l2, type=ivf_flat, lists=10);

-- 插点数据然后 refresh
INSERT INTO docs_ivf SELECT id, embedding FROM docs;
CALL dbms_vector.refresh_index('idx_ivf', 'docs_ivf', 'embedding', 1, 'FAST');

EXPLAIN SELECT id FROM docs_ivf
ORDER BY l2_distance(embedding, '[0.5,...]') APPROXIMATE LIMIT 3;
```

**断点：**

```gdb
(gdb) break ObVectorKmeansCtx::*       # k-means 聚类（建索引时）
(gdb) break ObVectorIndexIvfCacheMgr::*   # 缓存加载
```

CREATE INDEX 时停在 kmeans，看聚类过程；查询时停在 cache mgr，看缓存如何被填充。

### 6.3 小结

1. IVF 和 HNSW 的本质区别？（IVF 是"分桶 + 桶内扫描"，HNSW 是"图遍历"）
2. SQ8 / PQ 解决什么问题？（内存）
3. 为什么 IVF 需要专门的 cache？（centroids 频繁访问）

---

## Part 7 — 全文检索：分词器插件 + 倒排扫描

### 7.1 机制讲解

#### 倒排索引基础

文档：`{1: "机器学习是人工智能", 2: "数据库系统提供存储"}`

分词后：

```
文档 1 → ["机器", "学习", "是", "人工智能"]
文档 2 → ["数据库", "系统", "提供", "存储"]
```

倒排：

```
"机器"     → [1]
"学习"     → [1]
"人工智能" → [1]
"数据库"   → [2]
"系统"     → [2]
...
```

查 `MATCH 'AGAINST 数据库'`：拿"数据库"的 posting list `[2]` → 直接拿到候选文档 → 算 BM25 打分。

#### 分词器是插件

`src/storage/fts/` 下：

- `ob_whitespace_ft_parser`：按空白切（英文）
- `ob_ngram_ft_parser` / `ob_ngram2_ft_parser`：n-gram 切（通用，常用于中文）
- `ob_ik_ft_parser` + `src/storage/fts/ik/`：IK 中文分词（语义级）
- `ob_beng_ft_parser`：英文专用

`CREATE FULLTEXT INDEX ... WITH PARSER ik` 选用哪个。所有分词器实现统一接口（`ObFTParser` 类似），输入文本输出 token 列表。

**IK 分词的复杂性**：在 `src/storage/fts/ik/` 下能看到 `ob_ik_arbitrator`（多种切法仲裁）、`ob_ik_cjk_processor`（CJK 字符处理）、`ob_ik_quantifier_processor`（量词处理）等——这是因为中文分词有歧义。

#### 倒排扫描算子

具体算子名要靠 `EXPLAIN` 找。通常包括：

- 单词分词：`ObIKFTParser::segment`
- 倒排表查找：`ObFTSDocWordIterator`
- 多词合并：交集 / 并集计算
- BM25 打分：基于 tf, idf, doc length

#### 与向量索引的对称性

注意：FTS 也有"5 表架构"的影子（虽然叫法不同），也有"分词词典 vs 文档倒排"的分离，也有插件接口。这是 OB 的一贯设计风格：**抽象、插件化、LSM 友好**。

### 7.2 实验验证

```sql
DROP TABLE IF EXISTS articles;
CREATE TABLE articles (
  id INT PRIMARY KEY,
  content TEXT,
  FULLTEXT INDEX idx_fts(content) WITH PARSER ik
);

INSERT INTO articles VALUES
  (1, '机器学习是人工智能的一个重要分支'),
  (2, '现代数据库提供高性能存储与查询能力'),
  (3, '向量数据库支持语义级别的相似搜索');

EXPLAIN
SELECT id, MATCH(content) AGAINST('数据库') AS score
FROM articles WHERE MATCH(content) AGAINST('数据库') ORDER BY score DESC;
```

**断点：**

```gdb
(gdb) break ObIKFTParser::segment
(gdb) continue
```

发 SELECT。停在 segment：

```gdb
(gdb) p text_                        # query 文本 "数据库"
(gdb) finish                         # 跑完看输出 token 列表
```

中文分词调试很直观——能看到"数据库"被切成什么 token，然后这些 token 怎么去倒排索引取 docid。

### 7.3 小结

1. 倒排索引的核心数据结构？（词 → 文档列表）
2. 为什么分词器要做成插件？（不同语言、不同场景需求差异大）
3. IK 比 ngram 复杂在哪？（语义切分 + 歧义仲裁）

---

## Part 8 — Hybrid Search：优化器视角 + 选择性驱动

### 8.1 机制讲解

#### 优化器看到的 hybrid SQL

```sql
SELECT id
FROM hybrid_docs
WHERE MATCH(content) AGAINST('数据库')
ORDER BY l2_distance(embedding, '[...]') APPROXIMATE LIMIT 10;
```

optimizer 拿到这个 plan tree 时看到的是：

```
LIMIT 10
  └─ SORT BY l2_distance APPROXIMATE
      └─ FILTER (MATCH AGAINST)
          └─ TABLE SCAN hybrid_docs
```

但 hybrid_docs 既有 vector index 又有 fulltext index。optimizer 必须决定：**用哪些索引、怎么组合**。常见策略：

#### 策略 A：FTS 先过滤，向量精排（filter-then-rerank）

适用：FTS 选择性高（命中文档少）。

```
SCAN fts_index('数据库') → 拿到 100 个 docid
  → 精确算每个 doc 的 l2_distance
  → SORT + LIMIT 10
```

不需要走 ANN 索引——反正只有 100 行，brute-force 就行。

#### 策略 B：向量先 ANN，FTS 后过滤

适用：FTS 选择性低（命中很多）、向量索引选择性高（top-K 小）。

```
ANN_SCAN(vec, top=100) → 拿到 100 个 (vid, distance) 候选
  → 过滤 MATCH AGAINST '数据库'
  → 可能不够 10 行，迭代扩大 top-K
```

#### 策略 C：Filter 下推到 ANN 图遍历（最优）

适用：FTS 命中能做成 bitmap，且 vector index 支持 filter 下推。

```
FTS 扫一遍 → 把命中 docid 做成 roaring bitmap
  → 包成 ObHnswBitmapFilter 传给 vsag
  → vsag 在 HNSW 图遍历时跳过 bitmap 外的节点
  → 直接拿到 top-10 既满足 FTS 又向量近的结果
```

这就是 Part 5.1 讲过的 filter 下推。**这是 hybrid search 性能的关键**。

#### 选择性驱动

optimizer 怎么决定走哪个策略？基于**统计信息估算选择性**：

- FTS 命中估算（来自分词词典 + 文档统计）
- 向量索引的 ef/k 估算（基于配置）
- 表行数

策略选择写在 `src/sql/optimizer/` 里，具体函数要顺着 vector / fts 关键字 grep。

### 8.2 实验验证

```sql
DROP TABLE IF EXISTS hybrid_docs;
CREATE TABLE hybrid_docs (
  id INT PRIMARY KEY,
  content TEXT,
  embedding VECTOR(10),
  FULLTEXT INDEX idx_fts(content) WITH PARSER ik,
  VECTOR INDEX idx_vec(embedding) WITH (distance=l2, type=hnsw)
);

INSERT INTO hybrid_docs VALUES
  (1, '人工智能改变世界',          '[0.20,0.21,0.88,0.82,0.62,0.50,0.98,0.87,0.25,0.54]'),
  (2, '现代数据库支持向量检索',    '[0.74,0.67,0.90,0.45,0.23,0.66,0.77,0.23,0.58,0.93]'),
  (3, '语义搜索越来越流行',        '[0.33,0.05,0.08,0.39,0.97,0.37,0.18,0.94,0.01,0.63]'),
  (4, '数据库的索引技术',          '[0.55,0.66,0.77,0.88,0.99,0.11,0.22,0.33,0.44,0.55]');

CALL dbms_vector.refresh_index('idx_vec', 'hybrid_docs', 'embedding', 1, 'FAST');
```

**对比两种选择性场景：**

```sql
-- 高选择性 FTS（只命中 2 行）
EXPLAIN SELECT id FROM hybrid_docs
WHERE MATCH(content) AGAINST('数据库')
ORDER BY l2_distance(embedding, '[0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5]') APPROXIMATE LIMIT 10;

-- 低选择性 FTS（命中所有行）—— 通过插更多重复词测试
-- ...
```

观察 EXPLAIN 输出的算子组合差异。

**断点：**

```gdb
(gdb) break ObOptimizer::optimize
(gdb) break ObHnswBitmapFilter::test
(gdb) break ObVsagAdaptor::knn_search
(gdb) continue
```

- `ObOptimizer::optimize` 停下时，看 plan 树是怎么被规划的
- 如果走策略 C（filter 下推），`ObHnswBitmapFilter::test` 会被反复触发
- 如果走策略 A（filter-then-rerank），`ObVsagAdaptor::knn_search` 不会被触发

### 8.3 小结

1. Hybrid search 的三种执行策略？什么场景用哪个？
2. Filter 下推到 HNSW 图遍历是怎么做到的？关键抽象是什么？
3. optimizer 决策的依据是什么？

---

## Part 9 — Hybrid Vector Index：DB 内 embedding 的异步刷新

### 9.1 机制讲解

普通向量索引建在 `VECTOR(N)` 列上——你必须**先在客户端调 embedding 模型把文本转成向量**，再 INSERT。**Hybrid Vector Index** 把这一步搬到 DB 内：

```sql
-- 1. 注册 AI 模型
CALL DBMS_AI_SERVICE.CREATE_AI_MODEL('ob_embed', '{
  "type": "dense_embedding", "model_name": "bge-M3"
}');

CALL DBMS_AI_SERVICE.CREATE_AI_MODEL_ENDPOINT('ob_embed_endpoint', '{
  "ai_model_name": "ob_embed",
  "url": "https://your-embedding-service.com/v1",
  "access_key": "sk-xxx",
  "request_model_name": "bge-M3",
  "provider": "openai"
}');

-- 2. 在 VARCHAR 列上建 hybrid vector index
CREATE TABLE t_vec (
  id INT PRIMARY KEY,
  text VARCHAR(1024),
  VECTOR INDEX vec_idx(text) WITH (
    distance=l2,
    type=hnsw,
    model=ob_embed,
    dim=1024,
    sync_mode=immediate
  )
);

-- 3. 直接 INSERT 文本，DB 自动调 embedding 服务转成向量
INSERT INTO t_vec VALUES (1, '人工智能改变世界');
```

#### 为什么必须是 VARCHAR

`tools/deploy/mysql_test/test_suite/vector_index/t/create_table_with_hybrid_vector_index.test` 里有限制（`error 7601`）：不能用 `string`、`text(32)` 等。原因是 hybrid index 只支持有限定长度的字符串列（避免 embedding 输入过大）。

#### 异步刷新机制

`src/share/vector_index/ob_hybrid_vector_refresh_task.{h,cpp}` 是核心：

- INSERT 时不直接调 embedding 服务（会卡住客户端）
- 把"待 embedding 的行"写入 `embedded_table`（5 表里那个 `embedded_table_id`）
- 后台 task 周期性扫描，批量调 embedding 服务
- 拿到向量后写入 `inc_index_table`，再走正常的 refresh 合并流程

`sync_mode=immediate` vs `sync_mode=async` 的区别就在这——immediate 会同步等 embedding 完成，async 异步。

#### 失败处理

embedding 服务可能：
- 网络慢 / 超时
- 返回错误（比如 4016 model_name 错）
- rate limit

异步 task 框架（`ob_vector_index_async_task_util.cpp`）有重试 + 错误记录机制。

### 9.2 实验验证

需要一个真实的 embedding 服务（OpenAI compatible API），所以这一步可以选**只看代码不实跑**：

```gdb
(gdb) break ObHybridVectorRefreshTask::process
(gdb) break ObHybridVectorRefreshTask::call_embedding_service
```

如果有真实服务，按 9.1 的 SQL 跑一遍，能看到这两个断点被触发。

**只看代码也能学到**：
- 任务调度：周期 + 触发条件
- 批量打 embedding 服务（vs 一行一调）
- 失败重试策略
- 怎么把异步结果写回索引

### 9.3 小结

1. Hybrid vector index 解决什么痛点？（避免在客户端做 embedding，简化 RAG 工作流）
2. 为什么 embedding 必须异步？（同步会拖累 INSERT 性能）
3. 这个机制依赖哪些前置资源？（注册 AI 模型 + 可访问的 embedding 服务）

---

## Part 10 — 调试技巧汇总

### 10.1 不知道断点位置：三招找

```bash
# 1. grep 函数 / 类
grep -rn "VectorIndex\|knn_search\|vsag" src/share src/storage src/sql --include='*.cpp'

# 2. grep SQL 关键字（找 parser）
grep -rn "VECTOR INDEX\|APPROXIMATE\|FULLTEXT" src/sql/parser src/sql/resolver

# 3. gdb 模糊找
(gdb) info functions .*[Vv]ector.*[Ss]earch.*
(gdb) rbreak ^ObPluginVectorIndex.*::query$
```

### 10.2 trace_id 串联日志和断点

```sql
SELECT TRACE_ID FROM oceanbase.GV$OB_SQL_AUDIT ORDER BY REQUEST_TIME DESC LIMIT 1;
-- 假设输出：YB42AC1F010A-00064C2A...
```

```bash
grep "YB42AC1F010A-00064C2A" /tmp/obtest/single/log/observer.log
```

### 10.3 GV$OB_SQL_AUDIT 反向追溯

```sql
SELECT TRACE_ID, ELAPSED_TIME, ROWS_RETURNED, PLAN_ID, QUERY_SQL
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE QUERY_SQL LIKE '%vec%'
ORDER BY REQUEST_TIME DESC LIMIT 5;
```

### 10.4 silent 断点（自动打印不停下）

```gdb
break ObVsagAdaptor::knn_search
commands
  silent
  printf ">>> ANN: k=%d dim=%d\n", k, dim_
  bt 5
  continue
end
```

让你看到所有触发位置，但不打断执行。

### 10.5 看异步 task 状态

```sql
-- DDL 任务
SELECT * FROM oceanbase.__all_virtual_ddl_task_status\G

-- 向量索引 refresh 任务
SELECT * FROM oceanbase.DBA_OB_VECTOR_INDEXES;

-- 异步索引（hybrid）任务
SELECT * FROM oceanbase.GV$OB_VECTOR_INDEX_INFO\G
```

### 10.6 慎用

| 命令 | 风险 |
|------|------|
| `kill` (gdb) | 把 observer 杀了 |
| `set $rip = ...` | 改 PC，几乎一定崩 |
| `call some_function()` | 内核函数有线程上下文/锁假设，乱调一定崩 |

### 10.7 attach 后 client 卡住怎么办

90% 是断在某处忘了 continue：

```gdb
(gdb) c
```

随手按完。

---

## Part 11 — 7 天断点学习路线

每天 2-3 小时。每天结束后写 100 字笔记记录当天发现。

| 天 | 目标 | 必须验证的点 |
|----|------|-------------|
| **1** | Part 0-3：环境就绪 + 看清 SQL 整体骨架 | OMT、Plan Cache、Volcano 三件事各跑一次实验 |
| **2** | Part 4：CREATE INDEX 全链路 + 验证 5 表架构 | 查 `__all_table` 确认 5 张内部表，看 task state 流转 |
| **3** | Part 5：HNSW ANN 查询 + Filter 下推 | `ObHnswBitmapFilter::test` 触发；对比有无 APPROXIMATE 的 plan |
| **4** | Part 6：IVF + 对比 HNSW | 看 kmeans 聚类、cache 加载；对比 IVF/HNSW 的内存占用 |
| **5** | Part 7：FTS 全链路 | 中文分词的实际 token 列表；看 BM25 打分 |
| **6** | Part 8：Hybrid Search 的三种策略 | 构造高/低 selectivity 数据，观察 plan 切换 |
| **7** | Part 9：Hybrid Vector Index | 至少看代码理解异步 embedding 机制 |

跑完这一轮，你应该能：

- 给任意一条 vector / hybrid SQL，写出大致的执行计划
- 给定一个现象（比如"插完查很慢"），猜出大致原因（"增量没合并"）
- 看 EXPLAIN 输出能识别每个算子对应的代码模块
- 知道哪些是 OB 内核约束、哪些是 vsag 库限制、哪些是 hybrid 架构特定

---

## 附录 A — 关键代码地图

### 入口与协议
```
src/observer/main.cpp                            ← 进程入口
src/observer/ob_server.cpp                       ← Server 主类
src/observer/mysql/                              ← MySQL 协议
src/observer/mysql/obmp_query.{h,cpp}            ← COM_QUERY (ObMPQuery)
src/observer/omt/                                ← OMT 多租户管理
```

### SQL 栈
```
src/sql/ob_sql.cpp                               ← ObSql 主入口
src/sql/parser/                                  ← SQL parser (lex/yacc)
src/sql/resolver/                                ← 语义/DDL 解析
src/sql/optimizer/                               ← 物理计划优化
src/sql/code_generator/                          ← 算子代码生成
src/sql/engine/                                  ← 算子运行时
src/sql/plan_cache/                              ← Plan Cache
```

### 向量索引
```
deps/oblib/src/lib/vector/
  ob_vsag_adaptor.{h,cpp}                        ← vsag 库封装（HNSW/HNSW_SQ/HNSW_BQ/HGRAPH）
  ob_vector_util.{h,cpp}                         ← 距离函数（l2/cosine/ip）

src/share/vector_index/
  ob_plugin_vector_index_adaptor.{h,cpp}         ← 插件统一接口
                                                   含 ObVectorIndexInfo (5 表架构定义)
                                                   含 ObHnswBitmapFilter (filter 下推)
  ob_plugin_vector_index_service.{h,cpp}         ← 索引服务 / 内存管理
  ob_vector_kmeans_ctx.{h,cpp}                   ← IVF k-means 聚类
  ob_vector_index_ivf_cache_mgr.{h,cpp}          ← IVF centroids 缓存
  ob_vector_index_param.{h,cpp}                  ← WITH(...) 参数解析
                                                   含 ObVectorIndexQueryParam (运行时参数)
  ob_hybrid_vector_refresh_task.{h,cpp}          ← Hybrid Index 异步 embedding
  ob_vector_index_async_task_util.{h,cpp}        ← 异步任务框架

src/storage/access/
  ob_vector_store.{h,cpp}                        ← 向量列存储访问层

src/storage/vector_index/                        ← refresh 调度胶水

src/rootserver/ddl_task/
  ob_vec_index_build_task.cpp                    ← CREATE INDEX DDL 任务
  ob_drop_vec_index_task.{h,cpp}                 ← DROP INDEX
  ob_rebuild_index_task.cpp                      ← REBUILD
```

### 全文索引
```
src/storage/fts/
  ob_*_ft_parser.{h,cpp}                         ← 分词器：whitespace/ngram/ngram2/ik/beng
  ob_fts_doc_word_iterator.{h,cpp}               ← 文档-词迭代
  ob_fts_struct.h                                ← 核心数据结构
  ob_fts_plugin_helper.{h,cpp}                   ← FTS 插件桥
  dict/                                          ← 词典加载
  ik/                                            ← IK 中文分词实现
                                                   ob_ik_arbitrator (歧义仲裁)
                                                   ob_ik_cjk_processor
                                                   ob_ik_quantifier_processor
```

### Hybrid 调度
```
src/sql/optimizer/                               ← 决定要不要 hybrid plan
src/sql/engine/                                  ← 算子组合
src/share/vector_index/ob_hybrid_vector_refresh_task.cpp  ← Hybrid Vector Index 异步刷新
                                                            （注意：不是 hybrid search）
```

---

## 附录 B — 自助找断点位置的方法

### B.1 从 EXPLAIN 算子名找

```sql
EXPLAIN <你的 SQL>;
```

记下算子名（如 `PHY_VEC_IDX_SCAN`）：

```bash
grep -rn "PHY_VEC_IDX_SCAN" src/sql/engine/ | head
```

通常会找到对应的算子类（如 `ObVectorIndexScan`），断在它的 `inner_get_next_row` / `open` / `close`。

### B.2 从 SQL 关键字找 parser

```bash
grep -rni "approximate" src/sql/parser/ src/sql/resolver/ | head
```

通常会找到 `.l` / `.y` 文件 / 或 resolver 里的关键字判断。

### B.3 从虚拟表找服务实现

```sql
SELECT * FROM oceanbase.GV$OB_VECTOR_INDEX_INFO\G
```

```bash
grep -rn "GV\$OB_VECTOR_INDEX_INFO\|all_virtual_vector_index_info" src/observer/virtual_table/ src/share/ | head
```

### B.4 gdb 模糊找

```gdb
(gdb) info functions .*[Hh]ybrid.*
(gdb) info functions oceanbase::share::ObPluginVectorIndex.*
(gdb) rbreak ^.*::knn_search$
```

### B.5 mysqltest case 反推

`tools/deploy/mysql_test/test_suite/vector_index/t/*.test` 有 22 个可工作的最小复现，挑一个跟你想理解的功能最像的，跑一遍：

```bash
cd tools/deploy
./obd.sh mysqltest -n single \
  --test-dir ./mysql_test/test_suite/vector_index/t \
  --result-dir ./mysql_test/test_suite/vector_index/r \
  --test-set vector_index_basic
```

gdb 蹲在你怀疑的位置看是否触发。

---

## 一个最后的提醒

- **断点一停，整个 observer 就冻结**——客户端会"卡住"，记得 `c` 继续
- **不要在生产实例上调试**——会影响所有用户
- **看不懂某段代码时**：找对应的 mysqltest case 跑一遍，对照 `.result` 文件理解预期行为，比硬读源码快得多
- **追异步路径时**：log + silent 断点比手动 attach 更省心
