# P0-1-WD · 静默停滞检测设计 + 故障注入开关设计（V402-056）

> 性质：设计文档（0 代码改动、0 主仓提交、0 生产文件改动）
> 依据：`DOC/V402/T-15.md` §3 实测；主仓 `52ba806` 源码只读
> 优先级：P0-1-WD 检测 > 故障注入（注入只是验证手段）

## 1. 结论

现有两级看门狗都不覆盖"整条链路完全冻结"（位置与帧同时停）：

| 机制 | 判据 | 覆盖 | 缺口 |
|---|---|---|---|
| `PlayStallWatchdog`（`domains/player/PlayStallWatchdog.ets`） | 起播 8 s（HLS）/15 s 仍未进入 `Prepared` | **仅起播期** | `onFire` 里 HLS 分支遇到 `Playing/Paused/Error/Released` **直接 return** → 播放中永不判定 |
| `RenderLiveness`（`domains/player/watch/RenderLiveness.ets`） | 位置**在前进**（`pos > lastPos+200ms`）且 3 s 内无解码/输出帧 | 时钟推进但画面冻结（音画分离、渲染死锁） | 位置不前进时 `advanced=false` 直接 return → **全冻结不触发** |
| `SourceDowngradeCtl` / `HlsDegradeCtl` | ijk 显式 `onError` + `isHlsTransportFailure` | 传输**报错** | 静默停滞没有 error |

实测对应：停 SFF 后位置与帧同时冻结 → 上表三行全不触发 → 画面永久冻结、无错误层、CPU 0–3.5%（`T-15.md` §3）。

## 2. 判据来源（全部为现有可读量，不新增探针）

| 编号 | 判据 | 来源 | 用途 |
|---|---|---|---|
| S1 | `positionMs` 在 `state=Playing` 下停止前进 | 快照 `PlaybackSnapshot.positionMs` / `positionSampledAtMs`（`kernel/player_contract.ets:121`） | 主判据 |
| S2 | 解码/输出帧率为 0 | `player.getVideoDecodeFramesPerSecond()`、`getVideoOutputFramesPerSecond()`（`IjkPlayerBackend.ets:369`，与 `renderStalled()` 同源） | 必要佐证（防"位置冻结但画面在动"的假阳性） |
| S3 | 传输层错误 | ijk `onError` → `isHlsTransportFailure()`（现有） | 保持现状，不并入本检测 |
| S4 | 播放结束 | `PlaybackSnapshot.ended` / `isEnded()` | 豁免（片尾静止不是停滞） |
| S5 | seek 沉降窗 | 现有 `gate.poll()`（`IjkPlayerBackend` posTimer 内）与 `RequestScope.epoch` | 豁免（seek 期间位置本来不动） |

判定式（HLS，v1）：

```
fire  ⇔  state=Playing
      ∧  ¬ended
      ∧  ¬seekSettling            (S5)
      ∧  Δpos ≤ 200 ms 持续 ≥ 8000 ms      (S1)
      ∧  frames == 0 持续 ≥ 3000 ms        (S2)
```

- 阈值取 **8000 ms**：与 `HLS_STALL_HINT_MS` 同值；SFF 切片 2 s，等价于连续 4 个分片窗口无进展。
- 采样 **1000 ms**，8 次连续不前进才计满；`rate≠1` 时按 `Δpos_expected = elapsed×rate` 缩放，`Δpos < 0.2×expected` 视为不前进。
- 非 HLS 源 v1 **不启用**（SMB 代理本地读不会静默停滞；先收敛风险面）。

### 2.1 buffering 边界（结论：**有界宽限，不豁免、不重置**）

