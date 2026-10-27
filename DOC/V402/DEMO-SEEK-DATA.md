# DEMO-SEEK-DATA — libsmb2 直读 seek 数据（首批）

- DEMO 仓：`D:\devlop\ZplayerDemo`（未改主项目、未合并 DEMO）
- 设备：MLR-AL00 / `62BBB25A28201142`（DEMO 独立 bundle `com.example.zplayer`）
- 前置定性：见 `DOC/V402/DEMO-SEEK-PRIMITIVE.md`
- 凭据：运行期通过 `--ps pass <PASS>` 注入，**未写入源码、未提交 git**
- 本轮所有数字均来自本机设备 hilog / `/proc/net/dev`，原始日志在 `E:\temp\opencode\hilog-*.log`、`rx-*.log`

---

## 0. 样本与基线

`ffprobe`（从 SMB 直接读，只读元数据 + 前 45s 关键帧索引）：

| 项 | 值 |
|---|---|
| 路径 | `\\192.168.1.143\Video\AI视频\《小剧场》.mp4` |
| 文件 | 903,071,551 B = **861.2 MiB** |
| 视频 | HEVC **2160×3840（竖屏 4K）**，60 fps，39,344,544 bps（**≈4.92 MB/s**） |
| 音频 | AAC 97 kbps |
| 时长 | 183.136 s |
| GOP | **2.5 s**（关键帧 0/2.5/5.0/…，`kf.csv`） |
| start_time | 0.000000（无容器起始偏移） |

解码/渲染：`native_setup(VideoCodecMode.SURFACE)` + `mediacodec=1`，硬解直写 OHNativeWindow（零拷贝，见 PRIMITIVE §3）。

---

## 1. 链路打通（-10000 的成因与修复）

首轮 `prepare` 立刻返回 `error what=-10000 extra=0`。排除设备/网络后，native 侧日志给出根因：

```
13:12:38.065 41386 SMBPOC: [SMBPOC] event=native_server_fail nativeHttpStart failed:
             connect_share: Session setup failed with (0xc000000d) STATUS_INVALID_PARAMETER
```

即 **SMB 会话认证失败**（不是 seek、不是 decoder、不是 libsmb2 代码问题）。修复方式：启动参数注入凭据，不落盘：

```powershell
hdc shell aa start -a EntryAbility -b com.example.zplayer `
  --ps harness 1 --ps pass <PASS> --ps autorun <mode>
