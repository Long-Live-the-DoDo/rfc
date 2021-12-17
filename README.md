# 高质量 Flashback RFC

## 背景

TiDB 处理各种灾难故障可谓轻车熟路，但是常言道“天灾易躲，人祸难防”，对于各种误操作、bug 写入错误数据、甚至删库跑路，目前还没什么招。

因为 TiDB 的事务是基于 MVCC 的，所以一段时间内的旧版本都在，理论上对于上述人祸，都是可以进行手动恢复的。但是现有的功能和计划中的功能都太弱了：

- 很可能需要排查数据损坏的情况，目前只能指定 ts 去读一个时间点的数据，要查看某条记录的变化历史太麻烦
- recover table 只能恢复 drop/truncate 这种 ddl 操作，对 dml 没招
- GC safepoint 之前的数据恢复不了，如果想保留长时间的数据，又太费空间了
- 恢复数据要先把数据 dump 出来再重新写入，太慢了

## 构想

充分利用 MVCC 特性，加强 MVCC 数据的查询、整理、恢复的能力，提高问题处理的效率。MVCC 不只是可以用来暂时性地处理事务隔离，也完全可以做为冷备，相比于外部的备份，其优点是可以更省空间，恢复数据也更方便更快。

具体的我们可以：

- 增加虚拟列 `_tidb_mvcc_ts`，`_tidb_mvcc_op`，可以使用 SQL 直接查询一条记录的多个版本，方便排查问题，也可以直接运行 select into 还原数据，不用导出再导入
- 支持不同的表设置不同的 gc lifetime
- 允许定期保留版本，例如每天一个版本，GC 的时候保留每天的最后一个版本，既不会因为版本太多占用很多空间，又可以用来还原数据
- 基于 schema 亚秒级 flashback，记录要删除的数据的 ts 范围，TiDB 下发给 TiKV 后在 mvcc 层进行过滤

## 具体实施方案

### 1. MVCC Query in SQL

- 参考 `_tidb_rowid` 的实现，增加 `_tidb_mvcc_ts`，`_tidb_mvcc_op` 虚拟列。
- 当查询虚拟列时，TiDB 发送给 TiKV 的请求中要带上标记，指明要查询 MVCC 虚拟列。
- 修改 TiKV 的 MVCC 读取逻辑，当需要查询虚拟列时，需要扫描所有版本，而不是只扫描最新版本。然后设置每条数据对应的虚拟列值。`_tidb_mvcc_ts` 为事务的 `commit_ts`，`_tidb_mvcc_op` 为事务的操作类型，可以是 `PUT` 或 `DELETE`。

验收效果：

- 带上虚拟列查询，可以正常显示所有的 MVCC 版本。
- 可以在查询条件中添加针对 ts 和 op 的过滤条件，只显示符合条件的版本。
- 可以使用子查询的方式还原某一条数据。
- 可以使用 `SELECT INTO` 将数据转存到另一表。

### 2. GC Savepoint

- 添加 `gc_savepoint` 系统表，可以通过 SQL 增删改查来进行管理。
- GC 进行时，需要将 `gc_savepoint` 表的数据，与原本的 `gc_safepoint` 一同存放到 PD。
- 修改 GC 逻辑，回收数据时考虑 `gc_savepoint`。因为 GC 有传统 GC 和 compaction GC 两种，时间关系可以只做一种。

验收效果：

- 可以通过 SQL 进行 `gc_savepoint` 的管理。
- GC 之后，每个 savepoint 前的最后一个版本数据会被保留。
- 使用历史读或者 MVCC Query in SQL 查询时，可以展示 GC 按预期运行。

### 3. Subsecond Flashback

- 添加 `flashback table begin_ts [, end_ts]` SQL 语句，用于指定 table 进行数据还原。意义是还原指定时间范围内的写入，如果不指定 end_ts，则使用当前最新版本。
- 将时间范围写入 table schema 并触发 DDL 操作，DDL 同步完成即可返回操作成功。
- TiDB 请求 TiKV 时，需要将要忽略的 ts 区间放在请求中发给 TiKV。
- 修改 MVCC 读取逻辑，要根据指定的区间跳过对应的版本。（这里有个小问题，MVCC Query in SQL 要不要跳过区间？或者我们可以加个新的虚拟列标识出来？）
- 当 ts 区间超出了 GC 范围后，需要被清理。（这个时间关系可以不做）

验收效果：

- 使用 flashback 语句还原数据。
- 还原后启动新事务进行查询，看到旧版本的数据。

### 4. Table Bounded GC Lifetime  (选做，这个感觉做了效果一般)

- 把 GC lifetime 配置，和 GC savepoint 配置都改成表级的，这样可以更好地管理不同表的备份的需求。
- GC 的时候要根据数据所属的表进行特殊处理。
