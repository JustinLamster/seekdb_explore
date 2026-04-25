# seekdb 向量 / 混合检索：跑起来 + 断点追数据流

> 这份文档面向"想真正搞清 seekdb 向量索引和混合检索是怎么跑的"的开发者。
>
> **核心方法**：把 observer 跑起来 → 在关键函数下断点 → 发一条 SQL → 看调用栈 / 变量 / 控制流。
>
> **目标读者**：在 Linux 服务器上开发，会用 gdb，至少读过一些 C++ 项目源码。
>
> **不会教**：gdb 基础命令（自行 `apropos gdb`），SQL 基础。

---

## 目录

- [Part 0 — 概念厘清](#part-0--概念厘清)
- [Part 1 — 环境准备（Linux 服务器）](#part-1--环境准备linux-服务器)
- [Part 2 — Debug 准备：让 observer 可被 attach](#part-2--debug-准备让-observer-可被-attach)
- [Part 3 — 数据流追踪 ①：一条 SQL 的生命周期](#part-3--数据流追踪-一条-sql-的生命周期)
- [Part 4 — 数据流追踪 ②：CREATE VECTOR INDEX](#part-4--数据流追踪-create-vector-index)
- [Part 5 — 数据流追踪 ③：HNSW ANN 查询](#part-5--数据流追踪-hnsw-ann-查询)
- [Part 6 — 数据流追踪 ④：FTS 查询](#part-6--数据流追踪-fts-查询)
- [Part 7 — 数据流追踪 ⑤：Hybrid Search](#part-7--数据流追踪-hybrid-search)
- [Part 8 — 调试技巧汇总](#part-8--调试技巧汇总)
- [Part 9 — 7 天断点学习路线](#part-9--7-天断点学习路线)
- [附录 A — 关键代码地图](#附录-a--关键代码地图)
- [附录 B — 自助找断点位置的方法](#附录-b--自助找断点位置的方法)

---

## Part 0 — 概念厘清

seekdb 里 "hybrid" 这个词被用在**两个不同的东西**上，不分清楚后面看断点会迷路：

| 术语 | 含义 | DDL 触发 |
|------|------|----------|
| **Hybrid Vector Index** | 索引建在**文本列**上，DB 内部自动调 embedding 模型把文本转向量再建索引 | `VECTOR INDEX(text_col) WITH(model=ob_embed, ...)` + `DBMS_AI_SERVICE.CREATE_AI_MODEL(...)` |
| **Hybrid Search** | 同一条 SQL 里**同时用向量距离 + 全文打分** | `WHERE MATCH(c) AGAINST(...) ORDER BY l2_distance(...) APPROXIMATE LIMIT k` |

两者**互相独立**：hybrid search 的 SQL 既可以查普通 vector 索引（你预先算好 embedding），也可以查 hybrid vector index。本文档先聚焦后者（hybrid search），前者（hybrid vector index）放最后。

---

## Part 1 — 环境准备（Linux 服务器）

### 1.1 Debug 构建（必须）

> 目标：编译产物带 `-g -O0`、保留符号、保留帧指针，让 gdb 看得到所有变量。

```bash
cd ~/seekdb_explore                            # 替换为你的实际路径

# 第一次：装依赖（从 deps/3rd/ 下载，几分钟到几十分钟）
bash build.sh debug --init --make -j$(nproc)

# 增量：
bash build.sh debug --make -j$(nproc)
```

**验证 debug 符号在位**：

```bash
file build_debug/src/observer/seekdb            # 应该看到 "with debug_info, not stripped"
nm build_debug/src/observer/seekdb | head -5    # 有符号
```

### 1.2 部署单节点（最小拓扑）

```bash
cd tools/deploy
./obd.sh prepare -p /tmp/obtest                # 准备空数据目录
./obd.sh deploy -c ./single.yaml               # 起单节点

# 验证起来了
ps aux | grep -v grep | grep observer          # 看到 observer 进程
ss -tlnp | grep 10000                          # MySQL 端口（默认 10000）

# 出问题就毁了重来
./obd.sh destroy --rm -n single
```

> **注意**：`obd.sh` 启动的 observer 是 **release 优化版**还是你刚 build 的 debug 版？看 `single.yaml` 里 `home_path` 指向哪。如果想确保跑的是 debug build，**手动覆盖二进制**：
>
> ```bash
> cp build_debug/src/observer/seekdb /tmp/obtest/single/bin/seekdb
> # 然后重启 observer
> ```

### 1.3 连接 + 系统设置

```bash
mysql -uroot -h127.0.0.1 -P10000

# 连上后：
USE oceanbase;
ALTER SYSTEM SET ob_vector_memory_limit_percentage = 30;     -- 给向量索引留 30% 内存
ALTER SYSTEM SET enable_sql_audit = true;                    -- 打开 sql audit，方便回溯
ALTER SYSTEM SET syslog_level = 'INFO';                      -- 想更详细就 'DEBUG'，但日志会爆涨

CREATE DATABASE vec_test;
USE vec_test;
```

---

## Part 2 — Debug 准备：让 observer 可被 attach

### 2.1 找到 observer 进程

```bash
pgrep -a observer
# 或
ps -ef | grep observer | grep -v grep
```

记下 PID，下面记作 `$OBPID`。

### 2.2 关掉看门狗 / 健康检查（推荐）

observer 默认有自检线程，进程被 gdb 暂停几秒就可能被认为"卡死"而触发自杀或副本切主。在 debug 时建议放宽：

```sql
-- 系统租户里执行
ALTER SYSTEM SET _enable_check_diagnose_info = false;
ALTER SYSTEM SET enable_perf_event = false;
-- 调大 RPC 超时
ALTER SYSTEM SET rpc_timeout = 3600000000;                   -- 1 小时
```

### 2.3 attach gdb（命令行方式）

```bash
sudo gdb -p $OBPID
```

attach 上之后第一件事**别急着 continue**：

```gdb
(gdb) set pagination off
(gdb) set print pretty on
(gdb) set print thread-events off
(gdb) handle SIGPIPE nostop noprint              # observer 大量用，别打扰
(gdb) handle SIG34 SIG35 SIG36 nostop noprint    # OB 内部用的实时信号
```

> **⚠️ 重要**：attach 时整个 observer 进程会被冻结（所有线程都停）。如果有别的客户端连接也会被卡住。**调试用的 observer 一定要专用一台**，不要 attach 生产实例。

### 2.4 attach gdb（VS Code Remote SSH 方式）

如果你想在 IDE 里看：

1. VS Code 装 `Remote - SSH` + `C/C++` 扩展
2. SSH 进开发机
3. 打开 `~/seekdb_explore/` 文件夹
4. clangd 配置：把 `build_debug/compile_commands.json` 链到根：
   ```bash
   ln -sf build_debug/compile_commands.json compile_commands.json
   ```
5. 在仓库根新建 `.vscode/launch.json`：

```jsonc
{
  "version": "0.2.0",
  "configurations": [
    {
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
    }
  ]
}
```

按 F5 → 选 observer 进程 → 在编辑器里点行号边设断点。

### 2.5 调高日志等级（断点的重要补充）

很多内核路径走的是 task / 异步线程，不一定能蹲到。打开 DEBUG 日志可以补全：

```sql
-- 全局
ALTER SYSTEM SET syslog_level = 'DEBUG';

-- 只看某个模块
ALTER SYSTEM SET syslog_level = 'INFO,STORAGE.VECTOR:DEBUG';
```

日志位置：`/tmp/obtest/single/log/observer.log`、`election.log`、`rootservice.log`。

```bash
# 实时跟单条 SQL 的所有日志
tail -F /tmp/obtest/single/log/observer.log | grep -i 'trace_id=YXxxxx'
```

`trace_id` 怎么拿：`SELECT TRACE_ID FROM oceanbase.GV$OB_SQL_AUDIT ORDER BY REQUEST_TIME DESC LIMIT 1;`

---

## Part 3 — 数据流追踪 ①：一条 SQL 的生命周期

> **目的**：在追任何向量/FTS 路径之前，先建立"任意 SQL 从网络进来到执行完返回"的整体感觉。

### 3.1 实验 SQL

```sql
USE vec_test;
SELECT 1;                    -- 故意用最简单的，先让你看清楚整个骨架
```

### 3.2 推荐断点（按调用顺序）

| # | 断点位置 | 作用 |
|---|---------|------|
| 1 | `ObMPQuery::process` | MySQL COM_QUERY 命令进来的入口 |
| 2 | `ObSql::stmt_query` | SQL 主入口（parse + resolve + plan + execute 都从这里分发） |
| 3 | `ObSql::generate_physical_plan` | 物理计划生成（plan cache miss 时） |
| 4 | `ObPlanCache::get_plan` | 查计划缓存（plan cache hit 时不走 generate） |
| 5 | `ObResultSet::open` | 执行入口（实际开始 open 算子树） |
| 6 | `ObOperator::get_next_row` | 通用算子驱动函数（每个算子的 next 都从基类进） |

### 3.3 操作步骤

```gdb
(gdb) break ObMPQuery::process
(gdb) break ObSql::stmt_query
(gdb) break ObResultSet::open
(gdb) continue
```

切到 mysql client 发 `SELECT 1;`，gdb 会停在 `ObMPQuery::process`。

```gdb
(gdb) bt 30                    # 看上层调用栈：网络层 → RPC dispatch → MySQL 协议
(gdb) p sql_                   # 看到本次 SQL 字符串
(gdb) finish                   # 执行完当前函数，停在调用者
(gdb) continue                 # 跳到下一个断点
```

### 3.4 你应该看到的"骨架"

```
观察者主线程 / worker
  └─ MySQL 协议解析 (src/observer/mysql/)
      └─ ObMPQuery::process
          └─ ObSql::stmt_query
              ├─ Parser  (src/sql/parser/)
              ├─ Resolver (src/sql/resolver/)
              ├─ Optimizer (src/sql/optimizer/)
              ├─ Code Generator (src/sql/code_generator/)
              └─ ObResultSet::open / get_next_row
                  └─ ObOperator::get_next_row (递归驱动算子树)
```

把这个图刻在脑子里，下面追任何模块都是在这个骨架上 zoom in 到某个算子。

---

## Part 4 — 数据流追踪 ②：CREATE VECTOR INDEX

> **目的**：搞清楚一条 DDL 怎么落到后台 build 任务，最终生成 HNSW 索引文件。

### 4.1 实验 SQL

```sql
USE vec_test;
DROP TABLE IF EXISTS docs;
CREATE TABLE docs (
  id INT PRIMARY KEY,
  embedding VECTOR(10)
);

-- 关键这一句：
CREATE VECTOR INDEX idx_vec ON docs(embedding) WITH (distance=l2, type=hnsw, lib=vsag);
```

### 4.2 推荐断点

| # | 断点位置 | 文件 | 作用 |
|---|---------|------|------|
| 1 | `ObCreateIndexResolver::resolve` | `src/sql/resolver/ddl/` | DDL 语法解析 |
| 2 | `ObDDLService::create_index_table` | `src/rootserver/ob_ddl_service.cpp` | RootServer 收到 DDL 请求 |
| 3 | `ObVecIndexBuildTask::process` | `src/rootserver/ddl_task/ob_vec_index_build_task.cpp` | 异步构建任务的状态机 |
| 4 | `ObPluginVectorIndexAdaptor::init` | `src/share/vector_index/ob_plugin_vector_index_adaptor.cpp` | 索引插件初始化 |
| 5 | `ObVsagAdaptor::build_index` | `deps/oblib/src/lib/vector/ob_vsag_adaptor.cpp` | **真正调用 vsag 库构建 HNSW** |

### 4.3 操作步骤

```gdb
(gdb) break ObVecIndexBuildTask::process
(gdb) break ObVsagAdaptor::build_index
(gdb) continue
```

提交 CREATE INDEX 后，因为是异步任务，可能需要 **几秒到几十秒** 才会触发断点。这个过程能让你看到：

- DDL 是 task driver 驱动的状态机（不是同步执行）
- HNSW 索引的真实构建发生在 `ObVsagAdaptor::build_index`，里面调用 vsag 第三方库

```gdb
# 在 ObVecIndexBuildTask::process 里
(gdb) p task_status_                # 当前状态
(gdb) p tenant_id_, task_id_, table_id_, index_id_

# step 几次直到进入 vsag 调用
(gdb) step

# 在 ObVsagAdaptor::build_index 里
(gdb) p dim_, max_elements_, ef_construction_, M_
(gdb) bt                            # 看从 DDL 任务怎么一路调到 vsag
```

### 4.4 验证结果

```sql
SHOW INDEX FROM docs;
SELECT * FROM oceanbase.GV$OB_VECTOR_INDEX_INFO\G
SELECT * FROM oceanbase.DBA_OB_VECTOR_INDEXES;
```

---

## Part 5 — 数据流追踪 ③：HNSW ANN 查询

> **目的**：跟踪一次 ANN 查询从 SQL 到 vsag::Index::knn_search 的完整链路。

### 5.1 准备数据

```sql
INSERT INTO docs VALUES
  (1, '[0.20,0.21,0.88,0.82,0.62,0.50,0.98,0.87,0.25,0.54]'),
  (2, '[0.74,0.67,0.90,0.45,0.23,0.66,0.77,0.23,0.58,0.93]'),
  (3, '[0.33,0.05,0.08,0.39,0.97,0.37,0.18,0.94,0.01,0.63]'),
  (4, '[0.14,0.87,0.02,0.32,0.04,0.14,0.71,0.44,0.63,0.63]'),
  (5, '[0.33,0.85,0.88,0.66,0.98,0.41,0.20,0.19,0.95,0.79]');
```

### 5.2 实验 SQL

```sql
-- 先看 plan：
EXPLAIN
SELECT id FROM docs
ORDER BY l2_distance(embedding, '[0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5]') APPROXIMATE
LIMIT 3;
```

`EXPLAIN` 输出里应该看到 **`VEC INDEX SCAN`** 或类似算子（确切名字看输出，不同版本可能叫 `TABLE SCAN APPROXIMATE` / `ANN SCAN`）。**记下这个算子名**，下面好定位代码。

### 5.3 找到 ANN 算子的代码位置

```bash
# 假设上一步看到算子叫 PHY_VEC_IDX_SCAN
grep -rn "PHY_VEC_IDX_SCAN\|VEC_IDX_SCAN\|ObVectorIndexScan" src/sql/engine/ | head
```

或者直接 gdb：

```gdb
(gdb) info functions Vector.*Scan.*get_next
```

### 5.4 推荐断点

| # | 断点位置 | 作用 |
|---|---------|------|
| 1 | `ObSql::stmt_query` | 入口（SQL 字符串到这里） |
| 2 | (上面 grep 出的 ANN 算子)::`inner_get_next_row` | ANN 算子驱动 |
| 3 | `ObPluginVectorIndexAdaptor::query` 或 `::knn_search` | 索引插件层入口 |
| 4 | `ObVsagAdaptor::knn_search` | vsag 调用入口（真正的 HNSW 搜索） |
| 5 | `ObVectorStore::*` | 取 raw 向量做精排（如果有 rerank 阶段） |

### 5.5 操作步骤

```gdb
(gdb) break ObVsagAdaptor::knn_search
(gdb) continue
```

发 SELECT，gdb 停下后：

```gdb
(gdb) bt 25                          # 看从 ObResultSet → 算子 → 插件 → vsag 的完整栈
(gdb) p k                            # top-k 值，应该是 3
(gdb) p dim_                         # 维度，应该是 10
(gdb) p ef_search_                   # 搜索参数（影响召回率）
(gdb) finish                         # 看 vsag 返回了什么
```

### 5.6 你应该看到的关键事实

- **`APPROXIMATE` 是触发向量索引的开关**——去掉这个关键词，断点根本不会停（走的是精确 brute-force scan）
- ANN 算子拿到的是 vsag 返回的 `(rowid, distance)` list
- 如果 SELECT 列表里有非索引列（如 `id`），算子还要回表取 → 看 `ObVectorStore` 相关断点

### 5.7 顺带：试 IVF 系列

```sql
DROP TABLE IF EXISTS docs_ivf;
CREATE TABLE docs_ivf (id INT PRIMARY KEY, embedding VECTOR(10));
CREATE VECTOR INDEX idx_ivf ON docs_ivf(embedding) WITH (distance=l2, type=ivf_flat, lists=10);
```

断点改成 `ObVectorIndexIvfCacheMgr::*`、`ObVectorKmeansCtx::*`，能看到 IVF 的聚类逻辑。

---

## Part 6 — 数据流追踪 ④：FTS 查询

### 6.1 实验 SQL

```sql
DROP TABLE IF EXISTS articles;
CREATE TABLE articles (
  id INT PRIMARY KEY,
  content TEXT,
  FULLTEXT INDEX idx_fts(content) WITH PARSER ik          -- 中文用 ik，英文用 whitespace/ngram
);

INSERT INTO articles VALUES
  (1, '机器学习是人工智能的一个重要分支'),
  (2, '现代数据库提供高性能存储与查询能力'),
  (3, '向量数据库支持语义级别的相似搜索');

EXPLAIN
SELECT id, MATCH(content) AGAINST('数据库') AS score
FROM articles WHERE MATCH(content) AGAINST('数据库') ORDER BY score DESC;
```

### 6.2 推荐断点

| # | 位置 | 作用 |
|---|------|------|
| 1 | `ObIKFTParser::*` 或 `ObNgramFTParser::*` | 分词器入口（看 query 字符串如何被切词） |
| 2 | `ObFTPluginHelper::*` | FTS 插件桥 |
| 3 | (FTS 扫描算子, grep 找：`grep -rn "fts_index_scan\|ObFtsIndex" src/sql/engine/`) | 倒排扫描算子 |
| 4 | `ObFTSDocWordIterator::*` | 文档-词迭代器 |

```gdb
(gdb) break ObIKFTParser::segment
(gdb) continue
```

发 SELECT，断在分词器：

```gdb
(gdb) p text_                        # query 文本
(gdb) finish
(gdb) p tokens                       # 切词结果
```

> 中文分词调试很爽：能直接看到 "数据库" 被切成什么 token，然后这些 token 怎么去倒排索引取 docid list。

---

## Part 7 — 数据流追踪 ⑤：Hybrid Search

### 7.1 实验 SQL

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
  (3, '语义搜索越来越流行',        '[0.33,0.05,0.08,0.39,0.97,0.37,0.18,0.94,0.01,0.63]');

-- 这就是 hybrid search：
EXPLAIN
SELECT
  id,
  l2_distance(embedding, '[0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5,0.5]') AS vec_dist,
  MATCH(content) AGAINST('数据库') AS text_score
FROM hybrid_docs
WHERE MATCH(content) AGAINST('数据库')
ORDER BY vec_dist APPROXIMATE
LIMIT 5;
```

### 7.2 关键观察点

`EXPLAIN` 应该看到两个扫描算子（FTS scan + Vec scan）通过某种 **JOIN / 交集 / Top-K** 组合起来。**这一步是 hybrid 的精髓**，不同 seekdb 版本可能用不同算子组合：

- 选择性低的优先：先 FTS 过滤拿 docid 集合 → 再精确算 vec distance 排序
- 选择性高的优先：先 ANN 取 top-N candidates → 再 FTS 过滤
- 真正的 fusion：两路并行各取 top-N → reciprocal rank fusion / weighted

**你需要在 `EXPLAIN` 里看清是哪种**，再定位代码：

```bash
# 假设 EXPLAIN 里看到 ObTableScanWithVecIndex + ObTableScanWithFtsIndex + 上层 ObTopK
grep -rn "VecIndex\|FtsIndex\|Topk.*Vec\|Hybrid.*Scan" src/sql/engine/ | head
```

### 7.3 推荐断点

| # | 位置 | 作用 |
|---|------|------|
| 1 | `ObSelectResolver::*` 或 `ObOptimizer::optimize` | 看 plan 是怎么被规划成两个扫描的 |
| 2 | (上面 grep 出的 hybrid 调度算子) | 融合层入口 |
| 3 | `ObVsagAdaptor::knn_search` | 看是先扫向量还是先扫 FTS |
| 4 | `ObIKFTParser::segment` | 看是先扫 FTS 还是先扫向量 |
| 5 | (FTS scan 算子 + Vec scan 算子的 `inner_get_next_row`) | 看两个算子的输出怎么 merge |

### 7.4 实验：调换 selectivity

```sql
-- 极窄过滤：只 1 个文档命中 FTS
SELECT id FROM hybrid_docs
WHERE MATCH(content) AGAINST('改变')
ORDER BY l2_distance(embedding, '[0.5,...]') APPROXIMATE LIMIT 5;

-- 宽过滤：10000 个文档全命中
-- (需要先插大量数据)
```

观察 `EXPLAIN` 和断点触发顺序怎么变。优化器会根据估算的选择性切换执行策略。

---

## Part 8 — 调试技巧汇总

### 8.1 找断点位置的快速方法

不知道某个功能在哪？三招：

```bash
# 1. grep 函数名 / 类名
grep -rn "VectorIndex\|knn_search\|vsag" src/share src/storage src/sql --include='*.cpp' --include='*.h'

# 2. grep SQL 关键字（找 parser）
grep -rn "VECTOR INDEX\|APPROXIMATE\|FULLTEXT" src/sql/parser src/sql/resolver

# 3. gdb 里反向找
(gdb) info functions .*[Vv]ector.*[Ss]earch.*
(gdb) rbreak ^ObVectorIndex.*
```

### 8.2 trace_id 串联日志和断点

每条 SQL 在 observer 里都有唯一 `trace_id`。用 trace_id 把 gdb 断点期间的所有日志拉出来：

```sql
-- 客户端
SELECT TRACE_ID FROM oceanbase.GV$OB_SQL_AUDIT ORDER BY REQUEST_TIME DESC LIMIT 1;
-- 假设输出：YB42AC1F010A-00064C2A...
```

```bash
grep "YB42AC1F010A-00064C2A" /tmp/obtest/single/log/observer.log
```

### 8.3 GV$OB_SQL_AUDIT 是反向追溯神器

```sql
SELECT
  TRACE_ID, ELAPSED_TIME, ROWS_RETURNED, PLAN_ID, QUERY_SQL
FROM oceanbase.GV$OB_SQL_AUDIT
WHERE QUERY_SQL LIKE '%vec%'
ORDER BY REQUEST_TIME DESC
LIMIT 5;
```

执行过哪些 SQL、走了哪个 plan、耗时多少、返回多少行——一目了然。

### 8.4 一次 attach 多个断点

写一个 `.gdbinit-vec`：

```gdb
break ObVsagAdaptor::build_index
break ObVsagAdaptor::knn_search
break ObPluginVectorIndexAdaptor::query
commands
  silent
  printf ">>> ANN query: k=%d dim=%d\n", k, dim_
  bt 5
  continue
end
```

attach 时：

```bash
sudo gdb -p $OBPID -x .gdbinit-vec
```

`commands ... continue` 让断点**自动打印调用栈但不停下**——既不打断执行，又能看到所有触发位置。

### 8.5 慎用的命令

| 命令 | 风险 |
|------|------|
| `kill` (在 gdb 里) | 把 observer 杀了 |
| `set $rip = ...` | 改 PC，几乎一定崩 |
| `call some_function()` | observer 里很多函数有线程上下文/锁假设，乱调一定崩 |
| `signal SIGKILL` | 同上 |

---

## Part 9 — 7 天断点学习路线

每天 2-3 小时。每天结束后写 100 字笔记记录当天的发现。

| 天 | 目标 | 必须看到的断点 |
|----|------|---------------|
| **1** | 跑通环境 + 看清 SQL 整体骨架 | `ObMPQuery::process` → `ObSql::stmt_query` → `ObResultSet::open` |
| **2** | CREATE VECTOR INDEX 的 DDL 全链路 | `ObVecIndexBuildTask::process` → `ObVsagAdaptor::build_index` |
| **3** | HNSW ANN 查询全链路 | (ANN 算子) → `ObPluginVectorIndexAdaptor::query` → `ObVsagAdaptor::knn_search` |
| **4** | IVF 系列 + 对比 HNSW | `ObVectorKmeansCtx::*` + `ObVectorIndexIvfCacheMgr::*` |
| **5** | FTS 全链路 | `ObIKFTParser::segment` → FTS scan 算子 → `ObFTSDocWordIterator::*` |
| **6** | Hybrid Search 的 plan + 执行 | `ObOptimizer::optimize`（看 plan）+ FTS/Vec 两个 scan 算子的交错触发 |
| **7** | Hybrid Vector Index（DB 内 embedding） | `ObHybridVectorRefreshTask::*` + 看 embedding 模型怎么被异步调用 |

---

## 附录 A — 关键代码地图

按"看到 SQL 后能猜到代码在哪"组织。

### 入口与协议
```
src/observer/main.cpp                    ← 进程入口
src/observer/ob_server.cpp               ← Server 主类
src/observer/mysql/                      ← MySQL 协议
src/observer/mysql/obmp_query.{h,cpp}    ← COM_QUERY 处理 (ObMPQuery)
```

### SQL 栈
```
src/sql/ob_sql.cpp                       ← ObSql 主入口
src/sql/parser/                          ← SQL parser (lex/yacc 生成)
src/sql/resolver/                        ← 语义解析 / DDL 解析
src/sql/optimizer/                       ← 物理 plan 优化
src/sql/code_generator/                  ← 算子生成
src/sql/engine/                          ← 算子运行时（grep 找具体算子）
```

### 向量索引
```
deps/oblib/src/lib/vector/ob_vsag_adaptor.cpp     ← vsag 库封装（HNSW 真正实现）
deps/oblib/src/lib/vector/ob_vector_util.cpp      ← 距离函数 l2/cosine/ip

src/share/vector_index/
  ob_plugin_vector_index_adaptor.cpp              ← 插件统一接口（HNSW/IVF 都走）
  ob_plugin_vector_index_service.cpp              ← 索引服务 / 内存
  ob_vector_kmeans_ctx.cpp                        ← IVF 聚类
  ob_vector_index_ivf_cache_mgr.cpp               ← IVF 缓存管理
  ob_vector_index_param.cpp                       ← WITH(...) 参数解析
  ob_hybrid_vector_refresh_task.cpp               ← Hybrid Vector Index 异步 embedding

src/storage/access/ob_vector_store.cpp            ← 向量列存储访问层
src/storage/vector_index/                         ← 调度/refresh 胶水

src/rootserver/ddl_task/
  ob_vec_index_build_task.cpp                     ← CREATE INDEX DDL 任务
  ob_drop_vec_index_task.{h,cpp}                  ← DROP INDEX
  ob_rebuild_index_task.cpp                       ← REBUILD
```

### 全文索引
```
src/storage/fts/
  ob_*_ft_parser.{h,cpp}                          ← 分词器：whitespace/ngram/ngram2/ik/beng
  ob_fts_doc_word_iterator.{h,cpp}                ← 文档-词迭代
  ob_fts_struct.h                                 ← 核心数据结构
  dict/                                           ← 词典加载
  ik/                                             ← IK 中文分词实现
```

### Hybrid 调度
```
src/sql/optimizer/                                ← 决定要不要 hybrid plan
src/sql/engine/                                   ← 算子组合（具体算子名 grep 找）
src/share/vector_index/ob_hybrid_vector_refresh_task.cpp  ← Hybrid Vector Index（不是 hybrid search）
```

---

## 附录 B — 自助找断点位置的方法

文档列的断点不可能涵盖你想看的所有点。掌握下面这些，你能自己找：

### B.1 通过 EXPLAIN 找算子

```sql
EXPLAIN <你的 SQL>;
```

记下输出里的算子名（如 `PHY_VEC_IDX_SCAN`），grep：

```bash
grep -rn "PHY_VEC_IDX_SCAN" src/sql/engine/ | head
```

通常会找到对应的算子类（如 `ObVectorIndexScan`），断在它的 `inner_get_next_row` / `open` / `close`。

### B.2 通过 SQL 关键字找 parser

```sql
-- 想知道 APPROXIMATE 这个关键字是哪里识别的
```

```bash
grep -rni "approximate" src/sql/parser/ src/sql/resolver/ | head
```

通常会找到 `.l` 词法文件 / `.y` 文法文件 / 或 resolver 里的关键字判断。

### B.3 通过虚拟表找服务实现

```sql
SELECT * FROM oceanbase.GV$OB_VECTOR_INDEX_INFO\G
```

```bash
grep -rn "GV\$OB_VECTOR_INDEX_INFO\|all_virtual_vector_index_info" src/observer/virtual_table/ src/share/ | head
```

虚拟表后面通常挂的是真实服务模块的查询接口，能反向找到模块入口。

### B.4 gdb 模糊找

```gdb
(gdb) info functions .*[Hh]ybrid.*
(gdb) info functions oceanbase::share::ObPluginVectorIndex.*
(gdb) rbreak ^.*::knn_search$
```

### B.5 通过 mysqltest case 反推

`tools/deploy/mysql_test/test_suite/vector_index/t/*.test` 里有 22 个 case，每个 case 都是一个**可工作的最小复现**。挑一个跟你想理解的功能最像的，跑一遍，gdb 蹲在你怀疑的位置看是否触发。

---

## 一个最后的提醒

**断点一停，整个 observer 就冻结**。如果你的 mysql client 看起来"卡住没反应"，多半是断在某个内核函数没 continue。养成习惯：

```gdb
(gdb) c              # continue 的快捷
```

随手按完。
