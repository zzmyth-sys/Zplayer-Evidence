- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 主仓路径前缀: `entry/src/main/ets/`
- 本目录: `DOC/V402/CODE-REVIEW/`（代码原文摘录，不含摘要/结论）

## 文件清单

| 文件名 | 原路径 | 行号区间 |
|---|---|---|
| `player_contract.md` | `kernel/player_contract.ets` | 108-208 |
| `PlaybackCommandCenter.md` | `domains/player/command/PlaybackCommandCenter.ets` | 21-39、45-68、80-104、106-110、166-175 |
| `CommandAdmissionCtl.md` | `domains/player/command/CommandAdmissionCtl.ets` | 1-29（全文） |
| `HlsDegradeCtl.md` | `domains/player/recovery/HlsDegradeCtl.ets` | 1-74（全文） |
| `SourceDowngradeCtl.md` | `domains/player/recovery/SourceDowngradeCtl.ets` | 34-70、100-182、249-268 |
| `PlaybackProjection.md` | `pages/player/PlaybackProjection.ets` | 1-92（全文，@Trace 在 11-19） |
| `PlayerRuntime.md` | `domains/player/PlayerRuntime.ets` | 1-26（全文）+ 写入点 `PlayerAssembly.ets:214-220` |
| `PlayerAssembly.md` | `domains/player/PlayerAssembly.ets` | 194-243 |
| `SourceResolver.md` | `domains/player/prepare/SourceResolver.ets` | 28-82 |
| `SffAdapter.md` | `domains/smb/media_source/SffAdapter.ets` | 32-113 |
| `SmbAdapter.md` | `domains/smb/media_source/SmbAdapter.ets` | 31-43 |
| `SmbStreamHandle.md` | `domains/smb/SmbStreamHandle.ets` | 60-77 |
| `SmbHttpProxy.md` | `domains/smb/SmbHttpProxy.ets` | 77-143 |
| `PlaybackEntryFlow.md` | `flows/playback/PlaybackEntryFlow.ets` | 9-44 |
| `PreloadCommitFlow.md` | `flows/playback/PreloadCommitFlow.ets` | 9-26 |
| `PlaybackControlFlow.md` | `flows/playback/PlaybackControlFlow.ets` | 7-23 |
| `PresentationFlow.md` | `flows/playback/PresentationFlow.ets` | 6-11 |
| `PlaybackLifecycleFlow.md` | `flows/playback/PlaybackLifecycleFlow.ets` | 7-15 |
| `PreloadPrepareFlow.md` | `flows/playback/PreloadPrepareFlow.ets` | 7-14 |
| `FlowErrorBoundary.md` | `flows/playback/FlowErrorBoundary.ets` | 7-16 |

## FLOW 业务分支标注

| 文件 | 有 if/switch 业务分支 | 行号 |
|---|---|---|
| `PlaybackEntryFlow.ets` | 有 | 22（if `next === null`）、33（if `videoIdx >= 0`）、37（for）、40（三元） |
| `PreloadCommitFlow.ets` | 有 | 14（if `Committed`）、18（if `Fallback`） |
| `PlaybackControlFlow.ets` | 无 | 12 仅等值比较生成 `playing` |
| `PlaybackLifecycleFlow.ets` | 无 | — |
| `PreloadPrepareFlow.ets` | 无 | — |
| `PresentationFlow.ets` | 无 | — |
| `FlowErrorBoundary.ets` | 无 | 8-15 仅 try/catch 统一出口 |

- `flows/playback/RegisterPlaybackFlows.ets`（19 行）为唯一订阅聚合入口，无业务分支，未纳入上述 7 个文件。

## 脱敏

- 摘录内唯一出现的网络字面量为回环 `127.0.0.1`（监听/中继地址，属代码逻辑，未代号化）；无主机名、真实 IP、共享名、设备序列号、凭据字面量。
- 区间外内容（配置、凭据、部署拓扑）未拷入。