```

修复后同一路径 prepare 成功（`create_to_prepared=390ms`，`create_to_first=531ms`）。

---

## 2. 夹具（DEMO 沙盒内，可脚本触发）

改动文件（均在 `DRAG_HARNESS_BEGIN/END` 区，最终 HAR 移除脚本仍可用）：

| 文件 | 改动 |
|---|---|
| `entry/src/main/ets/fixture/DragHarnessCtl.ets` | 新增 `Single`(5×3s)、`Burst`(20×60ms)、`FarBack`(90%↔10%, 4s) 三种脚本；`DragSample` 增加 `tAck/posAck/bytesFire/bytesComplete/bytesAck/ptsAck`；CSV 增加对应列 |
| `entry/src/main/ets/fixture/DragHarnessPage.ets` | `onInfo` 增加 **10008**（`SEEK_RENDERING_START`）与 **10100**（`ACCURATE_SEEK_COMPLETE`）打点；`bytes()` 读 `getTrafficStatisticByteCount()`；`--ps autorun` 自动跑；`plan_done` 后自动 `csvDump()`；新增 `Burst 20 / Single 5 / FarBack 5` 按钮与**双源按钮**（小剧场 / 01.mp4）；`--ps accurate 1` 打开 `enable-accurate-seek` |
| `entry/src/main/ets/pages/PocCfg.ets` / `EntryAbility.ets` | 新增 `autorun`、`accurate` 两个参数入口 |

已有的 Slider（Begin/Moving/End → `SeekArbiter`）与 `OnSeekCompleteListener` 保持不变；`SeekArbiter` 仍是 150 ms 合并窗。

三种“seek 完成”信号在本轮的分工（详见 PRIMITIVE §2）：

- **600 / `MEDIA_SEEK_COMPLETE`**：demuxer 定位受理，`ff_ffplay.c:4221`，早于任何帧。
- **10008 / `MEDIA_INFO_VIDEO_SEEK_RENDERING_START`**：新 serial 首帧**上屏后**，`ff_ffplay.c:1240-1257`。本文件把它当作“首帧渲染”的唯一原语。
- **10100 / `MEDIA_INFO_MEDIA_ACCURATE_SEEK_COMPLETE`**：仅在 `enable-accurate-seek=1` 时出现，携带 `pts*1000`。

---

## 3. 原始打点（证明来源）

### 3.1 单次 seek（`Single 5`，accurate=off，5 次，间隔 3s）

```
13:39:02.992 fire seq=1 target=27470 wait=0 gen=3 bytes=64316006
13:39:03.007 seek_complete seq=1 target=27470 pos=27470 complete_dt=15 bytes=64316006
13:39:03.011 frame seq=1 drop=19 pos=27474
13:39:03.106 seek_render seq=1 ack_dt=114 pos=25276 bytes=68918782
...
13:39:03.105 seq=2 ack_dt=113 | 13:39:06.101 seq=3 ack_dt=108
13:39:09.093 seq=4 ack_dt=101 | 13:39:12.112 seq=5 ack_dt=119
```

### 3.2 连续快拖（`Burst 20`，20 请求 / 60 ms）

```
13:34:33.858 plan_start mode=burst20 steps=20 scripted=1200ms
13:34:33.920 fire seq=1 target=9157           → 10008 @ +54ms
13:34:34.131 fire seq=2 target=35181 (req=5)  → 10008 @ +128ms
13:34:34.314 fire seq=3 target=61206 (req=8)  → 10008 @ +187ms
13:34:34.484 fire seq=4 target=87231 (req=11) → 10008 @ +203ms
13:34:35.060 plan_done steps=20               → 20 请求只产生 8 次 fire
13:34:36.561 summary n=8 noAck=2 ack_p50=187 ack_p95=204 ack_max=208
```

### 3.3 50 连快拖（`Fast`，50 请求 / 40 ms）

```
13:29:17.264 plan_start steps=50 scripted=2000ms
13:29:19.266 plan_done steps=50
13:29:20.768 summary n=14 ok=13 stale=1 noAck=7 p50=65 ack_p50=168 ack_p95=187 ack_max=317
```

### 3.4 往返风暴（`BackForth`，100 请求 / 80 ms，10 轮）

```
13:32:09.812 plan_start steps=100 scripted=8000ms
13:32:17.816 plan_done steps=100
13:32:19.317 summary n=48 ok=48 stale=0 noAck=3 p50=91 ack_p50=188 ack_p95=222 ack_max=228
```

### 3.5 远距往返（`FarBack`，90%↔10%，5 次，间隔 4s）

```
13:36:10.478 fire seq=1 target=164822 → 10008 @ 10.603 ack_dt=126
13:36:14.477 fire seq=2 target=18314  → 10008 @ 14.567 ack_dt=90
13:36:18.477 fire seq=3 target=164822 → 10008 @ 18.579 ack_dt=102
13:36:22.478 fire seq=4 target=18314  → 10008 @ 22.568 ack_dt=90
13:36:26.478 fire seq=5 target=164822 → 10008 @ 26.581 ack_dt=104
```

### 3.6 精确 seek（`Single 5` + `accurate=1`）

```
13:26:55.747 seek_complete seq=1 bytes=65901191
13:26:56.806 accurate_pts seq=1 target=27470 pts=26650 delta=-820
13:26:56.815 seek_render seq=1 ack_dt=1079
...
delta: -820 / -858 / -861 / -702 / -802 ms
```

---

## 4. 数据表

### 4.1 端到端：seek 发起 → 首帧渲染（10008）

| 场景 | n（fire） | 合并 | ack_p50 | ack_p95 | ack_max | 对比主项目 184–585ms |
|---|---|---|---|---|---|---|
| Single 5（孤立） | 5/5 | 0 | **113 ms** | 114 | **119** | 全部低于下限 |
| Slow 20（250ms 间隔） | 20/20 | 0 | **147 ms** | 205 | **242** | 全部低于下限 |
| Burst 20（60ms） | 8/20 | 12（60%） | 187 ms | 204 | 208 | 落在区间下沿 |
| Fast 50（40ms） | 14/50 | 36（72%） | 168 ms | 187 | **317** | 落在区间内 |
| BackForth 100（80ms） | 48/100 | 52（52%） | 188 ms | 222 | 228 | 落在区间下沿 |
| FarBack 5（远距、孤立） | 5/5 | 0 | 102 ms | 104 | 126 | 全部低于下限 |
| Single 5（accurate=1） | 5/5 | 0 | **1050 ms** | 1051 | 1079 | 远高于上限（见 §5.4） |

### 4.2 分层拆解：定位完成 vs 上屏

| 场景 | demuxer 定位 `seek_complete` p50 / max | 首帧 `10008` p50 / max | 差值（解码+合成+消息） |
|---|---|---|---|
| Single 5 | 12 / 15 ms | 113 / 119 ms | ≈ 101 ms |
| Slow 20 | 19 / 43 ms | 147 / 242 ms | ≈ 128 ms |
| Burst 20 | 48 / 70 ms | 187 / 208 ms | ≈ 139 ms |
| Fast 50 | 60 / 94 ms | 168 / 317 ms | ≈ 108 ms |
| BackForth | 86 / 102 ms | 188 / 228 ms | ≈ 102 ms |

**结论（耗时侧）**：`FFP_MSG_SEEK_COMPLETE`（600）只值 12–102 ms，不能当 ack；真正的用户可感耗时要看 10008，两者相差约 **100–140 ms**（解码/上屏/消息排队），这正是主项目“无 ack + 时间窗仲裁”会把 seek 判早/判晚的区间。

### 4.3 落点偏差（目标 vs 实际）

| 场景 | 采样口径 | 偏差范围 | 中位 | 对比主项目 −6.9s/+3.5s |
|---|---|---|---|---|
| Single 5 | `getCurrentPosition()` @10008 | **−2194 … −2053 ms** | ≈ −2131 | 单 GOP（2.5s）内，量级小 3 倍 |
| Slow 20 | 同上 | −2052 … +227 ms | −955 | 单 GOP 内 |
| Burst 20 | 同上 | −8654 … +98 ms | −1487 | 末次 seek 偏差含竞态（见 §5.5） |
| BackForth | 同上 | −1455 … +35 ms | −1187 | 单 GOP 内（→173.9s 的 seek 稳定 −1.15~−1.22s，→9.1s 的 seek 稳定 −1.33~−1.37s） |
| Single 5（accurate=1） | `10100` 自带 pts | **−861 … −702 ms** | ≈ −820 | 优于主项目，但仍非 0 |
| 全部（`posComplete`） | seek 回调里的 pos | +0 … +7 ms | ≈ +1 | **不可用**：是 seek_pos 回显（见下） |

**重要口径说明**：`posComplete`（`OnSeekComplete` 里立刻 `getCurrentPosition()`）几乎恒等于目标值，因为 IJK 在 seek 受理时就把时钟设成了 `seek_pos`（`ff_ffplay.c:4178`），它只是回显，不能当落点。真实“首帧时刻的播放位置”只能取 10008 到达时的 `getCurrentPosition()`（`posAck`）或 accurate 模式的 `10100 pts`。

### 4.4 读放大（协议层 / 网络层）

`getTrafficStatisticByteCount()` = libsmb2 `url_read` 交付给 FFmpeg 的字节（含 ring 命中，**不含**预取线程尚未交付的网络读）；`wlan0 rx_bytes` = 真实网卡收包。

| 场景 | 协议层交付 | 网络层 rx | 网络/协议 | 视频全片 861.2 MiB 的倍数 |
|---|---|---|---|---|
| Slow 20（20 seek，5s） | **312.6 MiB** | — | — | 0.36× |
| Burst 20（20 请求/8 seek，1.2s） | **74.3 MiB** | **111.9 MiB**（+2s 内 161 MiB） | **1.51×** | 网络 0.13×（+2s 0.19×） |
| Fast 50（50 请求/14 seek，2s） | 62.7 MiB | — | — | 0.07× |
| BackForth 100（100 请求/48 seek，8s） | **678.0 MiB** | **1171.6 MiB** | **1.73×** | **1.36×** |

按“每次被受理的 seek”摊：

| 场景 | 协议层/seek | 网络层/seek | 主项目 B 链 880 MiB / 30 Range = 29.3 MiB/次 |
|---|---|---|---|
| Single 5 | 4.4–9.3 MiB（中位 5.8） | — | 更低 |
| Burst 20 | 5.75 MiB（按 ack 中位） | 14.0 MiB | 更低 |
| BackForth | 6.7 MiB | **24.4 MiB** | 略低（−17%） |
| FarBack（孤立远距） | 6.5–11.4 MiB | **≈30 MiB/次** | 持平 |

**理论所需**（用于算放大）：GOP=2.5s、码率 4.92 MB/s，一次 seek 至少要读“上一关键帧 → 目标帧”，即 `4.92 MB/s × (|target−落在的关键帧| + 1/60s)`。用实测 `|target−posAck|` 代入：

| 场景 | 理论所需（每次） | 实测协议层 | 实测网络层 | 放大（网络/理论） |
|---|---|---|---|---|
| Single 5 | 10.2–10.9 MiB | 4.4–9.3 MiB（0.4–0.9×） | — | ≤1（缓存命中） |
| Burst 20 | ≈10 MiB | 5.4 MiB | 14.0 MiB | ≈1.4× |
| BackForth | ≈8.5 MiB | 6.7 MiB | 24.4 MiB | ≈2.9× |
| FarBack（远距） | ≈10.5 MiB | 6.5–11.4 MiB | ≈30 MiB | **≈2.9×** |

---

## 5. 补充四项（T-13 / T-20 追问）

### 5.1 「环重置税」：把网络层和渲染层拆开

**隔离样本**（FarBack，4s 静默间隔，网卡基线为 0），`wlan0 rx` 对 seek 时刻（13:36:10.478 / 14.477）对齐：

| 时间轴（相对 seek，实测采样点） | far forward（→164.8s，跨 ~160s） | far backward（→18.3s，回退 ~147s） |
|---|---|---|
| +104 ms | 0.0 MB（+111 ms 采样） | 0.52 MB |
| +179 ms | 6.3 MB（+199 ms 采样） | 10.49 MB |
| +251 ms | 6.3 MB（+199 ms 采样） | 19.44 MB |
| +348 / +379 ms | 25.7 MB（+348 ms） | 30.33 MB（+379 ms） |
| +410 ms | 29.8 MB | 30.33 MB（停止） |

- **首帧（10008）**：126 ms（forward）/ 90 ms（backward）。
- **4 MB 预取完成**：**≈110–140 ms**（由 0.5→10.5 MB / 104→179 ms 的两点插值，Sample≈70 ms 分辨率；瞬时速率 ≈133 MB/s，4 MiB / 133 MB/s ≈ 31 ms + 70–100 ms 首包延迟）。
- **16 MB 环重置完成**：**≈220–250 ms**（同为插值，非直接打点）。
- **单次远距 seek 实际拉取**：**≈30 MiB**，而理论只需 ≈10.5 MiB → 网络放大 **≈2.9×**，浪费 ≈20 MiB/次。

**归因（关键）**：
1. `libsmb2_read` 在窗口外走 256 KiB 块缓存（`libsmb2.c:648-672`），首帧只需要 1–2 个块（0.25–0.5 MiB）即可解码；这解释了 **首帧 90 ms 早于 4 MB 预取完成 130 ms**。
2. 预取线程按 **4 MiB 整块** pread 后才挂进 ring（`libsmb2.c:219-248`），所以 ring 的“可用数据”以 4 MiB 为粒度；但它不挡首帧，只影响后续连续读。
3. 环重置触发点：前跳过 ring 尾立即 `ring_reset`（`libsmb2.c:642-646`）；回退则累计块缓存未命中 ≥1 MiB 才重置（`libsmb2.c:670-671`）。重置会丢弃最多 16 MiB 已预取数据（`libsmb2.c:61`、`165-175`）。

**结论**：T-13 的“每次 seek 重置 16 MB 环”成立，但它的代价是**带宽税（≈30 MiB/次、2.9×）**，不是**首帧延迟税**；首帧由 256 KiB 块缓存兜底。风暴下带宽税会叠加：BackForth 48 次 seek 打了 **1171.6 MiB ≈ 1.36 个全片** 的网卡流量（网络/协议 1.73×）。

### 5.2 SeekArbiter stale 丢弃有效性（快拖乱序）

`Fast 50`（40 ms 请求间隔、150 ms 合并窗）：

| 指标 | 值 |
|---|---|
| 请求数 → fire 数 | 50 → **14**（合并 36，72%） |
| fire 间隔实测 | 约 160–190 ms（≥150 ms 窗 + 处理开销） |
| 实际渲染（`frame`） | 13/14 |
| 拿到 10008 | 7/14；其余 7 次被后一次 seek 取代（serial 覆盖，`ff_ffplay.c:4181`） |
| `stale_gen` 标记 | 9/14（complete 回调返回时 arbiter 的 gen 已推进） |
| 渲染位置序列（`posFrame`） | 7675 → 22620 → 36075 → 49536 → 62989 → 76438 → 89896 → 103345 → 116798 → 130262 → 143708 → 157170 → 170629 |
| 位置回跳 | **0 次**（严格单调） |
| 迟到回调 | seq11 的 10008 在 fire 后 **317 ms** 到达（比 seq12 的 fire 晚 97 ms），但画面仍前进到 157170 |
| 黑帧 | `noFrame=0`；但本轮未做逐帧截图，**不能据此断言无黑帧** |

**关键判断**：
- 150 ms 合并窗确实吃掉了 40 ms 抖动（72% 请求未下发 native），把 native seek 压到 14 次/2s。
- `stale_gen` 只是 DEMO 侧**标签**，不会撤销 native seek；真正的“旧的不要”由 ffplay 的 serial 覆盖完成。因此“stale 丢弃”并不靠 `|pos−target|≤3000` 匹配来纠正画面。
- 观察到“迟到 ack”，但**没有观察到画面回跳**：迟到的 10008 是消息，不是帧；`latest_video_seek_load_serial` 的 `__atomic_exchange_n(...,-1)`（`ff_ffplay.c:1248-1250`）保证每个 serial 只 ack 一次。
- 但 10008/600 **都不带 seek id**，fast 风暴下“某次 ack 属于哪次 seek”本质上不可判定（本轮按“最新未 ack 样本”绑定，seq11 的 317 ms 很可能就是这种错绑）。这是 T-20 票据机制要解决的问题，DEMO 数据只能证明“画面没回跳”，不能证明“ack 可精确归属”。

### 5.3 读放大正面对比（直读 vs 代理）

| 口径 | DEMO 直读 | 主项目 B 链（代理，用户给定） | 判定 |
|---|---|---|---|
| 20 连拖（burst20，8 次 native seek） | **111.9 MiB**（1.2s 窗口） / 161 MiB（+2s） | 30 次 Range / **880 MiB** | DEMO 更低 |
| 48 次 seek 风暴（8s） | **1171.6 MiB**（1.36× 全片） | 880 MiB（30 Range） | DEMO 总量更高（seek 次数 1.6×） |
| 每次受理 seek 的网络字节 | 14.0（burst）/ 24.4（backforth）/ ≈30（远距） | 29.3 MiB/Range | 基本持平，远距 seek 略高 |
| 网络/协议层比 | 1.38×（burst）/ **1.73×**（backforth） | 未知 | 环重置造成 38–73% 冗余 |

**结论**：libsmb2 的异步预取不是“更费带宽的 Range 碎片化”，它单次操作与代理同量级；但在**高频反向/远距**拖动下，16 MiB 环重置会把它推到 **>全片流量**，这是代理 Range 模式没有的额外成本。是否“更省”取决于 N 与方向分布，不能一句话定死。

### 5.4 落点 PTS 偏差极值（决定 T-20“唯一真源”）

| 方法 | 偏差 | 可信度 |
|---|---|---|
| `OnSeekComplete` 内 `getCurrentPosition()` | +0…+7 ms | **不可用**（seek_pos 回显） |
| 10008 时刻 `getCurrentPosition()`（accurate=off） | −2194…+227 ms；孤立远距稳定 −2.05~−2.19 s | 中：受音频/主时钟影响，但量级与 GOP 一致 |
| 10100 `pts`（accurate=1） | −861…−702 ms | 中高：IJK 自带 pts，但含音频握手偏置 |
| 逐帧截图对比（J8） | **未做** | — |

- GOP=2.5 s，因此 non-accurate 落点必然落在 **[target−2.5s, target]**；实测 −2.19 s 正好落在 GOP 起点附近，符合“关键帧对齐”而非“seek 失败”。
- 与主项目 −6.9s/+3.5s 相比，DEMO 的落点误差**被限制在一个 GOP 内**，量级约为主项目的 1/3–1/2；但 DEMO 也没有做到帧级精确（accurate 模式仍有 −0.8 s 固定偏置）。
- **对 T-20 的直接含义**：IJK 的 `getCurrentPosition()` 不能作为唯一真源（它回显目标）；可作真源的候选是 `10008 时刻的时钟` 与 `10100 的 pts`，两者都带偏置，必须做零点标定。J8 截图对比是唯一无偏口径，本轮未做。

### 5.5 Burst/风暴中的“谁被丢弃”

- `Burst 20`：20 请求 → 8 fire → 6 个 10008；2 个 fire 在其首帧出现前被下一次 seek 的 serial 覆盖（记为 `no_ack`）。
- `BackForth`：100 请求 → 48 fire → 45 个 10008（3 个 no_ack）。
- `Fast 50`：50 请求 → 14 fire → 7 个 10008（7 个 no_ack）。
- **被合并**的请求从未下发 native（SeekArbiter 层拦截）；**被覆盖**的 fire 已下发 native，但不会产生 10008（serial 更新）。两类在数据上可区分。

---

## 6. 结论：libsmb2 直读 vs 主项目 IJK HTTP Range 代理

| 维度 | DEMO libsmb2 直读 | 主项目（用户给定） | 谁更好 |
|---|---|---|---|
| 首帧耗时（10008） | 孤立 113 ms / p95 205–222 / 极值 317 | 184–585 ms | **DEMO** |
| 定位受理耗时 | 15–95 ms | — | 仅参考（不是 ack） |
| 落点精度 | 单 GOP 内（−2.2s…+0.2s）；accurate −0.8s | −6.9s…+3.5s | **DEMO** |
| 20 连拖读放大 | 111.9 MiB（0.13× 全片） | 305 MiB / 880 MiB（B 链） | **DEMO** |
| 逐 seek 网络放大 | 2.9×（远距）、1.73× 冗余 | 未知 | 视 N 而定，非全面占优 |
| ack 可归属 | 10008 无 id，风暴中不可判定 | 无 ack | 都不满足 T-20 |

**回答“是否优于主项目 IJK 的 HTTP Range 代理”**：在**首帧耗时、落点精度、20 连拖流量**三项上，DEMO 直读优于给定主项目数字；在**每次 seek 的网络放大**上并不占优（远距 ≈2.9×，风暴总量可超全片）；两者都**没有**可归属的 seek ack。

**可外推到主项目**（依 T-14“FFmpeg/IJK 功能源码、libsmb2 二进制同源”前提，本轮未复核主项目）：
1. `url_seek` 纯改偏移、数据延迟到 `url_read`（`libsmb2.c:521-559`）——同源即同语义。
2. 256 KiB 块缓存兜底首帧、4 MiB 预取粒度、16 MiB 环重置（`libsmb2.c:61-66, 219-248, 642-671`）——同源即同行为；但 0020 补丁的**调参常量**不是同源保证。
3. IJK 的 10008 上屏 ack 链（`ff_ffplay.c:1240-1257`）与 600 的“早发”性质——需主项目 NAPI 确实透传 `10008`，本轮回溯不到。
4. 落点误差被 GOP 限制（2.5s）——由文件与 IJK seek 语义决定，可外推。

**仅 DEMO 有效**：
1. `SeekArbiter` 150 ms 合并窗与 gen 判旧（DEMO 策略层）。
2. HTTP bridge / `smb_io.cpp` 的 Range 语义（smb:// 路径不用它）。
3. 本机网络条件（SMB 服务器 `192.168.1.143`，实测 ≈133–190 MB/s 线速、RTT 23–29 ms）——所有绝对耗时依赖它。
4. SURFACE 零拷贝与主项目“同一套”仅由 T-14 前提支撑，本轮未对照主项目源码。

**未完成/不可宣称**：
1. J8 逐帧截图对比（真落点）。
2. ack 与 seek 的精确归属（缺 id）。
3. 黑帧的视觉确认（仅有 `noFrame=0`）。
4. 主项目侧的对照测量（本轮禁止读主项目）。

---

## 7. 复现

```powershell
# 1) 构建/安装（harness 已在 main_pages 中启用）
hvigorw.bat assembleHap --mode module -p product=default -p buildMode=debug --no-daemon
hdc install -r entry\build\default\outputs\default\entry-default-signed.hap

# 2) 采样网卡（可选）
hdc file send rxsample.sh /data/local/tmp/rxsample.sh
hdc shell "rm -f /data/local/tmp/rx.stop /data/local/tmp/rx.log; nohup sh /data/local/tmp/rxsample.sh >/dev/null 2>&1 &"

# 3) 跑场景（mode: single5 | slow | fast | burst20 | backforth | farback）
hdc shell aa start -a EntryAbility -b com.example.zplayer `
  --ps harness 1 --ps pass <PASS> --ps autorun backforth

# 4) 取数
hdc shell hilog | Select-String DRAGHARNESS     # CSV 行在 plan_done 后自动输出
hdc file recv /data/local/tmp/rx.log rx.log
```
