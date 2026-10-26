# V402 / MAP — Zplayer 项目地图（结构，不含代码与测试数据）

> 生效：V402-033｜基线：主仓 `main` @ `0316b56`，工作分支 `docs/v402-028-p0-2-closure-v2` @ `d67e179`
> 性质：只做结构定位；不写代码、不测数据、不改业务。证据仓：`Zplayer-Evidence`（public）。
> 脱敏：主机/共享/凭据/序列号一律用 `<SFF_HOST>`、`<NAS_HOST>`、`<SHARE>`、`<DEVICE_SERIAL>` 代号。

## 索引

| 章节 | 行号 | 一句话结论 |
|---|---|---|
| §1 仓库拓扑 | 17 | 主仓、加速器、SFF 实例、证据仓、Demo 沙盒、debugly 上游是六个独立对象；SFF 是部署宿主，不是仓。 |
| §2 三条播放链路 | 31 | S=主链路 HLS；B=本地 HTTP 代理兜底；A=Demo 内 IJK 原生 `smb://` 探索。 |
| §3 状态机与门禁 | 74 | UI 只走 IntentBus，命令只进 CommandCenter，编排只在 flows/playback，双槽由 Pool 原子切换。 |
| §4 工作项真实状态 | 133 | P0-2 已关闭；C6 主链已合但 009d/009e 未做且开关为 false；A4 仍未完成。 |
| §5 战略层级与判据 | 157 | SFF 主力、B 侧兜底、A 路径探索更好的兜底；判据分别来自基准白皮书、T-08/T-10、评估文档。 |

## §1 仓库拓扑

| 对象 | 用途 | 可见性 | 远端 | owner | 与其它仓的关系 |
|---|---|---|---|---|---|
| `Zplayer` 主仓 | HarmonyOS 客户端：播放域、双 IJK 槽、SMB 代理、SFF 取源、UI/媒体库 | **private** | `github.com/zzmyth-sys/Zplayer.git` | `zzmyth-sys` | 客户端主体；通过 HTTP 调 ZP-Accelerator；只读引用证据仓；A 路径产物来自 Demo。封板 tag `v1.1.0-V401` → `0c92e06`。 |
| `ZP-Accelerator` | SFF 服务端加速大脑：FastAPI，`/probe`、`/stream/start`、`/hls`、`/frame`、`/vfs`、`/health`，端口 8612 | 本地私有 | **无远端**（`git remote -v` 为空） | `zzmyth` | **独立仓**；部署到 SFF 主机；不是主仓子模块，也不是主仓内的一个目录。 |
| `SFF 部署` | 运行实例：Windows 主机 + 计划任务 `ZPAccelerator` + ffmpeg + 8612；SMB 共享由 `<NAS_HOST>` 提供（可能同机） | 非仓（主机实例） | 无 | `zzmyth-sys`（运维） | **SFF 是部署宿主，不是仓**。ZP-Accelerator 是代码，SFF 是它的运行环境；两者不是“同仓不同组件”，也不是“子模块”，而是“独立仓 + 部署实例”。 |
| `Zplayer-Evidence` | 证据交换仓：`DOC/V402/T-0x.md`、本图 | **public** | `github.com/zzmyth-sys/Zplayer-Evidence.git` | `zzmyth-sys` | 与主仓分离；主仓只引用不包含；不是子模块。 |
| `ZplayerDemo` | A 路径沙盒：定制 FFmpeg 8 + IJK + libsmb2，产出 `ijkplayer.har` / `v402-libs` | 本地私有 | **无远端** | `zzmyth` | **独立沙盒**；内嵌 `ohos_ijkplayer`、`ohos_ffmpeg`、`libsmb2` 三份上游源码（各有自己的 remote），但不是 git 子模块；产物只作评估，未合并主仓。 |
| `debugly` 上游 | `MRFFToolChainBuildShell`（libsmb2 URLProtocol 补丁 + OHOS 构建配方）、`fsplayer`（IJK+FFmpeg8 参考）、`ijkplayer`（无 SMB） | **public** | `github.com/debugly/*` | `debugly` | **外部上游**；项目只在 Demo 内自持 `0019/0020/0021` 三个 patch，不 fork、不作为子模块。 |

