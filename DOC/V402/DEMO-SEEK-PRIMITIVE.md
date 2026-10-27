# DEMO-SEEK-PRIMITIVE — libsmb2 直读 seek 原语测前定性

- 仓库：`D:\devlop\ZplayerDemo`（DEMO 线，未改主项目、未合并）
- 读取边界：只读「Custom AVIO / seek 回调 / 渲染入口」三类文件；grep 定位后取命中前后 40 行
- 未读：主项目代码、evidence 历史文档、DEMO 全仓、README / v402-prefetch-test-report.md
- 标注约定：**[观察]** = 本轮直接查到的源码行；**[推断]** = 由观察或外部前提推出的结论；**[前提]** = 用户提供的 T-14 事实，本轮未验证

---

## 1. DEMO 的 AVIO 接口：三个回调在哪，完成边界怎么定义

### 1.1 不是 `avio_alloc_context`，是 URLProtocol

**[观察]** DEMO 播放用的直读路径没有手写 `AVIOContext`，而是给 FFmpeg 注册了一个 URLProtocol（协议名 `smb`），三个回调在此挂载：

`native/src/ohos_ffmpeg/libavformat/libsmb2.c:839-854`
```
839 const URLProtocol ff_libsmb2_protocol = {
840     .name = "smb",
844     .url_open        = libsmb2_open,
845     .url_read        = libsmb2_read,
846     .url_write       = libsmb2_write,
847     .url_seek        = libsmb2_seek,
848     .url_close       = libsmb2_close,
```

- `read` = `libsmb2_read`（561-690）
- `seek` = `libsmb2_seek`（521-559）
- `write` = `libsmb2_write`（692-704），播放只读，实际不走

### 1.2 seek 完成边界 = lseek 返回即"完成"（纯内存改 pos，零网络）

**[观察]** `libsmb2_seek` 整个函数没有一次 `smb2_pread` / 网络调用：

`libsmb2.c:521-559`
```
526     if (whence == AVSEEK_SIZE) {            // 只返回缓存的 filesize
538     if (base == SEEK_SET)  newpos = pos;
541     if (base == SEEK_CUR)  newpos = libsmb2->pos + pos;
543     if (base == SEEK_END)  newpos = filesize + pos;
549     if (... (uint64_t)newpos > filesize) newpos = filesize;   // 末端 clamp
556     libsmb2->pos = (uint64_t)newpos;        // 唯一副作用
557     av_application_did_http_seek(..., 0, 200);
558     return newpos;                          // 立刻返回
```

- 没有校验目标偏移处是否可读，也没有等待 SMB 应答。
- 真正的数据移动发生在**下一次 read**：`libsmb2_read`
  - ring 命中：直接 `memcpy`（599-614）
  - 目标在 ring 尾部之后（前跳）：`ring_reset` 丢弃窗口后重开（642-646）
  - 目标落在窗口之后很远：走 256 KB 块缓存 `smb2_block_read`（648-672，块大小见 64-65 行）
  - 回退累计超过 1 MB 才重置预取窗口（670-671，阈值见 66 行）
  - ring 未启用时退化为直读 `smb2_pread_full`（568-587）

**[观察]** 预取线程独立推进 4 MB 分块（61-67、177-254），seek 只改 `gen`（167）让在途分块作废。

**[推断]** 所以 AVIO 层的 seek 是一个**乐观 seek**：`lseek 返回` ≠ 「目标位置的数据已经拿到」，只是「偏移量已记账」。任何"seek 完成"判断都不能停在这一层。

### 1.3 另一条自定义 IO（HTTP bridge，只在 `PocCfg.mode` 走 http 时用）

**[观察]** `entry/src/main/cpp/http_bridge.cpp` 起本地 HTTP 服务，自己解 `Range:`（325-345）、回 206 + `Content-Range`（395-399）、按块回写（420-431）；底层 `entry/src/main/cpp/smb_io.cpp:107-134` 用 `smb2_pread` 直读，`Seek` 同样是纯算术（136-159）。

**[观察]** 默认页走 HTTP：`entry/src/main/ets/pages/Index.ets:66-69` 起服务后 `http://127.0.0.1:${port}/video.mp4`；拖拽 fixture 走直读：`entry/src/main/ets/fixture/DragHarnessPage.ets:59` 用 `smb://...`。

