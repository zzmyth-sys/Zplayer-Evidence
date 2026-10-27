# 0020-RING-RISK-AUDIT — SMB 直读读模型（868 行）代码级隐患审计

- 日期：2026-10-27
- 审计对象：`native/patches/0020-smb2-async-prefetch-ring.patch`（949 行）及其落地源码
  - `native/src/ohos_ffmpeg/libavformat/libsmb2.c`（868 行，patch 新增文件本体）
  - `native/src/ohos_ffmpeg/libavformat/seek.c`（1 个 hunk，175-194 行）
- 标注约定：**[观察]** = 本轮直接读到的源码行；**[推断]** = 由观察或前提推出；**[前提]** = 用户提供的 T-14 / 主项目事实，本轮未验证
- 本轮未读：主项目源码（`SeekGate` / `PlaybackCommandCenter` 只按前提引用，不做事实断言）

---

## 0. 结论速览（先给判定，后给证据）

| 维度 | 判定 | 一句话 |
|---|---|---|
| 并发 / 同步 | **需改架构**（非"加一把锁"可解） | ring 账本本身是正的（无死锁、无越界、旧数据不会写进新环）；但**前向 seek 没有"首字节"通路**，必须等腰斩一整块 4 MB |
| 内存 / 生命周期 | **不泄漏，但不可接受** | `ring_reset` 不释放 16 MB 是"复用"而非泄漏；`destroy_smb2` 释放完整；问题在 22 MB/实例固定占用 + 无缩容，以及 close 可被在途 `pread` 卡住 |
| 异常 | **不会因错误码崩溃，但会永久失效** | 错误码映射正确、不再吞错；但 `err` 一旦置位不可清除（seek 也不清），一次瞬时错误 = 该 URLContext 永久读失败；另有一处 open 失败路径的空指针解引用 |

**合入判定：不具备直接运行的安全性。** 详见第 7 节必修清单（P0 两处 + P1 两处）。

---

## 1. 线程模型还原（回答"三者如何同步"）

### 1.1 三个 actor

| actor | 入口 | 触碰 ring 账本 | 发起网络 I/O | 备注 |
|---|---|---|---|---|
| 预取线程（唯一生产者） | `libsmb2_prefetch_thread` 177-254 | 持 `lock` | 持 `net_lock`，每次 ≤1 MB（149-153） | 全文件唯一写 `ring` 的地方（247） |
| 读线程（唯一消费者） | `libsmb2_read` 561-690 | 持 `lock`（除 655-657 一段） | 仅块缓存路径（656） | IJK 里就是 read_thread |
| seek | `libsmb2_seek` 521-559 | **完全不碰锁** | 无 | 只写 `libsmb2->pos`（556） |

### 1.2 同步原语的职责划分

- `lock`（91）：保护 `ring_base / ring_len / consume_off / gen / eof / err / pos / blocks`。
- `net_lock`（92）：串行化唯一的 SMB 会话，防两条线程同时 `smb2_pread` 打断同一 TCP 流。
- `cond`（93）：数据可读 / 空间可写 的双向通知；读侧 200 ms `timedwait` 轮询 + 25 s 硬超时（629-638）。

### 1.3 三个明确结论

1. **[观察] 无经典锁序死锁。** 所有持 `lock` 的临界区都不取 `net_lock`：预取在 232 行先解锁再 `pread`，块缓存在 293 行先解锁再 `pread`，`destroy_smb2` 也在 328 行先解锁再 `join`。不存在 `lock→net_lock` 的嵌套，ABBA 死锁不成立。
2. **[观察] 但存在三处无上界的阻塞**，见 H-1 / H-2。这不是死锁，是"读线程被拖住"。
3. **[观察] seek 完全在同步体系之外。** 它不加锁、不 `broadcast`、不 `gen++`、不 `ring_reset`。这在 IJK 里"恰好"成立（`av_read_frame` 与 `av_seek_frame` 同在 read_thread），但它是一份**隐式单线程契约**，代码里没有任何断言或注释声明（对比 164 行 `ring_reset` 明确写了 "must be called with s->lock held"）。