- 结论一句：**ZP-Accelerator 与 SFF 的关系是“独立仓 + 部署实例”**；SMB 共享由 `<NAS_HOST>` 提供，本图不判定 `<NAS_HOST>` 与 `<SFF_HOST>` 是否同一物理机。
- 主仓可见性受 J7 约束：保持 private；证据仓 public，故本图必须脱敏。

## §2 三条播放链路

### 链路 S（SFF HLS，主力）

- **入口**：UI `IntentBus.dispatch(PlayItem)` → `flows/playback/PlaybackEntryFlow` → `PlaybackCommandCenter.submit(Prepare)`。
- **组件**：`SourceResolver` → `domains/smb/media_source/SffAdapter.open` → `SffHttpClient`（`/vfs` → `/probe` → `/stream/start`）→ `SffAdapter.hlsUrl`；`IjkPlayerBackend.setDataSource(HLS)` → `libavformat` HLS demux → `libavcodec` `ohcodec` → `VideoCodecMode.SURFACE`。
- **时序**：`UI → CommandCenter(Prepare) → SourceResolver → SffAdapter → SFF /vfs → /probe → /stream/start → index.m3u8 → IJK 拉 seg-N.ts → ohcodec → Surface`。
- **出口**：IJK + 官方 ohcodec → Surface（消费方是渲染服务）。
- **故障注入点**：
  - `/vfs`、`/probe`、`/stream/start` 超时/5xx → `SffAdapter` 抛 `AttemptFallback` → 门面回落 SMB（取源期）。
  - `SffHttpClient.downUntil` 熔断只由 JS 侧 HTTP 失败置位，**看不到 native 拉流失败**（审计 D3/P1）。
  - `seg-0` 504 / 断连发生在 `prepare()` 返回 `Ok` 之后 → 走运行期 `streamFailure`，不是 prepare 降级（P0-1 的根因面）。
  - HLS 停滞 8s → `PlayStallWatchdog` → `onHlsStall` → 降级链（阈值来源见 §5 注）。
  - seek 连点 → 服务端 latest-wins 返回 409（`ZP-Accelerator` `seek 合并`）。
  - 4K 源当前会被 SFF 强制转 608×1080（`PERFORMANCE-BASELINE.md` §5 第 1 条）。

### 链路 B（SmbHttpProxy 兜底）

- **入口**：同一 UI 路径 → `SourceResolver` → `domains/smb/media_source/SmbAdapter.open` → `SmbService.openPlayStream` → `SmbStreamHandle.create`。
- **组件（逐跳）**：
  1. `SmbStreamHandle.create` → `new SmbHttpProxy(...).start()`（`SmbStreamHandle.ets:60-77`）。
  2. `SmbHttpProxy.start`：`PortPool.acquire` → `smbStatSize` → `socket.TCPSocketServer.listen(<loopback>:<port>)`（`SmbHttpProxy.ets:77-137`）。
  3. IJK `setDataSource(http://<loopback>:<port>/video.mp4)` → HTTP Range。
  4. `SmbHttpProxy` → `SmbOpQueue.run/runLow`（全局严格串行，高/低优 + 老化，`SmbOpQueue.ets:22-116`）。
  5. `ohos_libsmb` NAPI：`smb_napi.cpp` 的 `smbStatSize` / `smbReadAt` → `smb_session_io.cpp` 的 `smb2_pread` → `libsmb2.so.1`。
  6. SMB2 → `<NAS_HOST>:445/<SHARE>`。
- **时序**：`UI → CommandCenter → SourceResolver → SmbAdapter → SmbStreamHandle.create → SmbHttpProxy.start → PortPool → statSize → listen → IJK GET Range → SmbOpQueue → NAPI smbReadAt → smb2_pread → NAS`。
- **出口**：IJK HTTP 代理 → ohcodec → Surface。
- **故障注入点**：`statSize<=0`、端口池耗尽、队列被高优饿死/低优被 drop、SMB 断连重连（`SmbReconnect`）、NAPI 异常、256KB 块读 + `ProxyCache`、4K 直读 App CPU ≈160%。

### 链路 A（DEMO 路径二：IJK 内置 libsmb2 URLProtocol 直读）