**[推断]** 本专项要测的"libsmb2 直读 seek 原语"应绑定 `smb://` 路径（即 1.1/1.2 的 URLProtocol），HTTP bridge 的 Range 语义是另一套，不与主项目 IJK 路径同构。

---

## 2. DEMO 的"seek 完成"信号来源（路径二能否提供"可造 ack"的核心）

DEMO 里同时存在三个层级的信号，语义完全不同：

### L1 AVIO：**没有**完成信号

见 1.2。`url_seek` 返回即返回，无事件、无回调。

### L2 解封装层：`FFP_MSG_SEEK_COMPLETE` =「定位被受理」，不是「画面到位」

**[观察]** `ff_ffplay.c:4150` 调 `avformat_seek_file` → flush 队列 + 塞 flush_pkt（4168-4174）→ 提升 serial（4181-4183）→ 立刻发消息：

`ff_ffplay.c:4221`
```
4221 ffp_notify_msg3(ffp, FFP_MSG_SEEK_COMPLETE, (int)fftime_to_milliseconds(seek_target), ret);
```

**[观察]** 消息映射链：
- `ff_ffmsg.h:52` `FFP_MSG_SEEK_COMPLETE = 600`
- `proxy/ijkplayer_napi_proxy.cpp:120-123` → `MEDIA_SEEK_COMPLETE`
- `ijkplayer_android_def.h:37` `MEDIA_SEEK_COMPLETE = 4`
- ArkTS 侧落到 `OnSeekCompleteListener`（`DragHarnessPage.ets:141-159`、`Index.ets:202-213`），**回调不带位置参数**，只能回调里再 `getCurrentPosition()`

**[推断]** 这条消息在**任何一帧被读出之前**就发了（4221 之后才回到读循环），所以它不能当作"seek 完成 ack"。它只说明 demuxer 接受了定位。

### L3 渲染层：`FFP_MSG_VIDEO_SEEK_RENDERING_START` = 真·首帧上屏 ack

**[观察]** 每帧显示后，检查该帧 serial 是否等于"最近一次 seek 的 serial"，相等才发：

`ff_ffplay.c:1240-1258`
```
1240 SDL_VoutDisplayYUVOverlay(ffp->vout, vp->bmp);      // 先上屏
1242 if (!ffp->first_video_frame_rendered) { ... FFP_MSG_VIDEO_RENDERING_START }
1247 if (is->latest_video_seek_load_serial == vp->serial) {
1249     __atomic_exchange_n(&is->latest_video_seek_load_serial, -1, ...);
1250     if (latest_video_seek_load_serial == vp->serial) {
1251         ffp->stat.latest_seek_load_duration = (av_gettime() - is->latest_seek_load_start_at) / 1000;
1253         ffp_notify_msg2(ffp, FFP_MSG_VIDEO_SEEK_RENDERING_START, 1);
```
serial 在 seek 成功时打点：`ff_ffplay.c:4181`，起点时间戳 `latest_seek_load_start_at` 在 4183。

**[观察]** 映射链：
- `ff_ffmsg.h:42` `FFP_MSG_VIDEO_SEEK_RENDERING_START = 410`
- `proxy/ijkplayer_napi_proxy.cpp:149-152` → `MEDIA_INFO` / `MEDIA_INFO_VIDEO_SEEK_RENDERING_START`
- `ijkplayer_android_def.h:143` `= 10008`
- ArkTS `onInfo(what, extra)`；`extra`（arg1）是 1/0 = 是否 video-master（1252-1255），**不是位置**

**[推断]** 这才是"可造 ack"的原料：它严格发生在 `SDL_VoutDisplayYUVOverlay` **之后**、且要求 serial 匹配，等价于"本次 seek 之后的第一帧已经上屏"。**但它不自带落点**，落点仍要另取（`getCurrentPosition()` 或帧 pts），因此 acK 与落点必须成对采集。

### L4 精确 seek：`FFP_MSG_ACCURATE_SEEK_COMPLETE` —— DEMO 运行时不发