### 1.4 "ring_reset 竞态"的真实形态（回答用户问题一.2）

- **[观察] 旧数据不会写进新环。** `gen` 只在持锁下自增（167），预取在写入前比对快照（231 → 238），不一致就整块丢弃；`ring_base / ring_len` 只在持锁下改。这条防线是有效的。
- **[观察] 不会内存越界。** 写入前有容量门限 `ring_cap - ring_len < SMB2_PREFETCH_CHUNK → 等待`（214-217），`ring_len ≤ 12 MB`，`rd ≤ 4 MB`，`memcpy(ring + ring_len, …)` 上界 = 16 MB（247）。成立。
- **[观察] 但"reset 完成"≠"数据可用"，且 reset 无法撤销在途 I/O。** `ring_reset` 只有 11 行账本（165-175），是同步的、微秒级；真正的代价在预取线程：它已经发出的 4 MB `pread` 必须**跑完**才会回到 238 行发现 gen 变了然后丢弃。用户拖一下，这 4 MB 的带宽和这段时间就白烧了，而且新位置的首字节要再等一整块。

---

## 2. 高危代码点

### H-1（P0·并发）前向 seek 没有"首字节"通路：4 MB 块粒度 + 整块作废 → 快速拖动下活锁

**证据链（全部 [观察]）**

1. 读线程在窗口前沿只能等，且**它自己不发网络请求**：621-639（`pos == ring_base + ring_len` → `cond_timedwait` 200 ms，累计 25 s 判超时）。
2. 越过窗口 → 只改账本：642-646 调 `ring_reset`，`continue` 回到 621 继续等。
3. 预取线程取数的最小原子单位是 **4 MB**：219-233 算好 `off / want=4 MB` 后解锁，调用 `smb2_pread_full(s->scratch, want, off)`；该函数内部分成 ≤1 MB 的 SMB 请求循环（140、145-160），但**只在全部完成后才 return**（161）。
4. 返回后才发现 gen 变了 → 整块丢弃（238-239），然后从新的 `ring_base` 重新发一整块 4 MB。
5. 数据只有在整块落盘后才 `broadcast`（247-249）。

**推演**

- 单次前向 seek 的首字节延迟 ≈ `在途块剩余时间 + 4 MB / 有效带宽`。**[推断]** 按 DEMO 自测的 5.35 MB/s（该数字是"满足 1.0× 播出"的吞吐，未测链路峰值）计，≈0.75 s；主项目若在弱网/Wi-Fi 边缘，这个数会更大。
- **高频拖动 → 活锁**：设两次 seek 间隔 T。若 `T < 4 MB / 带宽`，则每一块都在 `gen` 比对上被判废、`ring_len` 始终为 0、读线程一直停在 621 空转；25 s 后 629-631 置 `err = AVERROR(ETIMEDOUT)`，而 `err` 不可清除（见 S-1）→ **整条播放链路永久读失败**。
- 这恰好命中主项目最典型的交互：`SeekGate` 120 ms 合并窗意味着拖动时会以 ~8 次/秒的节奏发 seek，远快于一块 4 MB 的取数周期。

**为什么 DEMO 没暴露**：DEMO 的 seek 验收只有一次 `seekTo(60000)`，且是首帧后 3 s 单发（`v402-prefetch-test-report.md:128-134`）；**没有测"seek 后首字节延迟"，也没有测连续拖动**。这是验收盲区，不是"已验证安全"。

**最小修复方向**：把"块"降为"子块发布"——`smb2_pread_full` 内部每完成 1 MB（或 512 KB）就 append + `broadcast` + 复查 `gen`，只作废尚未发出的子块；`ring_reset` 时置 `urgent` 标志，让下一块首个子块用小块（如 512 KB）先出首字节。改动集中在 136-162 / 219-250，属于局部重构，不动数据结构。

---

### H-2（P0·生命周期）阻塞不可中断：单次 `url_read` 最长 25 s，`close` 只能干等 `join`