- **现状**：`PlaybackSnapshot.buffering` 目前在 IJK 路径**恒为 false**——全仓只有 `StoreSinkFactory.onBuffering` 实现，`IjkPlayerBackend` 无调用点。v1 不得依赖该标志。
- **规则**（若日后接入 IJK onInfo buffering）：`buffering=true` 只给 **`BUFFER_GRACE_MS = 3000`** 的推迟，**不清零**已累计的冻结时长；累计达 8000 ms 照常 fire。即 buffering 最多把 fire 时刻推到 8 s，绝不无限延后。
- **恢复判定**（先到者生效，立即清零冻结累计与宽限）：`frames>0` 或 `Δpos > 200 ms`。仅 `buffering=false` 而无进展 → **不清零**。
- **理由**：SFF 局域网切片 2 s，正常缓冲 亚秒~2 s；连续 8 s 无进展 = 漏掉 4 个切片 = 传输已死。若把 buffering 当豁免，恰好会在"服务端已挂但 ijk 仍报 buffering"时失明——那正是 HLS 停滞的典型形态。

### 2.2 seek 沉降边界（结论：沉降期内**暂停累计**，退出需证据）

- **可用信号**（IJK 无 `isSeeking()` API，只能组合以下）：`PlaybackSnapshot.pendingKind === 'Seek'`、`RequestScope.epoch/reqId` 变化、位置跳变（`|Δpos| > 1500 ms` 或 `pos < lastPos`）、`SeekGate.lastDoneAt`（注意：`SERIAL_SETTLE_MS=120` 只是**命令合并**窗口，不是解码沉降，不可直接复用）。
- **进入沉降**（任一）：提交 Seek / 位置跳变 >1500 ms / 回退 / epoch 变更。
- **沉降期内**：冻结累计保持 0（不累加、不判定）。
- **提前退出**（恢复证据，任一即退出）：`Δpos > 200 ms` 或 `frames>0`（解码或输出帧计数）。
- **沉降上限**：`SEEK_SETTLE_MAX_MS = 12000`（实测远距 seek 2.5–3.2 s，留 4× 余量）。超过上限仍无进展 → 视为卡死，恢复累计，由上限定界不会无限挂起。
- **不误报保证**：累计只在沉降退出后开始，故沉降 7.9 s（<8000 ms）时，最早 fire 时刻 = 7.9 s + 8 s = 15.9 s，必然晚于沉降结束。

### 2.3 与 T-01 streamFailure 的汇合点（确认：同一漏斗）

`hls-freeze` 与既有两条（`stream-fail`、`hls-stall`）共用同一出口，未新增路径：

```text
PlayerBootstrapper.ets:189-191   watchdog.setOnHlsStall → host.onHlsStreamFailure('hls-stall', -4, ...)
PlayerAssembly.ets:489-490       onHlsStreamFailure → this.failHlsStream(reason, code, message)
PlayerAssembly.ets:225-226       failHlsStream → this.hlsDegrade.fail(reason, code, message)
PlayerAssembly.ets:236-238       hlsDegrade.downgrade → this.downgrade.downgradeFromHlsDetailed(...)  // :80 SourceDowngradeCtl
```

- ijk 显式错误走 `PlayerAssembly.ets:257  onHlsStreamFailure('stream-fail', code, message)`，同样落到上面第 2 行。
- 因此 `'hls-freeze'` 只需作为**第三个 reason 字符串**传入，`HlsDegradeCtl.fail` 内的 HLS 判定、`tried/gen` 去重、`avoid:['SFF']` 全部复用。

## 3. 落点（复用现有唯一路径，不新增决策路径）

```
IjkPlayerBackend.posTimer (500ms) ─► 快照 positionMs/positionSampledAtMs
                                          │
PlayStallWatchdog 新增 1s 采样链（CancelableTimer）
   └─ ProgressFreeze（纯函数，domains/player/watch/ProgressFreeze.ets）
        └─ fire → 复用现有 setOnHlsStall 回调
             → PlayerBootstrapper: host.onHlsStreamFailure('hls-freeze', -4, 'HLS 拉流停滞')
                  → HlsDegradeCtl → SourceDowngradeCtl(avoid ['SFF']) → SmbHttpProxy
```

