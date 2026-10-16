# 归档存储方案：讨论记录与结论

记录日期：2026-10-16。用途：保存归档存储（问题 1）讨论的背景、结论和代码依据，方便后续继续推进。

完整设计文档：https://claude.ai/code/artifact/918c0de5-9432-47c2-84f1-931dccc6d0d9

## 1. 背景

- BSC 是 go-ethereum 的长期 fork，`go.mod` 的 module 名仍是 `github.com/ethereum/go-ethereum`。BSC 不 rebase，而是定期 merge 上游 release，目前对齐 geth v1.17.3（合并提交见 `d1d934a`、`422b270`、`edb32e7`、`6d65a79`），BSC 自身版本为 1.8.0。
- BNB Chain 计划推出下一代 L1（公开路线图：2026 年底测试网、2027 年初主网），该链**需要兼容 EVM**（已确认）。
- 团队提出的三个问题：
  1. 归档节点约 50TB 且每天增长，需要优化存储（**本记录聚焦此问题**）；
  2. 新验证者同步慢；
  3. 验证者间区块同步约 40ms，其中网络约 20ms。

## 2. 关于 50TB 的判断

结论：50TB 是 hash scheme 且保留大量历史状态的节点，即 **hash 归档节点**；但很可能不是从创世块开始的完整归档，而是从某个快照高度开始的归档。

| 节点类型（BSC 主网） | 体积 | 依据 |
| --- | --- | --- |
| 轻节点 | 约 0.08–0.13TB（理论值） | 当前 BSC 只有 snap/full 同步，没有轻节点；按 1.28 亿个区块头 × 0.6–1KB 估算 |
| path 全节点（裁剪老区块） | 约 1.7TB | 官方 2026-08-05 快照 |
| path 全节点（完整区块） | 约 6.6TB | 同上；72 天增长 0.6TB，约 8GB/天 |
| 差异式归档（Erigon、reth-bsc） | 约 5–7.7TB | reth-bsc 归档快照 7.02TiB |
| hash 归档 | 2022 年约 30TB；约 2023 年约 50TB | NodeReal 等 |

推理要点：

- 50TB 是 path 完整全节点的 7.6 倍、差异式归档的 6.5 倍以上；区块、交易与收据只有约 4.9TB，剩下约 43TB 只能是历史状态，而只有 hash scheme 会原样保留历史 trie 节点。
- 30TB → 50TB 约用一年（约 55GB/天，当时每天 2.88 万块）。按此外推，从创世块同步的完整 hash 归档今天应 ≥120TB，所以 50TB 更可能是从快照高度开始的归档。
- 区块高度约 1.28 亿，是用 `params/config.go` 的硬分叉时间和出块间隔（3s → Lorentz 1.5s → Maxwell 0.75s → Fermi 0.45s）推算的；现在每天约 19.2 万块。
- 外部数字均来自搜索结果摘要，未能打开原文；公共 BSC RPC 在当前环境被代理拦截，无链上实测。

确认方法：

1. 查看 `--state.scheme`、`--gcmode`；
2. `geth db inspect` 看 trie 节点占比；
3. 对第 1、1000 万、5000 万块调用 `eth_getBalance`，看最早可用状态；
4. 记录一周日增量（path 全节点约 8GB/天；hash 归档在 2022–2023 年约 55GB/天）。

## 3. 关键概念（讨论中澄清过）

- 三个**独立**开关：`--syncmode`（snap/full，决定初始状态从哪来）、`--gcmode`（full/archive，决定旧状态删不删）、`--history.blocks` / `--history.chain`（决定老区块删不删）。"archive sync" 不是同步方式。
- 只有 `full sync + gcmode archive` 才有从创世块起的完整历史状态；snap sync + archive 只有 pivot 之后的。
- 归档节点存的是**每个历史区块时**的全部账户状态，体积大头是历史状态，不是区块。
- 状态裁剪方式：
  - hash 运行时：引用计数，只在内存丢弃（`core/blockchain.go:2073-2080` 的 `Dereference`），落盘后不再删除；
  - hash 离线：`geth snapshot prune-state`，布隆过滤器标记存活节点后遍历全库逐个 `batch.Delete`，再 compaction（`core/state/pruner/pruner.go:122-225`）；
  - path 当前状态：按路径原地覆盖或删除（`triedb/pathdb/flush.go:46-55`），没有垃圾；
  - path 状态历史：freezer 尾部截断，整文件删除（`triedb/pathdb/history.go:301`、`core/rawdb/freezer_table.go:720`）。
- BAL（Block-level Access List，EIP-7928）：随区块发布、记录本块读写的状态及修改后的新值；区块头只放哈希（`core/types/block.go:134`）。BSC 中类型已存在（`core/types/bal/`），但 Parlia 尚未生成或使用。代价是区块变大，与问题 3 的延迟目标存在冲突。

## 4. 方案要点（新链）