**证据链（全部 [观察]）**

1. `libsmb2.c` **全文 0 处** `interrupt_callback` / `ff_check_interrupt` / abort 检查（rg 计数为 0）。`h->interrupt_callback` 从未被读取。
2. FFmpeg 侧的打断点只在两次 `url_read` **之间**：`avio.c:515` `if (ff_check_interrupt(&h->interrupt_callback)) return AVERROR_EXIT;` 位于 `retry_transfer_wrapper` 的循环头。也就是说，**一次 `url_read` 内部无法被打断**。
3. 读线程自身的等待硬上限是 25 s（67、629-631）；单次 SMB 请求超时是 10 s（445-446），4 MB 块 = 最多 4 次请求。
4. `destroy_smb2` 用 `pthread_join` 同步回收（329），`quitting` 只在预取循环顶部（187）和 `pread` **返回之后**（236）检查；`join` 期间 `net_lock` 也没被用来中止请求。
5. IJK 确实注册了打断回调：`ff_ffplay.c:3843-3844` `ic->interrupt_callback = decode_interrupt_cb`（**[观察]**，但它对 `smb` 协议**无效**，因为 1 已证明协议层从不查它）。

**推演**

- `stop()` / `release()` 走 `abort_request` 后，read_thread 卡在 `libsmb2_read` 里，最长 25 s 才返回；`avio_close` → `libsmb2_close` → `join` 再等一个在途请求（≤10 s 的 SMB 超时）。
- 主项目 `PlaybackCommandCenter` 的严格 epoch/生命周期在"新实例已 attach、旧实例未 detach"的窗口里会出现**两个 URLContext 同时持 22 MB**（见 S-6），并让 release 的完成时间从 ~100 ms 级退化到秒级。
- 断网时 DEMO 实测 ~5 s 报 `onError`（报告 3.3），说明"能报错"这条是通的；但那 5 s 是**网络请求自身超时**给的，不是中断给的——一旦服务器"半死"（TCP 通但不应答），阻塞会拉到 10 s/请求的上限。

**最小修复方向**：给 `LIBSMB2Context` 加 `URLContext *h`，在 `smb2_pread_full` 的每次 1 MB 请求前、以及读线程 621-639 的每轮 200 ms 等待里检查 `h->interrupt_callback`，命中即返回 `AVERROR_EXIT` 并置 `quitting`；同时把 `SMB2_WAIT_TIMEOUT_S` 从 25 降到与产品可接受量级一致（如 3-5 s），并把 `rw_timeout` 下传到 `smb2_set_timeout`（当前固定 10 s，445-446）。

---

### H-3（P1·并发/内存）锁纪律两处破口：块缓存"锁外写 + 锁内发布"，seek "锁外改 pos"

**H-3a 块缓存写路径（293-316）**

```
293  pthread_mutex_unlock(&s->lock);          // 选好 victim 后解锁
295  b = &s->blocks[victim];
303  int rd = want ? smb2_pread_full(s, b->data, want, boff) : 0;   // 锁外写 b->data
307  pthread_mutex_lock(&s->lock);
308  b->used = 1; b->off = boff; b->len = rd;                       // 锁内才发布
```

- **[观察]** victim 槽在 307 行之前仍然是 `used=1 + 旧 off/len`，对"另一个调用者"完全可见（275 的命中判定只查 `used/off/within<len`）。
- **[推断]** 一旦出现第二个读者（第二个 URLContext 不算，但同一 context 的并发 `url_read`、或未来把预取也做成多线程、或 `avio_size`/stat 路径从 UI 线程进来），就会：①命中一个正在被覆盖的槽 → 读到半新半旧的静默脏数据；②两个调用者选中同一 victim 互相覆盖 `b->data`。
- **[推断]** IJK 当前单读线程下**不触发**，所以它不是"现网必崩"，而是一颗**随调用方假设变化就响的雷**；而代码里明明有锁、说明作者想要支持并发——这是"锁写了一半"，比完全没有锁更危险。