**[观察]**
- 常量：`ff_ffmsg.h:55` `FFP_MSG_ACCURATE_SEEK_COMPLETE = 900` → `10100`（`ijkplayer_android_def.h:146`），映射见 `proxy:125-128`
- 发射点全部在 `enable_accurate_seek` 分支内：`ff_ffplay.c:1856`（video 侧）、`4206-4219`（seek 后置请求）
- 默认关闭：`ff_ffplay_def.h:814` `ffp->enable_accurate_seek = 0;`，选项定义 `ff_ffplay_options.h:156-157`
- DEMO 从未设置该选项：`Index.ets:143-151`、`DragHarnessPage.ets:79-87` 只设了 mediacodec / framedrop / packet-buffering / reconnect / start-on-prepared

**[推断]** DEMO 实测中 `MEDIA_INFO_MEDIA_ACCURATE_SEEK_COMPLETE` 不会到达 ArkTS。若要把"精确落点 ack"作为证据，必须在 DEMO 侧显式打开 `enable-accurate-seek=1`，否则这条链是死代码。

### DEMO 独有的播放层仲裁（不属于 native ack）

**[观察]** `entry/src/main/ets/player/SeekArbiter.ets`
- 150 ms 合并窗（45-49），leading 立刻发、窗内合并、release 立即 flush（56-105）
- 用 `gen/pendingGen` 判定一次完成回调是否属于"最新目标"（108-113、146-160）

**[推断]** 这是 DEMO 侧策略层，不在 native，也不在 IJK 上游；它不能当作"主项目可复用 ack 机制"，只能当作"ack 消费方式"的参考实现。

---

## 3. DEMO 的渲染路径：是否 SURFACE 零拷贝

### 3.1 入口确认走 SURFACE

**[观察]**
- `Index.ets:135-142`：`native_setup(VideoCodecMode.SURFACE)`，默认值 `PocCfg.ets:10` `codec = 'surface'`
- `DragHarnessPage.ets:78`：`this.player.native_setup(VideoCodecMode.SURFACE)`（fixture 固定 SURFACE）

### 3.2 解码器直接写 OHNativeWindow（零拷贝）

**[观察]**
- XComponent 的 native window 传入解码节点：`ffpipenode_ohos_mediacodec_vdec.cpp:1416-1427`（`nativeWindow_ = nativeWindow`，并注册释放回调）
- 配置解码器时把 surface 交给硬件解码器：`ohos_video_decoder.cpp:54-61`
  ```
  54  if (nativeWindow_ != nullptr) {
  56      OH_AVErrCode ret = OH_VideoDecoder_SetSurface(decoder_, (OHNativeWindow*)nativeWindow_);
  ```
- 输出分支：`ffpipenode_ohos_mediacodec_vdec.cpp:954-960`（有 nativeWindow 走 Surface）→ `962-1006 DecoderOutputSurface`：只把 `bufferIndex` 包成 proxy 挂到 `frame->opaque`（1000），**不做 YUV memcpy**（对比 buffer 模式的 `DecoderOutputBuffer`，921-951 才有拷贝）
- 渲染时按 `render=1` 把 buffer 还给解码器/送显：`ijksdl_video/ijksdl_vout_android_nativewindow.c:314-333` → `ffpipenode_ohos_mediacodec_vdec.cpp:38-48 IjkohosVoutReleaseOutput`（`dec->RenderOutputBuffer`）

**[观察]** `SDL_VoutOhos_obtainBufferProxy` 只分配 24 字节左右的 proxy（282-297），不碰像素。

**[推断]** DEMO 的 SURFACE 路径是零拷贝：解码器 → OHNativeWindow 直接合成，应用层只持有 buffer index。

### 3.3 与主项目是否同一套

**[观察]** 本轮只读到 DEMO 侧实现，**未读主项目**（边界禁止）。

**[前提]** T-14 结论：FFmpeg / IJK 功能源码 / libsmb2 二进制同源。

**[推断]** 若主项目 IJK 用的是同一份 `ohos` pipeline（`ffpipenode_ohos_mediacodec_vdec.cpp` + `ohos_video_decoder.cpp` + `ijksdl`），则 SURFACE 零拷贝是同一套；此点本轮**无法直接证实**，需主项目侧一次 grep 对照（`OH_VideoDecoder_SetSurface` / `DecoderOutputSurface`）。

---

## 4. DEMO 的 IJK 是否发出三个 seek 相关消息

