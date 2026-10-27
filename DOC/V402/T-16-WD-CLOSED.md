# T-16 · P0-1-WD 关闭归档 + P1 登记 + 4K 同源 A/B（V402-056）

> 性质：证据归档（0 代码改动；主仓改动仅存在于分支 `topic/v402-p0-1-wd` @ `093523f`，**未合入 main**）
> 设备：1600×2560 / API 26 真机（序列号不入库）
> SFF：`<SFF_HOST>:8612`（ZP-Accelerator，encoder=hevc_nvenc）
> 时间：设备本地时钟；当日发生一次系统时间跳变（02:0x → 09:3x），本文各段内时间自洽
> 脱敏：0 IP / 0 序列号 / 0 凭据；样本与路径以代号或 `_v402tests` 临时名表示

## 0. 索引

| 章节 | 一句话 |
|---|---|
| 1 | WD 关闭：真实 4K 样本因能力闸失败已归因为样本问题；合规 1080p 样本端到端降级成功 |
| 2 | 端到端时间线（wd_hevc1080.mp4）：freeze → 12 s 判定 → 代理 listening → DOWNGRADE_OK → 续播 |
| 3 | UI 无污染：fire 后 +0/+1/+4 s 三连拍 OCR 仅视频画面，无错误层 |
| 4 | 4K 同源 A/B：同一文件 2160×3840@30 / GOP 5 s，A=HLS 10.7% vs B=代理 96.2%（9×） |
| 5 | P1-A / P1-B 登记 |
| 6 | 分支与合规、恢复状态、临时产物 |

## 1. 关闭依据（两条独立结论）

### 1.1 4K HEVC60 的 DOWNGRADE_FAIL = 能力闸，样本问题（非新缺陷）

`09:35:08.946` 失败前 8 行原始 hilog（节选，逐字）：

```text
09:35:08.862 … ZPLAYER/SmbHttpProxy: served start=902547263 declared=524288 sent=524288
09:35:08.946 … ZPLAYER/OpenAndPlay: SOURCE_UNSUPPORTED reason=hevc-4k-highfps
              codec=video/hevc 2160x3840 fps=60 uri=AI视频/<4K_SAMPLE>
09:35:09.097 … ZPLAYER/Lease: LEASE_REVOKE_REQ leaseId=L1 reason=replace-play
09:35:09.099 … ZPLAYER/SourceDowngrade: DOWNGRADE_FAIL trigger=hls-freeze outcome=PrepareFailed
```

判定：降级目标（SMB 直读）被 `SourceCapabilityGuard` 以 `hevc-4k-highfps` 拒绝，`outcome=PrepareFailed` 即该守卫的 `SOURCE_UNSUPPORTED`；WD 与降级分支本身正常。**记样本问题**。

### 1.2 合规样本端到端成功（唯一关闭条件）

样本：`_v402tests/wd_hevc1080.mp4`（HEVC 1920×1080@30 / 150 s / 75 MB，`testsrc2` 生成；未改任何业务文件）。

## 2. 端到端时间线

```text
09:36:06.7  WATCHDOG_ARM hls=true freezeView=true reqId=1 uri=_v402tests/wd_hevc1080.mp4
09:36:06.7  SFF /health: active=1 method=transcode segments=75
09:36:24    （测试动作：置 NOSTART 标志 + 停 uvicorn）
09:36:40.781 freeze: stalled pos=31833 stallMs=1002 state=Playing frames=true
09:36:51.801 HLS_STALL_FREEZE stallMs=12021 gen=1 reqId=1
09:36:51.922 SmbHttpProxy: listening 127.0.0.1:41013 size=78234361
09:36:52.938 SourceDowngrade: DOWNGRADE_OK trigger=hls-freeze
09:36:52.990 SmbHttpProxy: served start=0 declared=78234361 sent=1310720
09:38:35.039 SmbHttpProxy: served start=44 declared=78234317 sent=78234317
```

后置证据：
- `<SFF_HOST>:8612` 连通性 False、`python` 进程数 0（SFF 确实停着）；
- 画面差分 bbox 非空（在播）；OCR 计数 00:01:57.233 → 00:02:16.033（19 s 内前进 19 s）。