**H-3b `pos`（556 vs 591/606/668）**

- **[观察]** `libsmb2_seek` 在 556 行裸写 `libsmb2->pos`，不持锁；读循环在持锁下做 `pos = libsmb2->pos`（591）→ `libsmb2->pos = pos + n`（606）/ `= spos + rd`（668）的**读改写**。
- **[推断]** 若 seek 与 read 不同线程（例如后续把 `avio_size`、或某些 stat 逻辑放到 UI/统计线程），读循环会用 `pos + n` 覆盖掉刚写进去的新位置 → **seek 被静默吞掉**；上层（`FFP_MSG_SEEK_COMPLETE` 早在 `ff_ffplay.c:4221` 就发了）会认为定位已受理，实际数据仍在旧位置 → epoch 判旧逻辑拿到错误关联。
- **[推断]** aarch64 上 8 字节对齐访问不撕裂，所以不会读到半个地址；风险是"丢更新"，不是"读到垃圾地址"。

**最小修复方向**：块缓存改为"先读到临时缓冲，锁内发布"（或加 per-slot `filling` 标志，命中时跳过）；`pos` 改 `_Atomic uint64_t` 或引入独立 `pos_lock`（不能在 `lock` 上加，因为 `lock` 会被长临界区持有）。

---

## 3. 次级发现（不单独立项，但合入前应一并处理）

| ID | 位置 | 现象 | 后果 | 级别 |
|---|---|---|---|---|
| S-1 | 594-597、165-175、189-192 | `err` 一旦置位，`ring_reset` / `seek` 都不清除，预取线程从此停在 189 永久 park | 一次瞬时错误 = 该 URLContext 永久失败；seek 无法自救 | P1 |
| S-2 | 393-397 | `smb2_init_context()` 返回 NULL 时仍调 `smb2_get_error(libsmb2->ctx)` | 空指针解引用，进程崩溃（`onError` 无法捕获） | P2 |
| S-3 | 300-317 + 663-666 / 243-245 | 块读若返回短读（`rd < within`）→ `n=0` → 调用方当 EOF；预取 `rd==0` 也直接置 `eof`，不校验是否真到 `filesize` | 弱网/服务端短读 → **片中误报 EOF**，播放提前结束；`eof` 同样粘滞到下一次 `ring_reset` | P2 |
| S-4 | 221-229、298-301 | `stat` 失败 → `filesize = -1`，预取 `want` 恒为 4 MB，靠 `rd==0` 收尾 | 无长度上界；服务端异常返回 0 即提前 EOF | P2 |
| S-5 | 543、547-548 | `filesize == -1` 时 `SEEK_END` 得 `-1`，被 547-548 夹到 0 | "seek 到末尾"静默变成"seek 到开头" | P3 |
| S-6 | 487-511 | ring 16 MB + scratch 4 MB + 8×256 KB = **22 MB/实例**，只在 `destroy` 释放，无缩容 | DEMO 实测 PSS 108→121 MB（+13 MB，报告 3.1）；主项目若并发/重连期存在两个实例 → +44 MB | P1 |
| S-7 | seek.c 192-194 + aviobuf.c:43/83 | smb 跳过 `ff_configure_buffers_for_index`，`buffer_size` 与 `short_seek_threshold` 固定 32 KB | 所有 >32 KB 的跨轨回跳全部落到 8×256 KB = **2 MB 块缓存**；若主项目片源的两轨索引间距 >2 MB，LRU 抖动，退化为每 256 KB 一次网络读 | P2（片源相关） |
| S-8 | 206 | 窗口滑动在持 `lock` 下 `memmove` 最多 12 MB | 读线程每次滑动被阻塞数 ms，属可接受抖动，但会叠加到 seek 延迟 | P3 |
| S-9 | 754、756 | `libsmb2_open_dir` 的返回码反了（失败返回 0、成功返回 `AVERROR(ENOTDIR)`）；`libsmb2_delete/move/open_dir` 泄漏 `smb2_url` 与 `ff_urldecode` 结果；open 内 413/424/430 三个解码结果也泄漏 | 若主项目用它扫 SMB 目录列片单，会"成功返回失败"；长期运行有微量泄漏 | P2（非读路径） |