- **入口**：`ZplayerDemo/entry/src/main/ets/pages/Index.ets`（`mode=direct`，构造 `smb://<user>:<pass>@<host>/<share>/<path>`，`VideoCodecMode.SURFACE`）。
- **组件**：定制 `libavformat.so`（含 `ff_libsmb2_protocol`）+ `libsmb2.so.1` + `libijkplayer.so` / `libijksdl.so`；`libavcodec` 保持官方 ohcodec。
- **补丁位置**：
  - `native/patches/0019-add-built-in-smb2-protocol-via-libsmb2.patch`：新增 `libavformat/libsmb2.c`，并在 `configure` / `libavformat/Makefile` / `libavformat/protocols.c` 注册。
  - `native/patches/0020-smb2-async-prefetch-ring.patch`：预取环 + seek 白名单。
  - `native/patches/0021-ijkplayer-har-smb2-link.patch`：HAR 链接 libsmb2。
- **FFmpeg 接入点**：`native/src/ohos_ffmpeg/libavformat/libsmb2.c:399`（`smb2`→`smb` URL 重写）、`:839-840`（`const URLProtocol ff_libsmb2_protocol`，`.name="smb"`）、`libavformat/seek.c:193`（白名单加 `smb`）。
- **时序**：`Index → IJK setDataSource(smb://) → avformat_open_input → ff_libsmb2_protocol.open → smb2_connect_share/smb2_open/smb2_stat → smb2_pread 读写 → demux → ohcodec → Surface`。
- **出口**：IJK + ohcodec → Surface。
- **故障注入点**：读放大（构建 PoC 已修到 1.0×）、断网不恢复（PoC 已修到 ~5s onError）、双并发 `smb2_context` 未验证、凭据/中文路径/域、`net_lock` 串行化、4K60 长稳；连续远跳无 latest-wins（单次 ~0.9s）。

## §3 状态机与门禁

### 3.1 PlaybackCommandCenter（唯一命令门禁）

- **管哪些命令**：`Prepare` / `Adopt` / `Play` / `Pause` / `Seek` / `SetRate` / `Release`（`kernel/player_contract.ets:109`）。
- **准入规则**（`command/CommandAdmissionCtl.ets:9` `check`）：
  - `Released`：只放行 `Prepare` / `Adopt` / `Release`，其余 `Rejected`。
  - `Error`：`Play` / `Pause` 拒绝。
  - `releasing`：`Release` 放行且幂等；`Prepare` → `DeferPrepare`；`Adopt` 及其它 → `Rejected`。
- **关键迁移与函数**（`command/PlaybackCommandCenter.ets`）：
  - `submit`：先准入；`DeferPrepare` 不铸 epoch，执行时补铸。
  - `enqueue`：`Prepare`/`Adopt` 铸新 epoch、丢弃排队 Prepare、给当前执行体让位窗口；`Release` 置 `releasing`、铸 epoch、清队列、挂 `pendingRelease`；其余命令按 kind latest-wins。
  - `advanceEpoch` / `preemptRunning` / `cancelRunningForSuccessor` / `onHardTimeout` / `settle`：epoch、抢占、硬超时、orphan、结算。
  - `attachReleaseBody`：Lifecycle 把拆卸体挂到 Release 槽，首个 body 生效。
- **门禁边界**：只依赖 `PlaybackEventPort`（出口）与 `PlaybackCommandView`（只读 + 两本台账）；不直接碰 UI。

### 3.2 IntentBus + PlaybackProjection（UI 的唯一发/读通道）

- **发命令**：`app/IntentBus.ets` 的 `SimpleIntentBus.dispatch/subscribe`；无全局单例，由 `app/Setup.ets` 注入。订阅集中在 `flows/playback/RegisterPlaybackFlows.ets`。
- **读状态**：`pages/player/PlaybackProjection.ets`（`@ObservedV2` 只读镜像）+ `ProjectionCore`（纯逻辑）；`bind(view)` 订阅 state/position/session，`applySnapshot/applySession/mirror` 写入 `@Trace`。
- **铁律 G6**：UI 只发 IntentBus / 只读 Projection；编排只在 `flows/playback/`；`PlaybackCommandCenter` 唯一门禁；不新增运行时兜底。