| 消息 | 常量 / 行号 | 到 ArkTS 的映射 | DEMO 运行时是否发 |
|---|---|---|---|
| `FFP_MSG_SEEK_COMPLETE` | `ff_ffmsg.h:52` = 600 | `MEDIA_SEEK_COMPLETE` = 4（`ijkplayer_android_def.h:37`，proxy 120-123） | **发**，每次 seek 一次（`ff_ffplay.c:4221`） |
| `MEDIA_INFO_VIDEO_SEEK_RENDERING_START` | 源 `FFP_MSG_VIDEO_SEEK_RENDERING_START` `ff_ffmsg.h:42` = 410 | `10008`（`ijkplayer_android_def.h:143`，proxy 149-152） | **发**，每次 seek 后首帧上屏一次（`ff_ffplay.c:1247-1257`） |
| `MEDIA_INFO_MEDIA_ACCURATE_SEEK_COMPLETE` | 源 `FFP_MSG_ACCURATE_SEEK_COMPLETE` `ff_ffmsg.h:55` = 900 | `10100`（`ijkplayer_android_def.h:146`，proxy 125-128） | **不发**：`enable_accurate_seek` 默认 0（`ff_ffplay_def.h:814`），DEMO 未开启 |

补充（观察，影响第二步取数质量）：
- `DragHarnessPage.ets:21` 把 `INFO_RENDERING_START` 定义为 `3`（即 `MEDIA_INFO_VIDEO_RENDERING_START`，`ijkplayer_android_def.h:109`），而该消息只在整个播放生命周期发一次（`ff_ffplay.c:1242-1245`）。因此 harness 的 `frame` 打点实际主要来自 16 ms 位置轮询（`DragHarnessPage.ets:256-281`），**不是** 10008 渲染 ack。
- 10008 在 harness 里只会落到 `what > 10000` 的日志分支（`DragHarnessPage.ets:136-138`），带 hilog 时间戳但不与样本号绑定。

**[推断]** 第二步若要用 10008 作为"首帧渲染"证据，需要在 harness 里把 10008 与样本 seq 显式关联（或用 hilog 时间戳 + serial 顺序对齐），否则只能用位置轮询近似。

---

## 5. 结论（一句话 + 可外推清单）

**一句话**：DEMO 的 libsmb2 直读 seek 在 AVIO 层是"返回即完成"的乐观 seek（`libsmb2.c:521-559`，无网络、无校验）；真正可用的完成信号只有渲染层的 `MEDIA_INFO_VIDEO_SEEK_RENDERING_START(10008)`（`ff_ffplay.c:1240-1258`），它证明"新 serial 首帧已上屏"但不携带落点；`FFP_MSG_SEEK_COMPLETE(600)` 只代表定位被受理，`10100` 在 DEMO 未开启。

**可外推到主项目**（依据 T-14「FFmpeg/IJK 功能源码同源」前提）
1. libsmb2 URLProtocol 的 seek 语义：`url_seek` 纯改偏移、数据延迟到 read —— 只要主项目用同一份 `libsmb2.c`，语义即相同。
2. IJK 的 10008 渲染 ack 链（`ff_ffplay.c:1247-1257` → proxy → `10008`）—— 前提是主项目 IJK 同源**且**其 NAPI 层把 10008 透传到 ArkTS（DEMO 已透传，主项目未验证）。
3. `FFP_MSG_SEEK_COMPLETE(600)` 的"早发"性质 —— 同源即相同，可用于反面论证"它不是 ack"。

**仅 DEMO 有效**
1. `SeekArbiter.ets` 的 150 ms 合并窗 / gen 判旧 —— DEMO 侧策略，非 native。
2. HTTP bridge（`http_bridge.cpp` + `smb_io.cpp`）的 Range 语义 —— 与 smb:// 直读不同构。
3. `enable-accurate-seek` 的开关状态 —— DEMO 未开，不能代表"IJK 不支持"，也不能代表主项目状态。
4. 预定常量（16 MB ring / 4 MB chunk / 256 KB×8 块 / 1 MB 回退阈值，`libsmb2.c:61-66`）—— 属 0020 补丁调参，可移植但不是同源保证。

**对路径二的直接含义**：DEMO 能提供的"可造 ack"证据是 10008（渲染后、serial 绑定），这比主项目现有的"无 ack + 时间窗仲裁"更精确；但 ack 不含落点，且 libsmb2 的 lseek 本身不提供任何完成语义 —— 即"更精确"来自**渲染层已有的 IJK 消息**，不是来自 libsmb2 直读。移植判据见第三步，需先完成第二步数据。