补充 **[观察]**：`ring_reset` 之后 `consume_off / ring_base / ring_len` 全部归零，但**块缓存不失效**——这是对的（块缓存按 `off` 寻址，内容与窗口无关），不属于"旧数据污染"。

---

## 4. 三维结论（直接回答用户问题）

### 4.1 并发 / 线程安全

- 死锁：**安全**（无锁嵌套，第 1.3 节）。
- ring 数据正确性：**安全**（`gen` 校验 + 容量门限，第 1.4 节）。
- 时延与吞吐：**不安全（需改架构）**。前向 seek 无首字节通路（H-1），高频拖动下会退化成活锁 + 25 s 超时。
- 块缓存 / `pos`：**需加锁**（H-3）。

### 4.2 内存 / 生命周期

- `ring_reset` 是否释放旧 16 MB：**不释放，也不需要释放**——它是同一块内存的复用（168-170 只改索引），**不是泄漏**。不存在"每次 reset 漏 16 MB"。
- `onError` / `close` 时预取线程：**会被回收**（`destroy_smb2` 324-331 置 `quitting` + `join`；ring/scratch/8 个块在 345-348 全部 `av_freep`）。**不构成悬挂线程**。
  - 但回收**不是即时的**：`join` 必须等在途 `pread` 结束（H-2），最坏秒级。
- seek 期间的临时缓冲：**不累积**。块缓存 2 MB 在 open 时一次性分配（496-503），seek 不分配任何内存，无 PSS 单调增长路径。
- 真正的内存问题：**22 MB/实例固定占用 + 无法缩容**（S-6），以及并发/重连期的实例叠加。

### 4.3 异常

- 错误码映射：**安全**（121-134：`-1`→`ENOTCONN`、`STATUS_IO_TIMEOUT`→`ETIMEDOUT`、越界 errno→`EIO`；不再吞错返回 0/部分长度）。
- 崩溃风险：**有一处**（S-2 空指针），概率低但性质是"崩"而不是"报错"。
- 错误可恢复性：**不安全**。`err` 粘滞（S-1），一次错误即永久失败，且预取线程不重启、读线程无直读兜底（对比 568-587 的 `!ring` 兜底路径，ring 模式下**没有等价 fallback**）。
- 主项目 `onError` 能否接住：**能接住"错误码"，接不住"阻塞"和"崩溃"**。见 6.3。

---

## 5. 极端场景推演（静态推演，未跑真机）

| # | 场景 | 触发条件 | 预期表现 | 依据 |
|---|---|---|---|---|
| S1 | 快速连续拖动 | 两次 seek 间隔 < 4 MB 取数周期 | `gen` 反复作废整块，`ring_len` 恒 0，读线程空转；25 s 后 `ETIMEDOUT`，之后永久失败 | H-1、629-631、238-239 |
| S2 | 单次远距离前跳 | 目标在窗口之外 | 至少等一整块 4 MB 才出首字节 | H-1、621-646 |
| S3 | 弱网 / 高 RTT | 单请求 >1 s | seek 延迟线性放大；4 请求/块 → 单块可达 4 s | H-1、145-160 |
| S4 | 断网后 release | `abort_request` → `stop` | read_thread 卡到 25 s 上限；`avio_close` 再等 `join`（≤10 s） | H-2 |
| S5 | 服务端短读 | 半开连接、SMB 服务重启 | 片中误报 EOF（`eof=1`），播放提前结束且不自愈 | S-3 |
| S6 | 长播 2 h 后 seek | 稳态 | 无泄漏迹象（无 per-seek 分配）；风险只在 S-1 的"一次错误终身失败" | 3.1、第 4.2 节 |
| S7 | 快速切片（多实例） | 两个实例并存窗口 | +44 MB PSS 峰值；两实例各自 `net_lock`，互不阻塞（不同 ctx） | S-6 |
| S8 | 音轨远离视频 >2 MB 的片源 | 主项目片源与 DEMO 不同 | 块缓存 2 MB 被打穿，跨轨回跳退化为每 256 KB 一次网络读 | S-7 |