### 3.3 flows/playback（编排位置）

| 文件 | 订阅/职责 |
|---|---|
| `PlaybackEntryFlow.ets` | `PlayItem` / `StepItem` → 起播/切条入口 |
| `PreloadPrepareFlow.ets` | `PreloadAdjacent` → 相邻预加载 |
| `PreloadCommitFlow.ets` | `CommitSelection` → 预加载转正 |
| `PlaybackControlFlow.ets` | `Control` / `SeekGesture` → 播放控制 |
| `PlaybackLifecycleFlow.ets` | `Release` → 拆卸 |
| `PresentationFlow.ets` | `Presentation` → 呈现/方向 |
| `FlowErrorBoundary.ets` | 边界错误 → 事件出口 |

### 3.4 双 IJK 槽（A/B 槽）

- **槽与切换**：`PlayerBackendPool` 固定 0/1 两槽，初始 0 active；`preloadSlot()` 返回另一槽；`commit()` 原子翻转 active（`PlayerBackendPool.ets:53-66`）；`PlayerSlot.active` 决定事件是否转发与 fatal 是否按预加载失败处理（`PlayerSlot.ets:19-33`）。
- **切换触发**：用户选中已预加载项 → `PreloadCommitFlow` → `Adopt` → `PlaybackCommandCenter` → `IjkCommitAdmissionCtl.admit`（ready + uri + Intent + fence + surface + lease）→ `commit()`；未命中则走普通 `Prepare`。
- **预加载失败降级**：非 active 槽 fatal → `PlayerSlot.reportFatal` → `PlayerBootstrapper.onPreloadFailed` → `IjkPreloadCoordinator.onPreloadFailed` → `IjkPreloadCtl.onFailed`（`detachReady`、清 item/stream、`failed=true`、记 `failedPreloadGen`、`resetIdle`）→ commit 准入失败 → 主路径重新 prepare；`SurfaceOpsCtl.dropSlot` 负责 surface 侧清理。
- **释放时机**：`IjkPreloadCtl.abort`（释放 aux lease/stream、等 drain、`dropForReplace`、清 preload page）；`PlayerLifecycleCtl.release` → `intent.reset()` → `hlsDegrade.clear()` → `bindings.clearAll()`；`PlaybackCommandCenter` Release 槽 → `releaseBodyFor` → `pool.releaseAll()`。
- **边界**：SFF 不支持预加载槽（`SffAdapter.open` 对 `purpose='preload'` 直接 `AttemptFallback`），所以预加载槽实际是 **SMB aux**。

### 3.5 W2 降级链（现状校正）

> 指令里的“504 → onIjkFatal → 降级入口”是旧路径。现状（V402）**不是** `onIjkFatal`：`CodecFallbackController.onIjkFatal` 只发 Fatal，不降级（`:54-62`）。实际链路如下。

- **实际链路**：
  1. SFF `/hls/.../seg-N.ts` 504/断连/无帧 → `IjkPlayerBackend` 的 `onError` / 无帧分支判定 HLS 传输失败。
  2. → `setStreamFailureHandler` 注入的 sink（`PlayerBootstrapper.attachStreamFailure`，`:241-263`）。
  3. → `PlayerAssembly.onHlsStreamFailure`（`:486-487`）。
  4. → `PlayerAssembly.failHlsStream`（`:223-225`，一行转发）。
  5. → `HlsDegradeCtl.fail`（重入吸收、非 HLS 不吞错、Ok 后补 arm 看门狗）。
  6. → `SourceDowngradeCtl.downgradeFromHlsDetailed` → `run`。
  7. `run`：`isCurrent` →（v2：latestKey/inFlight）→ `tried` 门禁 → `releaseBackend(dead)` → await 后复检 → `beginGen` → `serialPrepare(item, avoid=['SFF'])` → 新源 `Prepared` → 返回 `Ok`。
  8. → `HlsDegradeCtl` 用新 `RequestScope` 重新 arm 停滞看门狗。