- **不新增门禁**：`PlaybackCommandCenter` 仍是唯一生命周期门禁；本检测只发 `ErrorRaised(-4, Recoverable)`，与现有 `HLS_STREAM_FAIL` 同型。
- **不自毁**：不 release、不 rebuild、不重启播放器（`DeadlockRecoveryCtl` 的强力重建仍只服务渲染死锁）。
- **不遮罩**：不弹错误层、不盖黑屏。
- **定时器来自 `kernel/clock` 的 `CancelableTimer`**，非裸 `setTimeout`；触发后立即 `dispose`，不重复。

## 4. 影响面（预估，不含实现）

| 文件 | 改动 | 量级 |
|---|---|---|
| `domains/player/watch/ProgressFreeze.ets` | 新增纯逻辑类（无 IO、无定时器） | +45 行 |
| `domains/player/PlayStallWatchdog.ets` | 新增"进度阶段"：1 s 采样链 + 阈值 + fire；`dispose` 覆盖 | +70 行（当前 8.4 KB，远低于 600 行上限） |
| `domains/player/store/PlaybackEventSink.ets` | `PlaybackWatchdogView` 增可选 `progress?(): {posMs:number; ageMs:number; frames:boolean}` | +3 行 |
| `domains/player/PlaybackEngine.ets` | 实现 `progress()`（读 `getSnapshot()`） | +6 行 |
| `PlayerAssembly` / `PlayerBootstrapper` | 0（`setOnHlsStall` 已接线，`PlayerBootstrapper.ets:189`） | 0 |

合规核对：

- **R-12**：不使用裸 `setTimeout` 掩竞态；不自毁；不遮罩。定时器走 `CancelableTimer`。
- **R-18**：不新增第二条降级路径——复用 `HlsDegradeCtl`/`SourceDowngradeCtl` 既有链路；不加 legacy 分支。
- **R-16/R-17**：新类单职责、<60 行单函数；不改 `PlayerAssembly` 装配结构。
- 假阳性防线：S2 帧判据 + seek 沉降 + `ended` + `rate` 缩放；误判后果仅为一次降级（有 `tried/gen` 去重，不会循环降级）。

## 5. 遥测（可判定证据）

- 采样期（仅 `PROBE_LOGS`）：`[PROBE-FREEZE] action=sample stallMs=%d pos=%d frames=%s rate=%f`，每秒一条，`stallMs>2000` 后才打（避免刷屏）。
- 触发：`HLS_STALL_FREEZE pos=%d stallMs=%d gen=%d reqId=%d uri=%s`（warn，单条）。
- 交接后沿用现有：`STREAM_FAIL stage=hls-...`、`DOWNGRADE_OK/FAIL`、`SmbHttpProxy: listening`。
- 验收：真机一次"冻结→8 s 内 `HLS_STALL_FREEZE`→`DOWNGRADE_OK`→`listening`"，且全程无 Fatal/无错误层。

## 6. 验证矩阵

| # | 场景 | 期望 |
|---|---|---|
| 1 | 纯逻辑单测：`ProgressFreeze` 在固定时钟下的 6 组输入（前进/暂停/seek/ended/rate0.5/frames>0） | 只在 freeze+无帧+非豁免时 true |
| 2 | 真机：正常 1080p60 HLS 30 min | 0 次误触发，`[PROBE-FREEZE]` 无 `stallMs>3000` |
| 3 | 真机：注入 6 s 冻结（见 §7 C2） | ~8 s 内 fire → 降级成功 |
| 4 | 真机：SFF 真停服（`ZPACCEL_NOSTART`） | 同上（端到端 S1） |
| 5 | 真机：暂停/拖动中持续 10 s | 0 次触发 |
| 6 | 真机：片尾静止 10 s | 0 次触发 |

## 7. 故障注入开关设计（替代"停 SFF"）

### 7.1 约束先说清

HLS 的 HTTP 传输在 `libijkplayer`（native）内部，ArkTS 层拦不到。客户端若要"真实注入 504/断流"，必须新增本地中继——v400 已按要求物理删除该中继，重加即违反 **R-18（禁双路径）** 与 **R-12（禁为测试加计时器路径）**。因此：