---

## 6. 与主项目状态机的兼容性推演

> 主项目细节为 **[前提]**（用户给定：`SeekGate` 120 ms 合并窗、`PlaybackCommandCenter` 严格 epoch），本轮未读其源码。

### 6.1 ring_reset 未完成时又来 seek

- **[观察]** 不存在"ring_reset 未完成"这个状态：它是 11 行同步账本（165-175），调用点全在读线程内（617/643/671）。
- **[观察]** 新的 `url_seek` 只做三件事：回调 `will/did_http_seek`（535/557）、算新位置、写 `pos`（556）。**不等任何 I/O**。
- **[推断]** 所以"seek 会不会被 ring_reset 挡住"的答案是：**不会，反过来才是问题**——seek 立即返回成功，但数据要等预取把在途 4 MB 跑完 + 新块 4 MB 跑完。`SeekGate` 的 120 ms 合并窗在这一层完全失去意义：它只能压掉重复的 seek 请求，压不掉"每次 seek 都要 4 MB"的物理成本。
- **[推断]** 由于 seek 与 read 同在 read_thread，seek 只能在前一次 `libsmb2_read` **返回之后**才被执行；前一次 read 最长 25 s（H-2）→ 用户连续拖动时，seek 请求在 IJK 的 `seek_req` 上堆积，最终只执行最后一个，中间目标被丢弃（这本身是对的，但与上层 epoch 的期望需要对齐：上层会收到多次 `SEEK_COMPLETE`，每次都不带位置，`ff_ffplay.c:4221`）。

### 6.2 会不会阻塞主项目命令队列

- **[推断]** 不会阻塞**入队**：IJK 的 `stream_seek` 只置标志位，`PlaybackCommandCenter` 的命令投递不会被 native 阻塞。
- **[推断]** 会阻塞**生效与回收**：read_thread 被 `libsmb2_read` 占住时，seek 不执行、`abort_request` 不生效、`release` 的 native 侧不能在数百 ms 内完成。若主项目的 epoch 逻辑以"native 侧 ack 到达时间"做超时判定，会持续判超时并重发 → 放大成 S1 的活锁。

### 6.3 onError 能否捕获这些异常

| 异常类型 | 能否捕获 | 原因 |
|---|---|---|
| SMB 错误码 / 超时 | **能** | 121-134 映射成 AVERROR → `av_read_frame` 返回负 → IJK 报 `onError(-10000)`（DEMO 实测 ~5 s） |
| 提前 EOF（S-3） | **不能** | 表现为"正常播完"，上层收到的是 EOF 而不是 error |
| 25 s 阻塞（H-2） | **不能** | 没有错误发生，只是不动；上层只能靠自己的超时兜 |
| 空指针崩溃（S-2） | **不能** | 进程级 SIGSEGV，JS 层 `onError` 无从触发 |

---

## 7. 合入判定与必修清单

### 7.1 判定

**不具备直接运行的安全性。** 这份代码在 DEMO 的验收口径下（单次 seek、千兆内网、单读线程、断网即放弃）是成立的；但主项目的三个前提——**高频 seek（120 ms 窗）、严格生命周期（release 需要秒内完成）、多片源（音轨距离不定）**——正好各自命中 H-1 / H-2 / S-7。原封不动合入，最先出问题的不是"崩溃"，而是 **seek 变慢 → 上层判超时 → 重发 seek → 活锁 → 永久报错**。

### 7.2 必须先修的四处（按顺序）