- **停滞分支**：`PlayStallWatchdog.setOnHlsStall` → `PlayerBootstrapper` 的 `onHlsStreamFailure('hls-stall')` → 同链。
- **键与开关**：`HLS_DEGRADE_KEY_V2=false`（`SourceDowngradeCtl.ets:18`）→ 当前走旧 `uri#gen` 键；v2 为 `uri#gen#attempt`。`clear()` 在 release 清键；`prepareChain` 对同 uri 串行（009c）。
- **谁调谁**：`PlaybackCommandCenter` 不直接调降级；降级是播放域内部 `PlayerBootstrapper → PlayerAssembly → HlsDegradeCtl → SourceDowngradeCtl` 的调用链；UI 只看到最终状态/错误。

## §4 当前工作项真实状态（校准表）

| 项 | 状态 | 锚点（commit / PR / 文件） | 说明 |
|---|---|---|---|
| P0-1 网络降级盲区 | **在飞** | `DOC/V402/T-02.md` §5.2；#19 `24b05df`、#20 `305a9e4`、#21 `1f86c35`、#24 `3f6c3f0` | W2 已定稿 `uri#gen#attempt`；代码已合，但 `HLS_DEGRADE_KEY_V2=false`，剩真机 U1a/U1b + 翻 true；未整体关闭。 |
| P0-2 SFF 死锁 | **已关闭（Closed-Resolved）** | 修复 `c6ad7fd`；回滚门禁 `a5375a1`；关闭 `65eaf03`（PR #27 / V402-028） | 已改判 scanner `probe` 的 `_used["probe"]` KeyError 泄漏（`stream`+`total` 各漏 1/次）；4/4 取消用例通过；证据 `T-06`/`T-09`。 |
| C6（HLS 降级单飞） | **在飞/收尾** | `T-02`；#19/#20/#21/#24 | 主链（单飞、attempt、分支层、T9）已完成；009d 未做；开关 false；C6 不判整体关闭。 |
| 009b | **已完成** | PR #21 / `1f86c35` | `HlsDegradeCtl` 分支层 + T5–T8/R-C6-1；255/255。 |
| 009c | **已完成** | PR #24 / `3f6c3f0`；收尾 #25 / `0316b56` | T9 串行 + a/b/c；258/258。 |
| 009d | **登记待做** | `T-02.md` §10.6；#25 / `0316b56` | 删 `SourceDowngradeCtl` boolean 包装 + `prepareChain` 收敛；开工前先出调用方清单。 |
| 009e | **登记待做（未开工）** | `DOC/V402/T-009e.md`；`65eaf03` | 纯测试；实现仓 `ZP-Accelerator/tests/`；不碰 009d、不改实现。 |
| T9 串行 | **已完成** | 009c / PR #24 | 同 uri 新旧 gen 并发 → `prepare` 串行、`maxActive=1`；258/258。 |
| R-C6-1 投影断言 | **已完成** | 009b / PR #21 | T8：新 Prepared 生效；迟到旧结果按 epoch/uri 丢弃；255/255。 |
| DEMO PoC-1 | **未开工** | 无任务单 / 无 commit | 仓库内无开工证据；既有 `ijk-smb-build-poc.md` 是构建 PoC（已完成），不等同本项。 |
| SFF 兜底验证（P1） | **未做/无独立单** | 审计 `network-degrade-boundary-audit.md` D3/D4（P1） | 未见独立任务单/提交；可对应的 P1 登记是“SFF 熔断看不到 native 拉流失败”与“prepare/playback 无失败语义传递”，均未闭环。 |
| 快速切换 `setDataSource=-3` P0 | **已知缺陷；根因已知，修复待复测** | `v401-multi-device-test.md` §1.3；`IjkPlayerBackend.ets:173-191`；`0c92e06` | 根因：同一 IJK 实例在非 Idle/Released 状态重复 prepare → `-3`；已有 stop/reset guard；主仓 T-10 §5 仍引用 v401 记录，**修复后是否仍复现未复测**。 |
| 12G 大文件测试 | **半做** | `T-10.md` §4；`d67e179` | A 侧 n=1：起播 435ms / CPU 11.3% / PSS 152MB / 读放大 1.16×；**B 侧未做**，非口径矩阵未做。 |
| A 路径稳定性（V402-030） | **定位完成；A/B 仍阻塞** | 证据仓 `T-10.md` §1–§3；`bd4fa38` | 4K 崩溃 3 样本不可复现（归 D）；单次远跳 ~0.9s，连续拖拽会反复重置 16MB 环；A 工装无进度条，拖拽闸无法测。 |
| B 路径 cppcrash | **已定位；无修复提交** | 证据仓 `T-10.md` §2.1 | 2 条 cppcrash 均在 `libsmbsession.so`（`SmbSession::MarkDead` / `ListDir` UAF）；与 A 路径无关；B 侧“不崩”判据已有已知反例。 |
| 分支保护 | **未闭环** | `GOVERNANCE-TEMP.md` §1/§8；`V402-005` §3 | private user-owned 仓 GET/PUT 均 403；需 Pro 或转 public；是否升级待裁。 |
| secret scanning | **未闭环** | `GOVERNANCE-TEMP.md` §6；`V402-005` §6 | 422 不可用；已不批准“为它升 Pro/Team”；仅 org+Team+Secret Protection 或 Enterprise Cloud 路径。 |
| workflow scope | **未闭环** | `GOVERNANCE-TEMP.md` §1/§8 | push workflow 被拒（无 workflow scope）；Actions workflows 总数 0；是否授权待裁。 |
| 009e 任务单位置 | **`DOC/V402/T-009e.md`** | `65eaf03` | 实现仓 `ZP-Accelerator`；仅 `tests/`；不碰 009d/不改实现。 |