结论：**P0-1-WD 关闭**——静默冻结被判定、事件汇入既有降级漏斗、新实例进入 Playing、代理持续服务。

## 3. UI 无污染（fire 时刻三连拍）

| 偏移 | OCR 全文（节选） |
|---|---|
| fire +0 s | `TabletShare / _v402tests / wd_hevc1080.mp4 / 00:00:01.033` + 视频画面 |
| +1 s | 同上，`00:00:02.700` |
| +4 s | 同上，`00:00:06.300` |

三张均无「错误 / 重试 / 不支持」文案、无遮罩、无弹层。`ErrorRaised(-4, Recoverable)` 未升 Fatal 层。

## 4. 4K 同源 A/B（DEMO 存亡判据）

**同源定义**：同一文件字节（`sha` 未单独记录，按 `size=609,346,839` + 同一拷贝来源核对）。源：AI 视频目录下的 `2160×3840@30 / h264 / GOP 5.0 s / 176.2 s`，拷贝为 `_v402tests/wd_4k_same.mp4`（与源同字节）。GOP 5 s > 2 s → 服务端策略 `transcode`（HLS）；h264 4K30 设备可解 → SMB 兜底可播。

| 项 | A = HLS（transcode，**输出 608×1080**） | B = SMB 代理（SmbHttpProxy，原始 2160×3840） |
|---|---|---|
| CPU（`top` 进程级单核，3×30 s） | 11.1 / 10.3 / 10.7% → **中位 10.7%** | 96.2 / 85.1 / 157% → **中位 96.2%** |
| PSS（`hidumper --mem` Total） | 148,601 / 155,895 / 159,484 kB → 中位 **152.2 MB** | 328,874 / 360,163 / 385,780 kB → 中位 **351.7 MB** |
| RES | 220 / 230 / 232 M | 363 / 408 / 410 M |
| fd / 线程 | 20 / 52–54 | 20 / 58–62 |

B 链证据（同一文件，播放中降级产生）：
```text
09:55:14    （停 SFF：NOSTART + 停 uvicorn）
09:56:09.947 freeze: stalled pos=141924 stallMs=1001 state=Playing frames=true
09:56:20.956 HLS_STALL_FREEZE stallMs=12010 gen=1 reqId=1 uri=_v402tests/wd_4k_same.mp4
09:56:21.218 SmbHttpProxy: listening 127.0.0.1:41013 size=609346839
09:56:22.252 SourceDowngrade: DOWNGRADE_OK trigger=hls-freeze
09:57:49.222 SmbHttpProxy: served start=34525108 declared=574821731 sent=1650764
（同时刻 8612 连通 False、python=0）
```

**A 侧输出分辨率（补标注）**：客户端 `/stream/start` 只发 `{uri, method}`，不带 `maxWidth/maxHeight`；服务端取默认 `ZP_MAX_WIDTH=1920`、`ZP_MAX_HEIGHT=1080`，`encoder.fit(2160, 3840, 1920, 1080)` → **608×1080**。直接证据：SFF 缓存分片名为 `hevc_nvenc-608x1080-transcode-pts1-seg-*.ts`。

因此本轮 A/B **不是同分辨率对照**：A 解码 0.66 MP，B 解码 8.29 MP（像素比 **12.6×**），B 另付代理拷贝税。**9.0× CPU 差距里含分辨率因子**；本数据不能用来说明"同分辨率下代理比 HLS 贵 9 倍"。要剥离该因子，需让 A 输出 4K（客户端上报 maxWidth/maxHeight 或服务端放开上限），属另开单。

对比结论：
- 同源 4K：B/A = **9.0× CPU**（96.2 / 10.7）、**2.31× PSS**（351.7 / 152.2）。代理链路成本成立，非采样口径问题（两者同为 `top` 进程级、3×30 s、中位）。
- 历史 `~160%`（HEVC 2160×3840@60 走代理）与本轮 `96.2%`（h264 2160×3840@30 走代理）**同编码层级但不同 codec/fps**；峰值 157% 已进入历史量级。**非同源历史值，不用于否认或确认 160%**；本轮给出的是"同文件、同 codec、同 fps"的干净对照。
- 木桶短板：B 侧 CPU/PSS 显著高于 A，是 DEMO（A 路径直读）存亡判断的关键数据；是否保留 DEMO 由裁判裁定，本文只出数。