| 序号 | 问题 | 位置 | 修法（最小改动） | 不修的后果 |
|---|---|---|---|---|
| **F-1（P0）** | 4 MB 块不可分割、seek 整块作废 | 136-162、219-250、165-175 | 子块发布：每 1 MB（或 512 KB）append + `broadcast` + 复查 `gen`；`ring_reset` 置 `urgent`，下一块首子块降到 512 KB | S1 活锁、S2 单次 seek ≥ 数秒 |
| **F-2（P0）** | 不可中断，read 25 s / close 等 join | 67、629-631、136-162、324-331、445-446 | 存 `URLContext *h`，在每次 1 MB 请求前 + 每轮 200 ms 等待里查 `h->interrupt_callback`，命中返回 `AVERROR_EXIT` 并置 `quitting`；`SMB2_WAIT_TIMEOUT_S` 降到 3-5 s；`rw_timeout` 下传 | S4 release 秒级卡顿，主项目生命周期断言失败 |
| **F-3（P1）** | 块缓存锁外写、`pos` 锁外改 | 293-316、556 | 块缓存"临时缓冲 + 锁内发布"（或 per-slot `filling`）；`pos` 改原子或独立 `pos_lock`；seek 里显式 `gen++`/`broadcast`（让预取尽快换向） | 一旦出现第二读者：脏数据 / 丢 seek，且**难复现** |
| **F-4（P1）** | `err` 粘滞、无自愈 | 165-175、594-597、189-192 | `ring_reset` 清 `err` / `err_logged`；读线程在 `err` 且 `prefetch_started` 时允许"重启预取一次"或直读兜底 | 一次瞬时错误导致整场播放不可恢复；与主项目的 fallback 语义冲突（上层以为要换源，其实只需重试） |

> 次级项（S-2 空指针、S-3 误 EOF、S-9 目录返回码反转）建议同批修，但**不是合入阻塞项**——它们要么触发概率低，要么不在读路径上。

### 7.3 修复后的可测验收判据（替代 DEMO 现有盲区）

1. `seekTo` 后到首字节的耗时分布（p50/p95），目标：局域网 < 300 ms，与 `SeekGate` 120 ms 同量级。
2. 连续拖动 20 次（间隔 100/150/200 ms），要求：不出现 `AVERROR(ETIMEDOUT)`、`pos` 最终落到最后一个目标、无 `err` 粘滞。
3. `release` 时延：断网状态与正常状态各测 10 次，目标 p95 < 1 s。
4. 播放中 `svc wifi disable` → 恢复，验证能否不换实例直接续播（当前 F-4 未修时必然不能）。
5. 用音轨距离 >2 MB 的片源复测 S-7（当前 DEMO 素材音轨就在文件头，掩盖了块缓存容量假设）。

---

## 8. 证据索引

| 断言 | 文件:行 |
|---|---|
| 预取线程唯一写 ring | `libsmb2.c:247` |
| 容量门限（越界防线） | `libsmb2.c:214-217` |
| `gen` 快照 / 校验 | `libsmb2.c:231`、`238-239` |
| 25 s 读等待硬上限 | `libsmb2.c:67`、`629-631` |
| 4 MB 块原子取数 | `libsmb2.c:219-233`、`136-162` |
| 块缓存锁外写 | `libsmb2.c:293`、`303`、`307-316` |
| seek 裸写 pos | `libsmb2.c:556` |
| `err` 粘滞 | `libsmb2.c:594-597`、`174`（ring_reset 不清 err） |
| join 回收 | `libsmb2.c:324-331` |
| 22 MB/实例 | `libsmb2.c:487-503`（16+4+2 MB） |
| 空指针（open 失败） | `libsmb2.c:393-397` |
| 协议层无中断检查 | `libsmb2.c` 全文 0 处 `interrupt` |
| AVIO 中断只在两次 read 之间 | `avio.c:504-547`（515） |
| IJK 注册了中断回调 | `ff_ffplay.c:3843-3844` |
| 跳过缓冲放大（smb） | `seek.c:192-194` |
| 默认 32 KB / 短 seek 阈值 | `aviobuf.c:43`、`83`、`265-289` |
| DEMO seek 验收只有单次 | `v402-prefetch-test-report.md:128-134` |
| PSS +13 MB | `v402-prefetch-test-report.md:123` |