## §5 战略层级与判据

| 层级 | 定位 | 判据 | 判据来源 | 当前实测/状态 |
|---|---|---|---|---|
| **SFF HLS（主力）** | 默认最优解 | 首帧 ≤400ms；App CPU ≤20%；seek 中位 ≤100ms | `PERFORMANCE-BASELINE.md` §6（验收清单）/ §2 | 2.5K：190–360ms / 14.0% / 45ms；4K 被强制降 608×1080 |
| **B 侧 SMB 代理（兜底）** | 能播、不卡死、不崩 | 不设 CPU 硬门槛；切换/失败/卡死/句柄按 G-C | `T-08.md` §6 G-C；`T-10.md` §6 G4/G5 | 4K：297ms / App CPU ≈160%；CPU 不作 B 侧硬门槛（A 侧另设 CPU 判据） |
| **A 路径 DEMO（探索更好的兜底）** | 原生 `smb://` 直读，未合并 | 稳定性优先；CPU <25%（评估文档门槛，裁判口径为加分） | `smb-native-migration-assessment.md` §7（评估文档） | 4K：327ms / 15.2% / 读放大 1.03×；12G：435ms / 11.3%；4K 崩溃不可复现（D）；单次远跳 ~0.9s、无 latest-wins；闸 2 未全开，维持封存 |

- **战略一句话**：SFF 主力、B 侧兜底、A 路径探索“更好的兜底”；A 路径的 CPU 数量级优势已被数据证明，但合并门槛（拖拽/快速切换/12G-B/回归）未过。
- **来源不明/未认证（不是互相矛盾，但不得当基线）**：HLS 停滞 8s 是代码常量，不是实测拟合；SFF 段长/切片重试未测（`PERFORMANCE-BASELINE.md` §8.2–§8.4）。引用时必须标“代码常量/待测”。

### 待裁决单点（≤3）

1. **A/B 主判据**：按“全面不劣于”字面，A 首帧慢 10% 即劣化；T-10 建议“首帧 +15% 容差、CPU 为主判据”。二者需裁定（`T-10.md` §9-1）。
2. **12G 的 B 侧是否必须补**：不补则 12G 只有 A 侧单边数据，不能进 A/B 结论；补则需先修 `--ps share` 环境（`T-10.md` §9-2）。
3. **快速切换先测还是先修**：A4 要求“切换失败 0 容忍”，当前 `setDataSource=-3` P0 未复测（代码已有 stop/reset guard，但未确认消除）；先测失败率还是先修 P0，需裁定（`T-08.md` §9-3）。

## 附：本图边界与复核

- 不写代码、不测数据、不改业务；只引用 `file:line` 与关键函数名。
- 不涉及 v1.1.0-V401 tag 移动/删除/强推；主仓保持 private。
- 文中“已完成”以主仓合并提交/PR 或证据仓文件为锚；没有锚的项一律写“未开工/无独立单/待查”。