- **A 路径（HLS）真实注入 → 放 SFF 路由侧**（测试实例，生产不启用）；
- **客户端侧 → 放两处已有代码点**：`SmbHttpProxy`（B 路径，真实传输）与 `IjkPlayerBackend` 进度契约（契约级冻结，用于精确验证 WD）。

### 7.2 三个方案（可组合，默认只开一个）

| 方案 | 注入点 | 能注入什么 | 真实性 | 适用 |
|---|---|---|---|---|
| **C2 契约级冻结**（推荐做 WD 验证） | `IjkPlayerBackend.posTimer` 的 `positionMs`/`renderStalled()` 取值处 | 位置不前进 + 帧标志置 0，持续 N ms；播放器状态不变 | 仅"症状"真实，不产生 error | WD 判定、误报验证、离线可控 |
| C1 代理注入 | `domains/smb/**/SmbHttpProxy` 的响应写出处 | 504 / 直接断流 / 半包后停 | **真实传输失败** | B 路径降级、错误码路径 |
| S1 服务端注入 | SFF `/debug/inject`（按 `sid`） | 504 / 停止切片 / 半包停滞 | 最真实（v1 实测场景） | HLS 端到端（测试实例） |

### 7.3 统一契约（三处共用，避免三套开关）

```
FaultSpec {
  match: string          // URL 子串（sid/lease/文件名）
  mode: 'http504' | 'abort' | 'half-stall' | 'freeze'
  afterMs?: number       // 起播后多久开始注入（默认 5000）
  durationMs?: number    // 持续多久（默认 0=一直；>0 自动过期）
  once?: boolean         // 默认 true：每个 gen 只注入一次
}
```

- 客户端入口沿用现有诊断约定：`aa start ... --ps fi=<base64(JSON)>`，落 `ProbeConfig`（与 `dli`/`pl`/`ff` 同族），**只有 `PROBE_LOGS=true` 才解析**；release 构建 `PROBE_LOGS=false` → 整段为死代码。
- 服务端入口：`POST /debug/inject`，仅当环境变量 `ZP_DEBUG_INJECT=1` 时挂载路由（不进生产 run-service 的环境）。
- 每次注入必打一条：`[PROBE-FI] action=inject mode=... url=<masked> afterMs=... durationMs=...`。
- 自动过期：`durationMs>0` 到点自撤；页面/进程重启即清空（不持久化，不落盘）。

### 7.4 影响面与红线

| 方案 | 改动文件 | 量级 | 红线 |
|---|---|---|---|
| C2 | `kernel/probe_config.ets`（+字段）、`IjkPlayerBackend.ets`（posTimer 内 2 处取值） | +25 行 | 不得改 `PlayerState`；不得停 `posTimer`；不得触发 `deadlockHandler` |
| C1 | `SmbHttpProxy.ets`（写出分支） | +30 行 | 仅 `PROBE_LOGS`；不得改端口/lease 语义；不得落在正式路径的默认分支 |
| S1 | `app/debug_inject.py`（新路由，仅 env 开） | +60 行 | 生产 `run-service` 不设 `ZP_DEBUG_INJECT`；不得写缓存/索引 |

- 三处都**不是兜底**：注入是"制造失败源"，不是"掩盖失败"。
- 三处都**不加新定时器**：C2 复用 500 ms posTimer；C1 复用写循环；S1 复用切片生成。
- 三处都**不改判据**：降级仍由现有 `HlsDegradeCtl`/`SourceDowngradeCtl` 决定。

### 7.5 建议顺序

1. 先做 **P0-1-WD**（§1–§5）；
2. 用 **C2** 做 WD 的可重复单点验证（不依赖服务端）；
3. 用 **S1** 在测试实例做一次端到端（真实 504/断流）；
4. **C1** 只在需要验证 B 路径错误码时启用。