## 5. P1 登记（未闭环）

| ID | 问题 | 证据 | 整改方向 |
|---|---|---|---|
| **P1-A** | `ProgressFreeze` 沉降退出仍用 `framesAlive`（`:94`）；IJK fps 计数冻结后残留非 0，导致沉降几乎立即退出，12 s 上限与 `seek-settle-exceeded` 实际不可达，seek 保护退化为"8 s 窗口恰好大于 3 s 沉降"的时序巧合 | 本次真机：`freeze: stalled` 行 `frames=true`；`seek-settle-exceeded` 全程 0 条 | 沉降退出改纯位置判据（去掉 `|| s.framesAlive`），并复核 `/frame` 计数可信度 |
| **P1-B** | 降级后从 0 起播，未续播原位置（冻结时 pos=31.8 s；降级后画面计数 00:00:01.0 起；4K 同源那轮冻结 pos=141.9 s，降级后 `served start=34525108` 亦为新会话起点） | §2 / §4 时间线 | 产品决策：是否在降级 prepare 后 seek 回原位置（涉及 resume 语义与首帧延迟权衡） |

## 6. 下一改动单排期（未做）

| 项 | 内容 |
|---|---|
| 单号 | `P1-A-1`（建议） |
| 范围 | `ProgressFreeze` 沉降退出改纯位置判据：删除 `:94` 的 `|| s.framesAlive`，仅保留 `delta > minAdvance`；同步复核 `seek-settle-exceeded` 是否恢复可达 |
| 提交 | 单独 commit（不与此前 WD 提交混签），同一分支或新分支，仍需 repo-guard 门禁 |
| 验证矩阵 | seek 5 s / 7.9 s（<8 s 不误报）/ 12 s+（触发 `seek-settle-exceeded` 并按 8 s 窗口判定）三档真机 + 纯逻辑单测三档 |
| 前置 | 本单不阻塞 4K 同源"剥离分辨率因子"的另行评估 |

## 7. 分支、合规与恢复

- WD 代码：分支 `topic/v402-p0-1-wd`，commit `093523f`（9 文件 / +446 −10；`PlayerAssembly` 0 改动）。
- `repo-guard`（`scripts/repo-guard.mjs`）实测：`files=664 fail=0 warn=1`。warn 唯一来源 = `[tracked-log] 13 个历史日志仍在版本库`，清单（全部为 v397/v400 既有归档证据，**非本次误入**）：
  `DOC/evidence/v397/{align-t10-upload.log,align-t10.log,closest-after-fix.log,e2e-local-0.12s.log,e2e-local-1.3s.log}`、
  `DOC/evidence/v400/perf/{A1-h264-smb-resume.log,A2-h264-smb.log,A3-h264-smb.log,B1-av1-cold.log,B2-av1-warm.log,B3-av1-resume.log,C-seek-storm.log,D-crop-triple.log}`。
- 处置：**未移除**（移除会把这些证据从后续 clone 的 HEAD 中删除，只有历史里能找到；不属"日志误入"）。故 `warn` 仍为 1；**`093523f` 未合入 main**，待裁决：接受 warn=1 合入，或先 `git rm --cached` 这 13 个归档日志（可回滚）后再合。
- 本轮未改任何代码；主仓工作区仅剩先存改动 `tests/unit/tests/stream-failure.test.ts`（非本轮）。
- 设备：已 force-stop；SFF 已恢复生产循环（`/health` 200、`ZPACCEL_NOSTART` 不存在）。
- 临时产物（SFF 媒体盘 `_v402tests/`）**已删除**：`wd_hevc1080.mp4`（75 MB）、`wd_4k_same.mp4`（609 MB），合计 684 MB 已释放。
- 删除后目录仍有 18 个**既有**文件（`b1.json/backend.log/fault_proxy.py/proxy.log/…`，非本轮产生，未触碰）；如要整目录清空需另确认。
- 0 IP / 0 序列号 / 0 凭据；样本目录名以 `<4K_SAMPLE>` 或 `_v402tests` 表示。