- **两层结构**（因为需要兼容 EVM，两层都要做）：
  - 第一层（流式历史，必做）：区块、收据、BAL 按高度写成不可变分段，存于对象存储，对外提供拉流接口（参考 Aptos indexer-grpc）。这是唯一的权威数据，替代归档节点。
  - 第二层（EVM 历史查询）：key-segments、key-index、tx-index、log-index 与历史状态读取器，提供 `eth_call`@N、trace、`eth_getLogs`。只建索引，指向第一层文件中的压缩帧，可从第一层重建。
- 分段文件内部按约 64KB 压缩帧组织并带偏移表，必须在第一个分段写出前定下。
- 已知风险：对象存储点查延迟（几十毫秒/次）；索引本身随历史增长（100k TPS 下 tx-index 约 8TB/年）；BAL 增大区块；历史过期带来的生态兼容问题。
- 决策时点：协议决策（BAL 及传播方式、去 logsBloom、历史过期、状态承诺结构）需在新链规范冻结前定；第一层格式需在第一个分段写出前定；第二层可上线后迭代。
- **BSC 现网立即可做**：把 hash 归档换成 Erigon、reth-bsc 或 Geth path 归档（`--state.scheme path --gcmode archive --history.state 0`，当前代码已支持），预计 50TB → 5–7TB。需先在测试网验证，因为 BSC 改过 `triedb/database.go`（+66 行）和 `pathdb/layertree.go`（+45 行）。

## 5. 涉及模块与 merge 策略

与 geth v1.17.3 逐文件 diff（仅非测试代码）的主要结果：

| 模块 | 与上游差异 | 方案中的动作 | merge 风险 |
| --- | --- | --- | --- |
| `core/blockchain.go` | +852 / −232 | 不改，通过钩子取数据 | 高 |
| `core/rawdb`（freezer 等） | 18 个文件，+1262 / −203 | 只读访问，不改格式 | 高 |
| `internal/ethapi/api.go` | +810 | 不改；第二层独立进程复用 | 高 |
| `eth/tracers/api.go` | +239 | 不改；第二层实现其 Backend 接口 | 中 |
| `eth/api_backend.go`、`eth/state_accessor.go` | +96、+34 | 可选：回退链末尾加查历史服务 | 中 |
| `consensus/parlia` | BSC 独有 | 新链生成与校验 BAL | 中 |
| `cmd/geth`、`cmd/utils` | +1703、+814 | 新命令放新文件 | 中 |
| `core/tracing`、`eth/tracers/live` | `hooks.go` +56；`live` 未改 | 第一层导出器做成 live tracer | 低 |
| `core/state`（`StateUpdate`、`HistoricDB`） | `stateupdate.go` 未改；`database_history.go` +4 | 第二层实现读分段的 Reader | 低 |
| `triedb/pathdb` | 31 个文件未改，5 个文件 +89 | 不改；作为原型参考 | 低 |
| `internal/era`（era1/erae） | 仅 `accumulator.go` +2 | 第一层格式向 erae 靠拢 | 低 |
| `core/filtermaps`、`core/txindexer.go` | +114、+23 | 第二层复用算法 | 低 |
| `core/types/bal` | 未改 | 新链启用 | 低 |

落地原则：新代码放新文件、新目录，通过上游已有接口挂接，不碰 `core/blockchain.go`、`core/rawdb` freezer、`internal/ethapi`。

- 写入侧挂点：`OnStateUpdate` 钩子（`core/blockchain.go:1993`），用 `--vmtrace` 启用 live tracer。
- 读取侧挂点：`eth/api_backend.go:226` 的回退链（`StateAt` → `HistoricState`，`core/blockchain_reader.go:561`）和 `core/state/database_history.go` 的 `HistoricDB`。
- 建议目录：`eth/tracers/live/historyexport.go`、新包 `history/`、`cmd/historyd/`、`cmd/geth/historycmd.go`。
- 挂在上游接口上的代码虽不会产生 merge 冲突，但上游改接口时可能编译失败，需把导出器与读取器测试纳入每次 merge 的验收。

## 6. 待决问题

- [x] 新链是否兼容 EVM —— 已确认兼容，第二层必做。
- [ ] 现有 50TB 的构成，以及是否从创世块开始（见第 2 节确认方法）。
- [ ] BAL 本体随区块传播还是只承诺哈希、异步传播（需与问题 3 一起定）。
- [ ] 归档数据主要使用方，以及 `eth_call`、trace、`getLogs` 的查询占比。
- [ ] 历史 `eth_getProof` 是否有刚性需求及时间窗口。
- [ ] 新链状态承诺结构；全节点历史保留窗口；历史服务由谁运营；历史查询 SLA。
- [ ] `StateUpdate` 能否完整取得明文地址与存储键（`StorageKeyType`）；启用 `OnStateUpdate` 后 `CommitWithUpdate` 的额外开销。

## 7. 建议的下一步

1. 拿到 50TB 节点的配置、`geth db inspect` 结果和日增量。
2. 在测试网验证 BSC 的 path 归档（索引与历史查询）。
3. 做第一层导出器原型：一个 live tracer，在本地开发链上导出每块状态变更，同时回答第 6 节最后一项。
