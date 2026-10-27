# SURFACE 零拷贝与 IJK 耦合边界审计（V402）

- 审计对象：`D:\devlop\Zplayer`，HEAD `edeadea`（2026-10-27）
- 审计方式：只读源码 + 只读二进制符号/字符串 + 历史文档对照；不构建、不跑真机、不改代码
- 证据等级：
  - **[观测]** 本次直接读到的源码行、二进制符号/字符串、文件哈希
  - **[文档]** 仓库历史记录中的既有结论（注明出处）
  - **[推断]** 由代码事实推导、但本次未做真机复现的结论

> 先纠正一个命名前提：全盘（`D:\devlop`）**未定位到名为“SURFACE 零拷贝.so”的单文件产物**。当前主线实际承担零拷贝的是官方 `@ohos/ijkplayer@2.0.11` 包内的 `libavcodec.so` + `libijkplayer.so` + `libijksdl.so` + `libavutil.so` 组合，入口是 ArkTS 的 `VideoCodecMode.SURFACE`。详见 §2。

---

## 0. 一句话结论

| 问题 | 结论 | 决定性证据 |
|---|---|---|
| 零拷贝发生在哪一层 | 在 OHCodec 解码器内部直写 `OHNativeWindow`，IJK 只持有 buffer index 并调用 `RenderOutputBuffer`；应用/FFmpeg 不拷贝像素 | `ohdec.c:145-201`、`ohdec.c:530-589`、`ohdec.c:492-527`、`ijksdl_vout_android_nativewindow.c:169-172` |
| 耦合方式 a/b/c | **a（vendor 层）+ c（Zplayer 集成层），不是 b** | Zplayer 无任何 IJK/FFmpeg native 源码；只调用 `native_setup(VideoCodecMode.SURFACE)`；SURFACE 分支在第三方 .so 内 |
| 能否用“删一个 .so”拆掉 SURFACE | **不能**。没有独立 SURFACE .so；SURFACE 横跨 4 个预编译库 | §2 的哈希与符号 |
| 当前最大生命周期风险 | XComponent 销毁回调里 `delete` NAPI 实例/代理，但**不释放 player、不置空 surface**；裸 `OHNativeWindow*` 无引用计数 | `ijkplayer_napi.cpp:1181-1221`、`ijkplayer_napi_proxy.cpp:303-320`、`ijksdl_vout_android_nativewindow.c:124-128` |

---

## 1. 历史记录提取（先查文档）

### 1.1 v393：`AvPlayerBackend.applySurface` 的 `detach-before-attach`

**历史事实链 [文档]**：

1. `DOC/DEVLOG.md:79`（v393-FlashCatch）：录屏抓到 1 帧（f133）横屏画面纵向拉满 2560，持续约 28 ms；几何比对指向 **Surface 绑定切换时的旧 buffer 瞬变**，不是占位图。
2. `DOC/DEVLOG.md:80`（v393-FlickerFix）：据此在 `AvPlayerBackend.applySurface` 中实现 `detach-before-attach`——Surface id 变化时先 `av.surfaceId=''` 解绑旧 buffer，再 attach 新 id；同时给占位图加 `animateTo({delay:32,duration:100})`。
3. `DOC/DEVLOG.md:81`（v393-FlickerFix-Revert）：**首版修复有害**。新录屏 57.3 s/2208 帧出现 **596 帧 h≥2000、15 段每段 0.5–1.2 s**；hilog 报 `5400102 Operate Not Permit ... unsupport video scale operation`，`AvRetryCtl` 反复重试。根因：`applyScaleType` 提前到 `initialized`，以及 `detach-before-attach` 让 Surface 空置/旧 buffer 停留。**两项均回滚**。
4. `DOC/DEVLOG.md:82-84`（v393-FlickerFix-2 / Verified / Closure）：最终策略是 `Swiper.clip(true)` + `tryShowPlaceholder` 幂等包含 `fading` + `fadeOut` 延迟；v393 验收通过。**`detach-before-attach` 已不在主线**。
5. `DOC/DEVLOG.md:85`（v394-TrueFix）：改为 `nativeSurfaceId` 幂等写入口，真实换面才重置首帧闸门，删除 32 ms 盲等；当前代码与之相符。

**当前代码 [观测]**：

- `entry/src/main/ets/domains/player/AvPlayerBackend.ets:150-179`：`applySurface` 只做 `targetChanged` 判定、重置 `firstFrameFlag/readySent`、必要时 `attachSurface(this.av, this.state, id)`；**没有先置 `''` 的 detach 分支**。
- `entry/src/main/ets/domains/player/AvVideoUtils.ets:31-43`：`attachSurface` 只执行 `av.surfaceId = id`。

**回答“为什么要这么做 / 不做会崩在哪”**：

- 当初的目的不是防崩溃，而是防 **1 帧旧 buffer 被拉伸**（v393-FlashCatch）。
- 实际结果相反：做了 `detach-before-attach` 后出现 **5400102 + 596 帧异常**（v393-FlickerFix-Revert），不是“不做就崩”。
- 当前主线不做 detach，也没有已知崩溃；残留风险是换面瞬间的旧 buffer 瞬变，由 `Swiper.clip` + 占位层 + v394 幂等闸门覆盖。

### 1.2 v383 / v399：`SurfaceOpsCtl` 与 `PlayerBindings` 对 XComponent 生命周期的接管

**v383（P1/P2/P3）[文档 + 观测]**：

- `DOC/DEVLOG.md:60`（v383-P1）：
  - `exitFull` 改为 `bindSlotSurface` 回稳定层；
  - `PlayerBindings.bindStable` 只在活动槽且 `intent.gen()>0` 时注入真 gen；
  - `releaseForSid` 对活动槽增加 `getAppliedSurfaceId()===sid` 守卫。
- `DOC/DEVLOG.md:61`（v383-P2）：
  - `PlayerBindings.registry` 改 `private`；
  - `updateActiveSurface`/`bindPage` 守卫升级为 `gen<=0` 一律拒绝，**gen=0 全链消灭**。
- `DOC/DEVLOG.md:62`（v383-P3）：`surfaceId/ijkContext/ijkXid` 合并为 `surface: SurfaceHandle`，删除 `SurfaceBinding.ets`。

对应现状：

- `PlayerBindings.ets:62-87`：`bindPage` 先校验 gen、解析 `intent.resolveSlot(uri,gen)`，再 `registry.setPage`，活动槽才 `inject`。
- `PlayerBindings.ets:98-105`：`bindStable` 只写稳定层；活动槽且 gen>0 才重注入。
- `PlayerBindings.ets:150-167`：`unbind(sid)` 用 `SurfaceRegistry` 的 `bySid` 索引清槽，并取消等待者。
- `PlayerBindings.ets:174-202`：`releaseForSid` 先查索引；活动槽只有在 `getAppliedSurfaceId()===sid` 时才 `applySurface('',false)`，避免迟到的页面解绑清掉刚绑定的稳定层。
- `SurfaceRegistry.ets:23-45`：`pages[]` 与 `stable[]` 分开存，`setStable` 的 `gen` 固定 0 但只作为存储；注入时用域内当前 gen。

**v399（S3/拖动接管）[文档 + 观测]**：

- `DOC/DEVLOG.md:93`（v399-DragTakeover）：拖动时切 IJK，松手后 AV 在**非活动槽独立 Surface** prepare，AV `seekDone` 触发 Store SwapBackend；`pool.commit` + 页面 `activeSurface` 翻槽 + `afterCommit` + 释放 IJK。
- `DOC/DEVLOG.md:94`（v399-S3-Assembly）：新增 `PlayerAssembly` 唯一组合根；`SurfaceOpsCtl` 从 `PlayerService` 切出，与 `PlaybackOpsCtl`/`LeaseOpsCtl` 分片。

对应现状：

- `PlayerAssembly.ets:165-172` 构造 `SurfaceOpsCtl`，注入 `currentGen`。
- `SurfaceOpsCtl.ets:82-112`：
  - `bindSlotSurface` → `bindStable`（稳定层）；
  - `bindPageSurface` → `bindRenderTarget`，gen 由域内 `currentGen()` 提供，UI 不持有 gen；
  - `unbindSurface` → `releaseForSid`；
  - `bindRenderTarget` 只做 `bindPage`/`bindStable`。
- `PlayerBindings.ets:107-121`：`afterCommit` 清旧活动槽页面登记，按 uri/gen 找回句柄并 `inject`，再 `pump` 唤醒等待者。
- `PlayerPage.ets:119-137`：页面 `aboutToDisappear` 最后一步 `dispatchRelease('page-disappear')`；`PlayerPage.ets:282-289` 只是发 `UiIntentKind.Release` 意图。
- `PlayerLifecycleCtl.ets:59-87`：release 同步段先 `bindings.clearAll()` + `surfaceAuthority.clear()`，再把拆卸体交给 Store 的 Release 槽。
- `SurfaceAuthority.ets:75-89`：唯一向下写点；`applySurface(id, attachNow)` 之外，只有 `attachNow && nativeContext` 时才 `attachIjkContext`。

**边界结论 [观测]**：v383/v399 接管的是**逻辑句柄（surfaceId + uri + gen + slot）**，不是 `OHNativeWindow` 本身的释放；真正的 window 生命周期仍完全在 IJK NAPI/`libijksdl` 内部（见 §9）。

### 1.3 P4：`native_setup(VideoCodecMode.SURFACE)` 前后的初始化约束

**历史记录 [文档]**：

- `DOC/PERFORMANCE-BASELINE.md:14-16`：P4 起主线使用官方完整版 `libavcodec.so`（含 `h264/hevc_ohcodec`）；播放模式必须 `SURFACE`；`setOption(mediacodec*)` 必须在 `native_setup()` 之后，否则 `mp NULL`、硬解静默失效、`onPrepared` 不回调。
- `DOC/PERFORMANCE-BASELINE.md:61-68`：同流同解码器 BUFFER 66.8% CPU / SURFACE 9.7% CPU；验收线要求 SURFACE 相对 BUFFER ≥5×。
- `DOC/PERFORMANCE-BASELINE.md:122-123`：SURFACE 下禁止 `screenshot()`；`setOption(mediacodec*)` 顺序禁令。
- `DOC/evidence/v401/p4-audit.md:31-51`：`setContext → native_setup(SURFACE) → setOption` 顺序合规；全仓 `native_setup` 只有主播放器 SURFACE 与缩略图 BUFFER 两处。

**当前代码 [观测]**：

- `IjkPlayerBackend.ets:209-218`：
  1. `p.setContext(this.ctx, this.xid)`（`ctx` 是 XComponent `onLoad` 的 context 对象）；
  2. `p.setDebug(IJK_DIAG)`；
  3. `p.native_setup(VideoCodecMode.SURFACE)`；
  4. `p.setOption(... 'mediacodec-all-videos','1')` / `'mediacodec-hevc','1'`。
- `IjkMediaPlayer.ets:115-120`：`setContext` 只是把 JS 侧 NAPI 模块对象赋给 `this.ijkplayer_napi`，**不触发 native 调用**。
- `IjkMediaPlayer.ets:682-689`：`native_setup(mode)` 记住 `codecMode`，再调 `_native_setup(id, codecMode===SURFACE)`。
- `IjkFrameProvider.ets:113`：缩略图实例显式 `native_setup()`（默认 BUFFER），代码注释写明“screenshot 要求非 SURFACE”。

**P4 约束的真实含义 [观测 + 推断]**：

1. `setContext` 必须先于 `native_setup`：否则 `ijkplayer_napi` 为 null，`native_setup` 直接跳过，后续 `setOption` 也没有 native 实例可写。
2. `native_setup` 必须先于 `setOption`：`native_setup` 才创建 `IjkMediaPlayer` 原生实例并插入全局 map；之前 `setOption` 找不到 mp。
3. `native_setup(SURFACE)` 必须在 **XComponent 的 native `OnSurfaceCreated` 之后**：它从 `nativeWindowMap_` 取 window。取不到时：
   - `ffpipeline_set_surface(..., NULL)` → `ff_ohcodec_try_setup_surface_hw_device()` 返回 1 → `output_to_window=false`，零拷贝不成立；
   - `SDL_VoutAndroid_SetAndroidSurface(vout, NULL)` → vout 的 `native_window` 也为空 → `func_display_overlay_l()` 在 `ijksdl_vout_android_nativewindow.c:144-149` 直接返回 -1。
   - 结果是**无画面（黑屏）或等待后续重新绑定**，而不是“自动退化为可看的 Buffer 模式”。
4. 官方完整版 `libavcodec.so` 是前提：hvigor 的 `PrepareIjkCodecLib` 在 `DoNativeStrip` 后、`PackageHap` 前强制覆盖并校验 `h264_ohcodec`/`hevc_ohcodec`（`entry/hvigorfile.ts:9-38,48-53`）。
5. SURFACE 下 `screenshot()` / `startRecord()` 直接返回 false（`IjkMediaPlayer.ets:691-720`）。

### 1.4 历史雷区清单（重构前必须保留的约束）

| # | 雷区 | 约束 |
|---|---|---|
| H1 | AVPlayer 运行中 `surfaceId` 解绑/重绑 = `5400102` | 不要在 `AvPlayerBackend` 恢复 v393 的 `detach-before-attach`；当前幂等写入口（`nativeSurfaceId`）是已验收方案 |
| H2 | `applyScaleType` 不能提前到 `initialized` | 只能在 `applyPrepared`/合法状态后设置（v393-FlickerFix-Revert） |
| H3 | 页面 Surface 解绑不能清掉稳定层记账 | `releaseForSid` 的 `getAppliedSurfaceId()===sid` 守卫必须保留（v383-P1） |
| H4 | gen=0 注入会让预加载/页面 Surface 串号 | `bindPage`/`bindStable` 的 `gen>0` 拒绝必须保留（v383-P2） |
| H5 | IJK `setOption(mediacodec*)` 早于 `native_setup` 会静默失效 | 顺序必须是 `setContext → native_setup(SURFACE) → setOption`（v401/P4） |
| H6 | 无 `ohcodec` 的 `libavcodec.so` 会让 SURFACE 退化为软解/高 CPU | 构建期必须保留 `PrepareIjkCodecLib` 的符号校验（`entry/hvigorfile.ts:15-25`） |
| H7 | SURFACE 下 `screenshot()` 永远失败 | 缩略图必须继续走独立 BUFFER 实例（`IjkFrameProvider.ets:113`） |
| H8 | 换面后 `startRenderFrame` 不保证重发 | 不要假设“换 surface 会自动再发首帧”（v399 全屏审计结论） |

---

## 2. 产物与组成：到底哪个“零拷贝 .so”

### 2.1 命名核对

- 全盘未找到文件名为 `SURFACE 零拷贝.so` 的产物（`D:\devlop` 下按文件名 `*SURFACE*`/`*零拷贝*` 过滤，0 命中）。
- Zplayer 仓库内**没有** `entry/src/main/cpp`；`git ls-files` 的 C/C++ 文件只包含 `libs/ohos_libsmb`（SMB 探针/会话）、`poc/` 下的 SMB 试验代码，以及 `tools/build-ffmpeg-ohos-dav1d/ohcodec_stub.c`（历史 stub），**没有任何 IJK 或 FFmpeg 解码/渲染源码**。
- 因此“SURFACE 零拷贝.so”不是本仓库独立编译的库；它是第三方包 `@ohos/ijkplayer@2.0.11` 的预编译产物组合。版本锁定见 `entry/oh-package.json5`（`"@ohos/ijkplayer": "2.0.11"`）与 `libs/ohos_ijkplayer/oh-package.json5`（wrapper 同样依赖 2.0.11）。

### 2.2 实际加载的二进制（[观测]，SHA-256 前 12 位）

| 文件 | 大小 | 哈希前缀 | 作用 |
|---|---:|---|---|
| `entry/oh_modules/@ohos/ijkplayer/libs/arm64-v8a/libavcodec.so` | 12,565,832 | `67876E843D86` | FFmpeg fork + `ohcodec` 解码器；`OH_VideoDecoder_SetSurface/RenderOutputBuffer` 的调用者 |
| `.../libijkplayer.so` | 307,168 | `F1F48C92EE32` | IJK 主库；`ff_ffplay.c` 的 surface 硬件设备挂接、强制 `h264/hevc_ohcodec` |
| `.../libijksdl.so` | 303,480 | `38978365B63D` | 视频输出层；`SDL_FCC__HAVC` 直出分支、`av_ohcodec_release_buffer` 调用者 |
| `.../libijkplayer_napi.so` | 130,992 | `62A6EC6A370C` | NAPI/XComponent 生命周期；`native_setup`、`OnSurfaceCreated/Destroyed` |
| `.../libavutil.so` | — | — | `AVOHCodecDeviceContext`（含 `native_window`）与 hwcontext |

关键一致性检查：

- `entry/build/default/intermediates/libs/default/arm64-v8a/` 下三个主库哈希与上表一致 → 打进的 HAP 就是这套二进制。
- `libavcodec.so` 与 `tools/build-ffmpeg-ohos-dav1d/prebuilt/arm64-v8a/libavcodec.so` 哈希一致（`67876E843D86`），该文件已由 `e1c1f44` 入库（`a655858` 起走 Git LFS），并由 hvigor 强制同步。
- 该 `libavcodec.so` 含字符串 `h264_ohcodec`、`hevc_ohcodec`、`av_ohcodec_release_buffer`、`OH_VideoDecoder_SetSurface`、`OH_VideoDecoder_RenderOutputBuffer`；NEEDED 含 `libnative_media_vdec.so`、`libnative_media_venc.so`、`libnative_media_codecbase.so`、`libnative_media_core.so`、`libnative_window.so`。

> 对照：`D:\devlop\ZplayerDemo\v402-libs\arm64-v8a\` 是另一个**自定义 SMB/HAR 构建**（`libavcodec.so` 12,496,056，哈希 `061BE8E7347F`；`libijksdl.so` 269,472，哈希 `5DC0EA03DB0C`；`libijkplayer.so` 306,624，哈希 `48BE7DB8230C`）。Zplayer HAP 中这三个主库的哈希均与自定义构建不同，**当前主线不使用该自定义构建**。

### 2.3 SURFACE 相关源文件（vendor 源码树，非 Zplayer 编译）

> Zplayer 不含这些源码。下列行号来自本机与 2.0.11 二进制日志字符串逐条对上的 vendor 树：`D:\devlop\ZplayerDemo\native\src\ohos_ijkplayer` 与 `...\ohos_ffmpeg`。二进制中存在完全相同的日志串（如 `OHCODEC av_hwdevice_ctx_create failed for %s`、`ohcodec: release output buffer index=%u failed %d`、`Set surface failed, %d, %s`），可证明代码路径同源；但**不能证明与 2.0.11 发布源码逐字相同**（2.0.11 源码未入库，见 §10）。

| 库 | 文件 | SURFACE 相关函数 |
|---|---|---|
| `libijkplayer.so` | `ijkplayer/ff_ffplay.c` | `ff_ohcodec_try_setup_surface_hw_device()`（3469-3504）、`ff_ohcodec_get_format()`（3455-3463）、`stream_component_open()` 强制 ohcodec（3545-3553、3590-3594）、`video_image_display2()`（1195-1245）、`video_refresh()`（1637-1706） |
| `libijkplayer.so` | `ijkplayer/pipeline/ffpipeline_android.c` | `ffpipeline_set_surface()`（183-190）、`ffpipeline_get_surface()`（192-197）、`func_open_video_decoder()`（64-68） |
| `libijkplayer.so` | `ijkplayer/ijkplayer_android.c` | `ijkmp_android_set_surface_l()`（55-62） |
| `libijkplayer.so` | `ijkplayer/ff_ffplay.c` | `ffp_destroy()`（4764-4790，vout/pipeline 释放顺序） |
| `libijksdl.so` | `ijksdl/video/ijksdl_vout_android_nativewindow.c` | `SDL_VoutAndroid_CreateForANativeWindow()`（210-235）、`SDL_VoutAndroid_SetNativeWindow_l()`（237-261）、`func_display_overlay_l()`（138-196）、`SDL_VoutOhos_releaseBufferProxyP_l()`（314-333）、`func_free_l()`（88-136） |
| `libijksdl.so` | `ijksdl/video/ijksdl_vout_overlay_ohos_avcodec.c` | `func_fill_frame()`（95-144）、`SDL_VoutOverlayOhosAVCodec_releaseFrame_l()`（190-197） |
| `libavcodec.so` | `libavcodec/ohdec.c` | `oh_decode_release()`（92-101）、`oh_decode_set_format()`（145-201）、`oh_decode_start()`（365-399）、`oh_decode_init()`（401-428）、`oh_decode_close()`（433-462）、`av_ohcodec_release_buffer()`（492-528）、`oh_decode_wrap_hw_buffer()`（530-590）、`oh_decode_wrap_sw_buffer()`（592-686）、`oh_decode_output_frame()`（688-724） |
| `libavutil.so` | `libavutil/hwcontext_oh.c/.h` | `AVOHCodecDeviceContext { native_window }`（`hwcontext_oh.h:27-32`） |
| `libijkplayer_napi.so` | `napi/ijkplayer_napi.cpp` | `native_setup()`（319-341）、`setXComponentAndNativeWindow()`（1097-1114）、`onSurfaceCreatedCB()`（1140-1156）、`onSurfaceDestroyedCB()`（1181-1221）、`getInstance()`（1279-1292）、`RegisterCallback()`（1304） |
| `libijkplayer_napi.so` | `proxy/ijkplayer_napi_proxy.cpp` | `IJKPlayerNapiProxy::IjkMediaPlayer_native_setup()`（303-320）、`set_media_player()`（277-292）、`IjkMediaPlayer_release()`（464-475）、`IjkMediaPlayer_reset()`（477-489） |

### 2.4 死代码与“未编入”的 OHOS 解码节点

- `ijkplayer/CMakeLists.txt:11-33` 的源文件列表**只有** `pipeline/ffpipenode_ffplay_vdec.c`，没有 `ohos/ffpipenode_ohos_mediacodec_vdec.cpp` 与 `ohos/ohos_video_decoder.cpp`。
- `ffpipeline_android.c:64-68` 的 `func_open_video_decoder()` 固定返回 `ffpipenode_create_video_decoder_from_ffplay(ffp)`；`func_init_video_decoder()` / `func_config_video_decoder()` 的 OHOS mediacodec 分支被整段注释（92-96、103-108）。
- 结论：仓库里的 `ohos_video_decoder.cpp`（215 行，含 `OH_VideoDecoder_SetSurface`/`RenderOutputBuffer`）与 `ffpipenode_ohos_mediacodec_vdec.cpp` 是**保留但未编入**的死代码；实际 SURFACE 走 FFmpeg `ohcodec`，不要按这两个文件估算改造量。

---

## 3. 从 XComponent 到 `OH_VideoDecoder_SetSurface` 的完整调用链

### 3.1 ArkTS 层：谁拿到 surface/context

| 位置 | 事实 |
|---|---|
| `pages/player/VideoSurface.ets:119-124` | `XComponent({ id: this.surfaceKey, type: SURFACE, libraryname: this.allowIjk ? 'ijkplayer_napi' : '', controller })` |
| `pages/player/VideoSurface.ets:137-147` | `.onLoad((context: object) => { sid = controller.getXComponentSurfaceId(); ... onSurface({ surfaceId: sid, nativeContext: context, renderId: surfaceKey }, slot) })` |
| `pages/player/VideoSurface.ets:148-155` | `.onDestroy()` 只清 `currentSid` 并打日志；**不释放 native window** |
| `pages/player/ZIVPlayerLayer.ets:370-388` | 页面级 XComponent（**没有 `libraryname`**）；`onLoad` 取 sid 后 `rebind()`；`onDestroy` → `releaseSurface()` |
| `pages/player/ZIVPlayerLayer.ets:95-105` | `releaseSurface()` 用空 `surfaceId` + `owner=旧 sid` 走 `attachSurface` 解绑 |
| `pages/Index.ets:74-80` | 缩略图专用 XComponent（`libraryname='ijkplayer_napi'`），`onLoad` 调 `bindIjkThumbContext(context, THUMB_IJK_ID)` |
| `pages/player/VideoStage.ets:97-139` | 两个 `VideoSurface`（slot 0/1），`allowIjk=IJK_ENABLED`；`onSurface` 上报 `SurfaceHandle` + slot |

> HarmonyOS ArkTS 没有名为 `OnSurfaceCreate` 的回调；ArkTS 侧是 `.onLoad`，Native 侧是 `OnSurfaceCreated`。

### 3.2 句柄如何进入域层

```
VideoSurface.onLoad
  → VideoSurface.ets:146  onSurface({surfaceId, nativeContext, renderId}, slot)
  → VideoStage.ets:108-112 / 135-139  onSurface(h, slot)
  → PlayerLayout.ets:87-89  events.onSurface
  → PlayerPage.ets:471      this.surfaces.bind(slot, h)
  → SlotSurfaces.ets:14-23  p.attachSurface(slot, h, '')
  → PlayerService.ets:39-41 → PlayerAssembly.attachSurface()
```

`PlayerAssembly.attachSurface()`（`PlayerAssembly.ets:431-441`）：

- `handle.surfaceId === ''` → `surface.unbindSurface(owner)`；
- `owner === ''` → `surface.bindSlotSurface(slot, handle)`（稳定层）；
- 否则 `surface.bindPageSurface(handle, owner)`（页面层，owner=uri）。

稳定层路径继续：

```
SurfaceOpsCtl.bindSlotSurface            (SurfaceOpsCtl.ets:82-84)
  → PlayerBindings.bindStable            (PlayerBindings.ets:98-105)
    → SurfaceRegistry.setStable          (SurfaceRegistry.ets:35-45)
    → 若活动槽且 gen>0: inject(handle, gen)
      → PlayerBootstrapper.updateSurface (PlayerBootstrapper.ets:279-284)
        → SurfaceAuthority.set           (SurfaceAuthority.ets:39-58)
        → SurfaceAuthority.apply(true)   (SurfaceAuthority.ets:75-89)
          ├─ sink.applySurface(id, true) → IjkPlayerBackend.applySurface (IjkPlayerBackend.ets:105-107)
          └─ sink.attachIjkContext(ctx, renderId) → IjkPlayerBackend.attachIjkContext (136-139)
```

页面层路径：

```
SurfaceOpsCtl.bindPageSurface (SurfaceOpsCtl.ets:87-90)
  → bindRenderTarget(handle, currentGen, uri) (98-112)
  → PlayerBindings.bindPage (62-87)
    → intent.resolveSlot(uri,gen)
    → registry.setPage
    → 若活动槽: inject(handle,gen) →（同上）
```

**关键事实 [观测]**：

- `SurfaceHandle` 同时塞了两套语义：`surfaceId`（AVPlayer 用的字符串）与 `nativeContext`/`renderId`（IJK 用的 XComponent 对象/ID）。定义见 `kernel/player_contract.ets:30-34`。
- **IJK 不使用 `surfaceId` 字符串**：`IjkPlayerBackend.applySurface` 只是保存它（105-107），`prepare()` 用的是 `this.ctx` 与 `this.xid`（209-214）。真正的 window 来自 `nativeContext`。

### 3.3 NAPI 层：surface/window 如何进入原生

**入口不是 `setContext`，而是 `_native_setup`**：

- `IjkMediaPlayer.ets:682-689` → `this.ijkplayer_napi._native_setup(this.id, true)`。
- NAPI 导出名：`napi/ijkplayer_napi.cpp:1338` `DECLARE_NAPI_FUNCTION("_native_setup", IJKPlayerNapi::native_setup)`。
- 实现：`IJKPlayerNapi::native_setup()`（`ijkplayer_napi.cpp:319-341`）：
  1. 解析 `xcomponentId`；
  2. `getXComponent(id)`（返回 `OH_NativeXComponent*`）；
  3. `getNativeWindow(id)`（返回 `void*`，即 `OHNativeWindow*`）；
  4. `IjkMediaPlayer_native_setup(xcomponent, nativeWindow, isSurface)`。

**window 是在 XComponent 原生回调里缓存的**：

- `onSurfaceCreatedCB()`（`ijkplayer_napi.cpp:1140-1156`）：`OH_NativeXComponent_GetXComponentId` → `getInstance(id)->onSurfaceCreated(component, window)` → `setXComponentAndNativeWindow(id, component, window)`。
- `setXComponentAndNativeWindow()`（1097-1114）：把 `OH_NativeXComponent*` 与 `void* window` 写入 `nativeXComponentMap_` / `nativeWindowMap_`。
- 注册点：`IJKPlayerNapi` 构造函数（1266-1274）设置 `OnSurfaceCreated/Changed/Destroyed`，`1304` `OH_NativeXComponent_RegisterCallback`。
- **没有 `OH_NativeWindow_CreateFromSurface` / `OH_NativeWindow_CreateNativeWindowFromSurfaceId` 调用**（全仓 + vendor 树 grep 0 命中）。window 是 XComponent 回调直接给的，不是从 surfaceId 创建的。

**创建原生播放器并绑定 surface**：

- `IJKPlayerNapiProxy::IjkMediaPlayer_native_setup()`（`ijkplayer_napi_proxy.cpp:303-320`）：
  - `ijkmp_android_create(message_loop)`；
  - `set_media_player(id_, mp)`（全局 map）；
  - `ijkmp_android_set_surface(mp, nativeWindow, isSurface)`（314）；
  - `GLOBAL_NATIVE_WINDOW = nativeWindow`（311，裸指针全局变量）。
- `ijkmp_android_set_surface_l()`（`ijkplayer_android.c:55-62`）：
  - `SDL_VoutAndroid_SetAndroidSurface(vout, native_window)`；
  - `ffpipeline_set_surface(pipeline, isSurface ? native_window : NULL)`。

### 3.4 IJK 层：SURFACE 如何变成硬解直出

- `ffpipeline_set_surface()`（`ffpipeline_android.c:183-190`）只把裸指针存进 `IJKFF_Pipeline_Opaque.native_window`。
- `stream_component_open()`（`ff_ffplay.c:3507` 起）：
  - `ffp_should_use_oh_hw_vdec()`（3441-3453）在 `mediacodec_all_videos`/`mediacodec_hevc` 命中时返回 1；
  - 强制 `h264_ohcodec` / `hevc_ohcodec`（3545-3553）；
  - 打开解码器前调用 `ff_ohcodec_try_setup_surface_hw_device()`（3590-3594）。
- `ff_ohcodec_try_setup_surface_hw_device()`（3469-3504）：
  - `ffpipeline_get_surface(ffp->pipeline)` 取 window（3482）；
  - 没有 window 就 `return 1`（跳过，退化为 Buffer 拷贝）；
  - `av_hwdevice_ctx_create(AV_HWDEVICE_TYPE_OHCODEC)`（3486）；
  - `oh_dev->native_window = native_window`（3496）；
  - `avctx->hw_device_ctx = hw_device_ctx`、`avctx->get_format = ff_ohcodec_get_format`（3499-3501）。

### 3.5 OHCodec 层：`OH_VideoDecoder_SetSurface` 的调用点与时序

- `oh_decode_init()`（`ohdec.c:401-428`）顺序：
  1. `oh_decode_create()`（411）；
  2. `oh_decode_set_format()`（414）；
  3. FIFO 分配（418-424）；
  4. `oh_decode_start()`（426）。
- `oh_decode_set_format()`（145-201）：
  - 从 `avctx->hw_device_ctx` 取 `AVOHCodecDeviceContext.native_window`（150-155），置 `s->output_to_window = true`；
  - `OH_VideoDecoder_Configure()`（181）；
  - `OH_VideoDecoder_SetSurface(s->dec, window)`（191）。
- `oh_decode_start()`（365-399）：
  - `OH_VideoDecoder_RegisterCallback()`（376）；
  - `OH_VideoDecoder_Prepare()`（383）；
  - `OH_VideoDecoder_Start()`（390）。

**时序结论 [观测]**：`Configure → SetSurface → RegisterCallback → Prepare → Start`，**SetSurface 严格在 Prepare 之前**，符合官方要求。死代码 `ohos_video_decoder.cpp:49-67` 也是 `Configure → SetSurface → SetCallback → Prepare`，但未编入。

### 3.6 端到端调用链（单行版）

```text
[XComponent 创建]
  → ijkplayer_napi.cpp:1140 onSurfaceCreatedCB
  → ijkplayer_napi.cpp:1155 setXComponentAndNativeWindow
  → ijkplayer_napi.cpp:1097-1114 nativeWindowMap_[id] = window

[ArkTS onLoad]
  VideoSurface.ets:137 .onLoad(context)
  → VideoSurface.ets:146 onSurface({surfaceId, nativeContext, renderId}, slot)
  → VideoStage.ets:108-112
  → PlayerLayout.ets:87-89
  → PlayerPage.ets:471
  → SlotSurfaces.ets:14-23
  → PlayerService.ets:39-41
  → PlayerAssembly.ets:431-441
  → SurfaceOpsCtl.ets:82-84 (稳定层) / 87-90 (页面层)
  → PlayerBindings.ets:98-105 (bindStable) / 62-87 (bindPage)
  → PlayerBootstrapper.ets:221 / 279-284
  → SurfaceAuthority.ets:82-83
  → IjkPlayerBackend.ets:136-139 attachIjkContext(ctx, renderId)

[prepare]
  IjkPlayerBackend.ets:209-214
  → IjkMediaPlayer.ets:115-120 setContext
  → IjkMediaPlayer.ets:682-689 native_setup(SURFACE)
  → IjkMediaPlayer.ets:685 _native_setup(id, true)
  → ijkplayer_napi.cpp:1338 (NAPI export) → 319-341 native_setup
  → ijkplayer_napi.cpp:332-333 getXComponent/getNativeWindow
  → ijkplayer_napi_proxy.cpp:303-320 IjkMediaPlayer_native_setup
  → ijkplayer_napi_proxy.cpp:314 ijkmp_android_set_surface(mp, window, true)
  → ijkplayer_android.c:55-62
       ├─ SDL_VoutAndroid_SetAndroidSurface(vout, window)
       └─ ffpipeline_set_surface(pipeline, window)
  → ffpipeline_android.c:183-190 (opaque->native_window = window)
  → ff_ffplay.c:3590-3594 stream_component_open
  → ff_ffplay.c:3469-3504 ff_ohcodec_try_setup_surface_hw_device
  → ff_ffplay.c:3482 ffpipeline_get_surface
  → ff_ffplay.c:3496 oh_dev->native_window = native_window
  → ohdec.c:401-428 oh_decode_init
  → ohdec.c:145-201 oh_decode_set_format
  → ohdec.c:181 OH_VideoDecoder_Configure
  → ohdec.c:191 OH_VideoDecoder_SetSurface(s->dec, window)
  → ohdec.c:383 OH_VideoDecoder_Prepare
```

---

## 4. 解码帧到 Surface 的数据通路

### 4.1 帧如何产生、是否经过 CPU

`oh_decode_output_frame()`（`ohdec.c:688-724`）：

- `s->output_to_window == true` → `oh_decode_wrap_hw_buffer()`（721-722）；
- 否则 → `oh_decode_wrap_sw_buffer()`（723）。

`oh_decode_wrap_hw_buffer()`（530-590）：

- `frame->format = AV_PIX_FMT_OHCODEC`（545）；
- `OHCodecBuffer` 只保存 `index`、`OH_AVBuffer*`、`dec_ref`、`released` 原子量（546-560）；
- `frame->buf[0] = av_buffer_create(NULL, 0, oh_buffer_release, buffer, READONLY)`（571）；
- `frame->data[3] = (uint8_t*)buffer`（585）；
- `frame->pts = av_rescale_q(attr->pts, AV_TIME_BASE_Q, avctx->pkt_timebase)`（586）。

**没有 `OH_AVBuffer_GetAddr()`、没有 `av_image_copy2()`、没有 `memcpy` 像素**。像素由 OHCodec 直接写入 `OHNativeWindow`；IJK 只拿 buffer index/句柄。

对照 Buffer 路径：`oh_decode_wrap_sw_buffer()`（592-686）在 652 行 `OH_AVBuffer_GetAddr()`，在 673 行 `av_image_copy2()` 拷贝到 AVFrame，再经 EGL 上传。这正是 BUFFER 模式 CPU 高的原因（`DOC/PERFORMANCE-BASELINE.md:61-68`）。

### 4.2 IJK 渲染回调是否被绕过

**是，SURFACE 格式走独立 overlay，GLES 被主动终止**：

- `ff_ffplay.c:2001-2016`：`src_frame->format == AV_PIX_FMT_OHCODEC` 时 `av_frame_ref` 到 `vp->frame`，再 `SDL_VoutFillFrameYUVOverlay(vp->bmp, fill_frame)`；不拷贝像素。
- `ijksdl_vout_android_nativewindow.c:63-78`：`func_create_overlay_l` 对 `AV_PIX_FMT_OHCODEC` 返回 `SDL_VoutOhosAVCodec_CreateOverlay()`。
- `ijksdl_vout_overlay_ohos_avcodec.c:95-144`：`func_fill_frame` 只把 `AVOHCodecBuffer*` 包成 `SDL_OhosAVCodecBufferProxy`，并设置 `pixels[0]=NULL`、`pitches[0]=0`、`format=SDL_FCC__HAVC`。
- `ff_ffplay.c:1240`：`SDL_VoutDisplayYUVOverlay(ffp->vout, vp->bmp)`。
- `ijksdl_vout_android_nativewindow.c:169-172`：`case SDL_FCC__HAVC:` → `IJK_EGL_terminate(opaque->egl)` → `SDL_VoutOverlayOhosAVCodec_releaseFrame_l(overlay, true)`。
- `ijksdl_vout_overlay_ohos_avcodec.c:190-197` → `SDL_VoutOhos_releaseBufferProxyP_l(..., render=true)`。
- `ijksdl_vout_android_nativewindow.c:314-333` → `av_ohcodec_release_buffer((*proxy)->ohcodec_buf, 1)`。
- `ohdec.c:492-528`：`render` 为真 → `OH_VideoDecoder_RenderOutputBuffer(dec, index)`（516）；否则 `FreeOutputBuffer`（518）。

**结论 [观测]**：SURFACE 路径的实际“渲染”就是 `RenderOutputBuffer`；IJK 原有 `IJK_EGL_display` 分支只在 `SDL_FCC_RV24/I420/I444P10LE/NV12/YV12/RV16/RV32` 等格式命中（`ijksdl_vout_android_nativewindow.c:173-190`），SURFACE 帧格式是 `SDL_FCC__HAVC`，不会进入这些分支。

**死代码/保留代码**：

- `SDL_FCC__AMC` 分支直接返回 NULL（164-168）；
- `ffpipenode_ohos_mediacodec_vdec.cpp` / `ohos_video_decoder.cpp` 未编入（§2.4）；
- GLES 渲染代码仍保留，供 BUFFER/软解路径（缩略图）使用，不是死代码，但对 SURFACE 不生效。

### 4.3 帧同步与“已解码未上屏”状态

- FFmpeg 输出先进入 `s->output_queue`（FIFO 分配在 `ohdec.c:418-424`；输出回调写入在 `ohdec.c:346-363`），由 `oh_decode_output_frame` 取帧。
- IJK 侧进入 `pictq` 帧队列：`ffp->pictq_size = VIDEO_PICTURE_QUEUE_SIZE_DEFAULT`（`ff_ffplay_def.h:183-186`，默认 3，最大 16），初始化在 `ff_ffplay.c:4446`。
- 送显节拍由 `video_refresh()`（`ff_ffplay.c:1637-1706`）控制：`compute_target_delay()` 计算延迟（1581），`frame_timer` 控制；`update_video_pts()` 在 1698 用 `vp->pts` 更新时钟；`video_image_display2()` 在 1195-1245 真正送显。
- PTS 来自 OHCodec buffer attr（`ohdec.c:586`），不是 IJK 重新生成的。

**“已解码但尚未上屏”的可观测状态 [观测]**：

- `pictq` 中 `frame_queue_nb_remaining() > 0` 即存在已解码、未调用 `RenderOutputBuffer` 的帧；
- 这些帧持有 `AVOHCodecBuffer` 的 `dec_ref`，因此解码器不会立即 Destroy；
- 应用层可间接观察：`IjkPlayerBackend.hasLiveFrames()`（`IjkPlayerBackend.ets:366-373`）比较 `getVideoDecodeFramesPerSecond()` 与 `getVideoOutputFramesPerSecond()`；两者不等说明“解了但没出/出得慢”；
- `MEDIA_INFO_VIDEO_RENDERING_START`（`IjkPlayerBackend.ets:239-241`）只表示首帧送显，不表示每次换面都有新帧（v399 审计已证，`DOC/evidence/v399/fullscreen-surface-audit.md:191-195`）。

**没有额外的自定义队列**：SURFACE 路径没有 Zplayer 自己维护的帧队列；队列都在 IJK/FFmpeg 内部（OHCodec output queue、pictq、codec 内部 buffer queue）。`OH_VideoDecoder_RenderOutputBufferAtTime` 未被使用，A/V 同步仍由 ffplay 时钟驱动（`DOC/evidence/v400/ijk-surface-feasibility.md:148`）。

---

## 5. 耦合方式判定（a / b / c）

### 5.1 判定：a（vendor 层）+ c（Zplayer 集成层），不是 b

| 选项 | 判定 | 证据 |
|---|---|---|
| **a) 修改 IJK Native 源码** | **在第三方 .so 内成立；Zplayer 本仓库不成立** | vendor 的 `ff_ffplay.c` 有 SURFACE 分支（3469-3504、3545-3553、3590-3594）；`ijksdl` 有 `SDL_FCC__HAVC` 直出分支（169-172）与 `SDL_VoutOhos*`；`ohdec.c` 有 `output_to_window`/`SetSurface`/`RenderOutputBuffer`。但 Zplayer 没有 `entry/src/main/cpp`，`git ls-files` 中无 IJK/FFmpeg 解码/渲染源码（只有 SMB 探针、poc 与 `ohcodec_stub.c`）→ 这些修改不是 Zplayer 做的，是官方 2.0.11 包自带的 vendor fork。 |
| **b) NAPI 旁路自己接管解码/渲染** | **否** | Zplayer 没有创建自己的 `OHCodec` 实例，也没有独立 Surface 送显；它明确调用 `p.native_setup(VideoCodecMode.SURFACE)`（`IjkPlayerBackend.ets:214`），解码/渲染都在 IJK 内。 |
| **c) 通过标准接口挂接，IJK 主库未（由 Zplayer）改动** | **是（Zplayer 集成层）** | 只使用公开 ArkTS API：`IjkMediaPlayer`、`VideoCodecMode`、`setContext`、`native_setup`、`setOption`、监听器；没有 import 任何 native 头文件/私有符号。 |

**一句话**：从 Zplayer 看是 **c**；从“这个 .so 怎么实现零拷贝”看是 **a（vendor 修改过的 IJK/FFmpeg fork）**。两者不矛盾，但不能把 vendor 的修改算成 Zplayer 的 native 改动。

### 5.2 依赖反向检查（readelf / strings）

| 检查 | 结果 [观测] |
|---|---|
| `libijkplayer.so` / `libijksdl.so` 是否依赖 `libavcodec.so` | `NEEDED libavcodec.so`（两者都有） |
| `libijksdl.so` 是否引用零拷贝符号 | `UND av_ohcodec_release_buffer@LIBAVCODEC_62`（`llvm-readelf --dyn-syms`） |
| `libavcodec.so` 是否导出该符号 | `av_ohcodec_release_buffer@@LIBAVCODEC_62`（GLOBAL DEFAULT） |
| `libavcodec.so` 是否依赖 OH NDK | `NEEDED libnative_media_vdec.so`、`libnative_media_venc.so`、`libnative_media_codecbase.so`、`libnative_media_core.so`、`libnative_window.so`；`UND OH_VideoDecoder_SetSurface`、`UND OH_VideoDecoder_RenderOutputBuffer` |
| `.so` 是否依赖 IJK 私有头/结构体 | vendor 源码层：`ff_ffplay.c:88-90` include `libavutil/hwcontext_oh.h`、`libavcodec/ohdec_external.h`；`ff_ffplay.c` 用 `ffp->pipeline`/`ffpipeline_get_surface`；`ohdec.c:37,49,153` 用 `AVOHCodecDeviceContext`。这些是 FFmpeg fork/IJK 内部 ABI。Zplayer 层：无任何私有头依赖。 |

**结论**：IJK 主库确实引用了 FFmpeg fork 的非上游符号 `av_ohcodec_release_buffer`；反过来，FFmpeg fork 的 `ohdec.c` 依赖 IJK 传入的 `AVOHCodecDeviceContext.native_window`。这是一条**双向私有 ABI 耦合**，不是单纯“标准 FFmpeg 解码器”。

---

## 6. 若抽成独立 Sidecar：必须切断的耦合点

> 目标不是“删一个 .so”，而是把 **window 所有权、NAPI 实例生命周期、IJK vout 直出分支、OHCodec 解码器绑定** 四件事从 IJK 主库中切出来。按层列出。

### 6.1 ArkTS / 域层（Zplayer）

| # | 文件/函数 | 现状耦合 | 切断动作 |
|---|---|---|---|
| A1 | `pages/player/VideoSurface.ets:137-147` | `onLoad` 把 XComponent 的 `context: object` 直接塞进 `SurfaceHandle.nativeContext`，一路传到 NAPI | Sidecar 只接收显式 `nativeWindow` 句柄/ID；不再把 NAPI 模块对象当 window 传 |
| A2 | `kernel/player_contract.ets:30-34` | `SurfaceHandle` 混合 AV 的 `surfaceId` 与 IJK 的 `nativeContext/renderId` | 拆成 `RenderTarget { surfaceId }` 与 `NativeWindowHandle { handle, gen }` 两套契约 |
| A3 | `domains/player/surface/SurfaceAuthority.ets:50-83` | 保存 `nativeContext`，在 `apply(true)` 时调用 `sink.attachIjkContext` | 改为调用 Sidecar 的 `attach/detach`，不感知 IJK |
| A4 | `domains/player/IjkPlayerBackend.ets:105-107,136-139,209-214` | `applySurface` 只存 id；`attachIjkContext` 存 ctx；`native_setup(SURFACE)` 在 `prepare` 内完成绑定 | 把 window 绑定从 `prepare` 移到 Sidecar；`IjkPlayerBackend` 只保留播放器控制 |
| A5 | `pages/Index.ets:74-80` + `domains/thumbnail/IjkFrameProvider.ets:113` | 缩略图用第二个 `ijkplayer_napi` XComponent + BUFFER 模式 | Sidecar 需明确排除缩略图实例，或为 BUFFER 保留独立 NAPI 实例 |
| A6 | `pages/player/ZIVPlayerLayer.ets:370-388,95-105` | 页面级 XComponent 无 `libraryname`，但其 `onDestroy` 通过 `attachSurface('', sid)` 走同一套 Surface 解绑 | Sidecar 需区分“页面 Surface（AVPlayer）”与“IJK NativeWindow”，避免空 sid 解绑误伤 |
| A7 | `domains/player/PlayerBindings.ets:98-105,174-202` | 逻辑上把“稳定层/页面层”与“活动槽注入”绑定；`releaseForSid` 只清逻辑登记 | Sidecar 接管 native 生命周期后，这里应只做路由，不再承担 window 生命周期语义 |
| A8 | `domain/player/preload/IjkPreloadCtl.ets:204-230` | 预加载只 `applySurface(h.surfaceId,false)`，**不注入 nativeContext**（`attachIjkContext` 只在 `SurfaceAuthority.apply`） | Sidecar 需为预加载槽提供独立的 window attach，否则 IJK 预加载会没有 native window（见 §9 R10） |

### 6.2 NAPI 层（vendor `libijkplayer_napi.so`）

| # | 文件/函数 | 现状耦合 | 切断动作 |
|---|---|---|---|
| N1 | `napi/ijkplayer_napi.cpp:1097-1114` | `nativeXComponentMap_`/`nativeWindowMap_` 直接读写，无锁 | 由 Sidecar Bridge 独占 window 表，带锁 + refcount + generation |
| N2 | `napi/ijkplayer_napi.cpp:1140-1156` | `onSurfaceCreatedCB` 把裸 `window` 存 map | 改为 `sb_attach(id, window, w, h)`，由 Bridge 持有引用 |
| N3 | `napi/ijkplayer_napi.cpp:1181-1221` | `onSurfaceDestroyedCB` 直接 `delete instance` / `delete proxy` | 只 `sb_detach(id)`；player 释放由播放器生命周期驱动，不在 UI 回调里 delete |
| N4 | `napi/ijkplayer_napi.cpp:1238-1255` | `onSurfaceCreated/Changed/Destroyed` 空实现，player 不知道 window 失效 | 由 Bridge 发 `SB_EV_LOST/DETACHED` 事件，播放器决定重绑/重试 |
| N5 | `napi/ijkplayer_napi.cpp:319-341` | `native_setup` 从 map 取 window 并创建 player | 改为从 Bridge `sb_acquire` 取带引用的快照 |
| N6 | `proxy/ijkplayer_napi_proxy.cpp:303-320` | `GLOBAL_NATIVE_WINDOW` 裸指针；`native_setup` 创建 mp + 立即绑 surface | 删除全局裸指针；window 只存在 Bridge 中 |
| N7 | `proxy/ijkplayer_napi_proxy.cpp:277-292` | `set_media_player` 用 `insert`，同 id 第二次 `native_setup` 不会替换旧 mp | Sidecar 需要显式的 `replace/destroy` 语义，避免旧 window 残留（见 §9 R2） |
| N8 | `proxy/ijkplayer_napi_proxy.cpp:464-489` | `release()` 不置空 surface；只有 `reset()` 调 `ijkmp_android_set_surface(NULL)` | 统一 `detach_window()`，release/reset/页面退出都走同一入口 |

### 6.3 IJK 主库（vendor `libijkplayer.so` / `libijksdl.so`）

| # | 文件/函数 | 现状耦合 | 切断动作 |
|---|---|---|---|
| I1 | `ff_ffplay.c:3469-3504` | 从 `ffpipeline_get_surface` 拿裸 window，写进 `AVOHCodecDeviceContext.native_window` | 改为从 Sidecar `sb_acquire` 拿快照（带 gen/ref），用完 `sb_release` |
| I2 | `ff_ffplay.c:3545-3553` | 无 window 时仍强制 `h264/hevc_ohcodec`，静默退化 Buffer | 改为显式事件 `SB_E_NO_WINDOW`，由上层决定重试/切软解 |
| I3 | `pipeline/ffpipeline_android.c:183-197` | `IJKFF_Pipeline_Opaque.native_window` 裸指针 | 删除该字段，改为 Bridge 句柄 |
| I4 | `ijkplayer_android.c:55-70` | 同时把裸 window 写给 vout 和 pipeline | 只保留一次 `sb_attach`，由 vout/decoder 各自向 Bridge 取快照 |
| I5 | `ijksdl/video/ijksdl_vout_android_nativewindow.c:237-261` | `SDL_VoutAndroid_SetNativeWindow_l` 直接赋值裸指针，acquire/release 被注释 | 由 Bridge 负责 acquire/release；vout 不持有裸指针 |
| I6 | `ijksdl/video/ijksdl_vout_android_nativewindow.c:138-196` | `func_display_overlay_l` 直接解引用 `opaque->native_window` | 每次 display 前 `sb_acquire` 快照，detach 后返回 `SB_E_NO_WINDOW` 而不是野指针 |
| I7 | `ijksdl/video/ijksdl_vout_android_nativewindow.c:124-128` | `ANativeWindow_release` / `NativeLayerHandle(RELEASE_REF)` 被注释，window 从不归还 | 明确所有权：XComponent 给的 window 由 Bridge 引用/解引用 |
| I8 | `ijksdl/video/ijksdl_vout_overlay_ohos_avcodec.c:95-144,190-197` | overlay 持有 `AVOHCodecBuffer` proxy，release 时直接 `render=true` | 保留 buffer 侧逻辑，但补 generation 校验；detach 后只 Free 不 Render |
| I9 | `libavcodec/ohdec.c:145-201` | `SetSurface` 从 `AVOHCodecDeviceContext` 取裸 window | Sidecar 传入的 window 必须是已持有引用的快照；`ohdec` 不负责生命周期 |
| I10 | `libavcodec/ohdec.c:433-449` + `92-101` | `oh_decode_close` 只 Stop，`Destroy` 延迟到最后一个 `dec_ref` 释放 | Sidecar 需显式等待/隔离 in-flight buffer，避免 surface 先销毁、decoder 后 Destroy |
| I11 | `libavcodec/ohdec.c:492-528` | `RenderOutputBuffer` 在 `av_ohcodec_release_buffer` 内，可能晚于 surface 销毁 | 增加“surface generation/valid”校验，detach 后只 FreeOutputBuffer |
| I12 | `libavutil/hwcontext_oh.h:27-32` | `AVOHCodecDeviceContext.native_window` 是裸指针 | 作为 Sidecar ABI 的一部分显式版本化；不再暴露裸指针给 IJK 主库 |
| I13 | `entry/hvigorfile.ts:9-53` + `tools/.../prebuilt` | 构建期强制用官方 `libavcodec.so` 覆盖，Sidecar 若换库会被覆盖 | Sidecar 独立打包/独立 .so，禁止再依赖 `PrepareIjkCodecLib` 覆盖链 |

---

## 7. 生命周期与销毁路径

### 7.1 Surface 销毁（ArkTS → 域 → Native）

**ArkTS 侧 [观测]**：

- `VideoSurface.ets:148-155`：`onDestroy` 只 `currentSid=''` + 日志，**不调用任何释放接口**。
- `ZIVPlayerLayer.ets:385-388`：`onDestroy` → `releaseSurface()`（95-105）：
  - `d.surfaceAttach.attachSurface(this.index, { surfaceId: '' }, this.sid)`；
  - 域层 `PlayerAssembly.attachSurface`（431-435）看到空 id → `surface.unbindSurface(owner=旧 sid)`；
  - `SurfaceOpsCtl.unbindSurface`（92-96）→ `PlayerBindings.releaseForSid`（174-202）；
  - 活动槽若 `getAppliedSurfaceId()===sid`，才 `backend.applySurface('',false)`；
  - `IjkPlayerBackend.applySurface`（105-107）**只是把 `this.surfaceId=''`**，不调用任何 native 解绑。

**Native 侧 [观测]**：

- `onSurfaceDestroyedCB()`（`ijkplayer_napi.cpp:1181-1221`）：
  1. `destroyResource = true`；
  2. `instance->onSurfaceDestroyed(component, window)`（空实现，1252-1255）；
  3. 从 `ijkPlayerNapi_` map 擦除；
  4. `delete instance->ijkPlayerNapiProxy_`；
  5. `delete instance`。
- **没有** `ijkmp_android_set_surface(NULL)`，**没有** `IjkMediaPlayer_release()`，**没有** `OH_NativeWindow` 的 acquire/release。
- `IJKPlayerNapiProxy` 没有析构函数（`ijkplayer_napi_proxy.h:33-83`），`delete proxy` 不会清理 `GLOBAL_IJKMP`（`ijkplayer_napi_proxy.cpp:17`）。

**结论**：Surface 销毁时，native window 的“释放”实际上**不存在**；只有 NAPI 包装对象被 delete。window 指针仍留在 `GLOBAL_IJKMP` 里的 mp/ffplayer/vout/pipeline 中（如果之前没有显式 `player.release()`）。

### 7.2 解码器销毁与 Surface 释放的先后

**没有显式顺序保证 [观测]**：

- 正常退出：`PlayerPage.aboutToDisappear`（119-137）→ `dispatchRelease`（282-289）→ `PlayerLifecycleCtl.release`（59-87）→ `releaseRun`（89-139）→ `runTeardown` → `pool.releaseAll()`（`PlayerBackendPool.ets:68-72`）→ `IjkPlayerBackend.release()`（621-646）→ `p.stop()` + `p.release()`。
- 同时，ArkUI 会独立触发 `XComponent.onDestroy` → `onSurfaceDestroyedCB`。两者没有共同锁/顺序约束；谁先谁后由 UI 生命周期与 Store 宏任务调度决定。
- `IjkMediaPlayer_release()`（`ijkplayer_napi_proxy.cpp:464-475`）→ `ijkmp_shutdown` → `dec_ref` 归零 → `ffp_destroy`（`ff_ffplay.c:4764-4790`）：`SDL_VoutFreeP` 先于 `ffpipeline_free_p`；`func_free_l` 把 `opaque->native_window=NULL`（`ijksdl_vout_android_nativewindow.c:124-128`），但实际 `OHNativeWindow` 释放被注释。
- 解码器侧：`oh_decode_close`（`ohdec.c:433-449`）只 `OH_VideoDecoder_Stop`，把 `s->dec=NULL` 并 `av_buffer_unref(&s->dec_ref)`；真正 `OH_VideoDecoder_Destroy` 在 `oh_decode_release`（92-101），由最后一个 `dec_ref` 触发。若还有 in-flight AVFrame（`oh_decode_wrap_hw_buffer` 持有的 `buffer->dec_ref`），`Destroy` 会**晚于** `avcodec_close`，甚至晚于 surface 销毁。

**因此**：

- 代码中**没有**“先释放 Surface，再 Destroy 解码器”的显式保证；只有“解码器持有 surface 引用，surface 对象本身由 OH 平台管理”这一层间接保护（本次未真机验证其存活语义，见 §10）。
- 若在 surface 销毁后仍有帧被 `av_ohcodec_release_buffer(render=1)`，会调用 `OH_VideoDecoder_RenderOutputBuffer`（`ohdec.c:516`）对已 detach 的窗口送显；可能返回错误（当前只打日志，`ohdec.c:520-524`），也可能触发平台层未定义行为（[推断]）。

### 7.3 已知/潜在风险清单

| # | 风险 | 证据 | 性质 |
|---|---|---|---|
| R1 | XComponent 销毁回调里 `delete instance/proxy`，其他 NAPI 调用可能已拿到裸指针 | `ijkplayer_napi.cpp:1181-1221`；`getInstance` 返回裸指针（1279-1292） | [观测] 代码事实；[推断] UAF 风险 |
| R2 | 同 id 第二次 `native_setup` 不会替换旧 mp | `set_media_player` 用 `GLOBAL_IJKMP.insert`（`ijkplayer_napi_proxy.cpp:277-292`），key 已存在时 insert 失败；旧 mp/旧 window 保留 | [观测] 代码事实；[推断] 换面后仍绑旧窗口 |
| R3 | window 无 acquire/refcount，display 直接解引用 | `ijksdl_vout_android_nativewindow.c:142-193`；`237-261` | [观测] 代码事实 |
| R4 | `ANativeWindow_release` 被注释，window 引用从不归还 | `ijksdl_vout_android_nativewindow.c:124-128,252-259` | [观测] 代码事实 |
| R5 | 只有 `reset()` 清 surface，`release()` 不清 | `ijkplayer_napi_proxy.cpp:464-475` vs `477-489` | [观测] 代码事实 |
| R6 | Surface 销毁回调是空实现，player 不知道 window 已失效 | `ijkplayer_napi.cpp:1252-1255` | [观测] 代码事实 |
| R7 | `nativeWindowMap_` 无锁 | `ijkplayer_napi.cpp:1097-1114` | [观测] 代码事实 |
| R8 | `GLOBAL_NATIVE_WINDOW` 全局裸指针，多实例互相覆盖 | `ijkplayer_napi_proxy.cpp:311` | [观测] 代码事实 |
| R9 | Decoder Destroy 延迟到最后一个 buffer 释放；可能晚于 surface 销毁 | `ohdec.c:92-101,433-449,530-589` | [观测] 代码事实；[推断] 送显/销毁竞态 |
| R10 | IJK 预加载槽只 `applySurface(sid,false)`，不注入 `nativeContext`；若该槽随后直接 `prepare`，`IjkPlayerBackend.ctx` 可能为 null → `native_setup` no-op | `IjkPreloadCtl.ets:204-230`；`IjkPlayerBackend.ets:209-214`；`SurfaceAuthority.ets:82-84` | [观测] 代码事实；[推断] 预加载转正可能黑屏/无 native 实例 |
| R11 | ArkTS `release()` 与 XComponent `onDestroy` 无显式排序 | `PlayerPage.ets:119-137,282-289`；`PlayerLifecycleCtl.ets:59-87`；`ijkplayer_napi.cpp:1181-1221` | [观测] 代码事实 |
| R12 | `native_setup` 时若 `OnSurfaceCreated` 尚未发生，window 为 null → `output_to_window=false`，且 vout 的 `native_window` 为空 → `display_overlay` 返回 -1（黑屏/待重绑），不是自动可看的 Buffer 模式 | `ijkplayer_napi.cpp:333-339`；`ff_ffplay.c:3482-3484`；`ohdec.c:150-155`；`ijksdl_vout_android_nativewindow.c:144-149` | [观测] 代码事实 |

**未发现的风险（避免误报）**：

- 当前代码**没有**对 `OHNativeWindow` 做显式 release，因此也不存在“重复释放 OHNativeWindow”的代码路径；风险是“从不释放/悬空使用”，不是 double free。
- `IjkPlayerBackend.release()` 有幂等标记（`released`，`IjkPlayerBackend.ets:49-50,621-646`），避免重复 `player.release()` 二次 free。

---

## 8. 对 Sidecar 重构的直接建议（只列边界，不写实现）

1. **先承认 SURFACE 不是“一个 .so”**：Sidecar 的 ABI 面应覆盖 `OHNativeWindow` 所有权 + generation + refcount，而不是试图替换某个库。
2. **把 window 生命周期从 NAPI 回调里拿出来**：`sb_attach/sb_detach/sb_acquire/sb_release` 作为唯一入口；`onSurfaceDestroyedCB` 不得 `delete player`。
3. **保留 IJK 的解码/时钟/seek 主循环**：Sidecar 只接管“窗口与解码器绑定”，不要重写 `ff_ffplay.c` 的 `video_refresh`/`pictq`。
4. **给 `RenderOutputBuffer` 加 generation 闸门**：detach 后只 `FreeOutputBuffer`，不 Render。
5. **把 `AVOHCodecDeviceContext.native_window` 从裸指针改成 Sidecar 句柄**：这是 FFmpeg fork 与 IJK 的私有 ABI，必须一并版本化。
6. **预留 BUFFER 回退**：缩略图 `IjkFrameProvider` 依赖 BUFFER + screenshot（`IjkFrameProvider.ets:113`），不能被 SURFACE Sidecar 一并切断。
7. **构建面独立**：Sidecar 不能依赖 `entry/hvigorfile.ts` 的 `PrepareIjkCodecLib` 覆盖链；否则每次打包都会被官方 `libavcodec.so` 覆盖。

---

## 9. 未定位 / 未验证（按要求标注卡点）

| 项 | 状态 | 卡在哪 |
|---|---|---|
| 名为 `SURFACE 零拷贝.so` 的独立文件 | **未定位** | `D:\devlop` 全盘按文件名过滤 0 命中；Zplayer 无 `entry/src/main/cpp`；只能定位到 §2.2 的 4 个官方 .so |
| `@ohos/ijkplayer@2.0.11` 的精确源码 commit/行号 | **未定位** | 2.0.11 源码未入库；`D:\devlop\ZplayerDemo\native\src\ohos_ijkplayer` 是 2.0.13 master 源码树。二进制日志串与代码一致，但不能证明逐字相同 |
| `OH_NativeWindow_CreateFromSurface` / `CreateNativeWindowFromSurfaceId` 调用 | **确认不存在** | 全仓 + vendor 树 grep 0 命中；window 来自 `OnSurfaceCreated` 回调 |
| `OH_NativeWindow_NativeObjectReference/Unreference` 的可用性 | **未验证** | 代码中完全没有使用；蓝图 §4.2 也标注“待验证符号名与 API 版本”（`SIDECAR-REFACTOR-BLUEPRINT.md:549`） |
| Surface 销毁后 decoder 延迟 Destroy 的实际平台行为 | **未验证** | 需要真机/平台内部实现；本次只读，不能证明“一定崩”或“一定安全” |
| R1/R3 的真实崩溃复现 | **未验证** | 历史文档只给出代码级崩溃链（`SIDECAR-REFACTOR-BLUEPRINT.md:513-527`），没有 tombstone；本次不跑真机 |
| `pictq` 运行时占用长度 | **未暴露** | 只有 `getVideoDecodeFramesPerSecond`/`getVideoOutputFramesPerSecond` 间接指标；无 pictq 长度探针 |
| `RenderOutputBufferAtTime` 的使用 | **确认未使用** | 全仓 grep 0 命中；当前只用 `RenderOutputBuffer`（`ohdec.c:516`） |
| ZplayerDemo 自定义 HAR 是否会被主线采用 | **未定位** | 当前 Zplayer HAP 的哈希与 `v402-libs` 不一致；若未来切换到自定义 HAR，§2.2/§5 的结论需重做 |

---

## 10. 证据索引

### 10.1 代码（Zplayer）

- `entry/src/main/ets/pages/player/VideoSurface.ets:119-155`
- `entry/src/main/ets/pages/player/ZIVPlayerLayer.ets:95-105,357-388`
- `entry/src/main/ets/pages/Index.ets:74-80`
- `entry/src/main/ets/pages/player/VideoStage.ets:97-139`
- `entry/src/main/ets/pages/player/PlayerLayout.ets:87-89`
- `entry/src/main/ets/pages/player/PlayerPage.ets:119-137,282-289,471`
- `entry/src/main/ets/pages/player/SlotSurfaces.ets:14-23`
- `entry/src/main/ets/kernel/player_contract.ets:30-34`
- `entry/src/main/ets/kernel/ports.ets:147-149`
- `entry/src/main/ets/domains/player/SurfaceOpsCtl.ets:82-112`
- `entry/src/main/ets/domains/player/PlayerBindings.ets:62-121,150-202`
- `entry/src/main/ets/domains/player/SurfaceRegistry.ets:23-45`
- `entry/src/main/ets/domains/player/surface/SurfaceAuthority.ets:39-89`
- `entry/src/main/ets/domains/player/PlayerBootstrapper.ets:208-222,279-284`
- `entry/src/main/ets/domains/player/PlayerAssembly.ets:165-172,431-441`
- `entry/src/main/ets/domains/player/PlayerLifecycleCtl.ets:59-139`
- `entry/src/main/ets/domains/player/PlayerBackendPool.ets:68-72`
- `entry/src/main/ets/domains/player/IjkPlayerBackend.ets:105-107,136-139,163-218,366-373,621-646`
- `entry/oh_modules/@ohos/ijkplayer/src/main/ets/ijkplayer/IjkMediaPlayer.ets:67-69,115-120,200-228,682-720`
- `entry/src/main/ets/domains/thumbnail/IjkFrameProvider.ets:113`
- `entry/src/main/ets/domains/player/AvPlayerBackend.ets:150-179`
- `entry/src/main/ets/domains/player/AvVideoUtils.ets:31-43`
- `entry/src/main/ets/kernel/constants.ets:69`
- `entry/hvigorfile.ts:9-53`

### 10.2 代码（vendor 源码树，非 Zplayer 编译）

根目录：`D:\devlop\ZplayerDemo\native\src\ohos_ijkplayer\ijkplayer\src\main\cpp`、`D:\devlop\ZplayerDemo\native\src\ohos_ffmpeg`

- `napi/ijkplayer_napi.cpp:319-341,1097-1114,1140-1156,1181-1221,1238-1255,1279-1292,1304,1338`
- `proxy/ijkplayer_napi_proxy.cpp:17,262-292,303-320,464-489`
- `ijkplayer/ijkplayer_android.c:55-70`
- `ijkplayer/pipeline/ffpipeline_android.c:64-68,92-108,183-197`
- `ijkplayer/ff_ffplay.c:88-90,1195-1245,1637-1706,1984-2023,3441-3504,3545-3553,3590-3594,4764-4790`
- `ijksdl/video/ijksdl_vout_android_nativewindow.c:63-78,88-136,138-196,210-261,314-333`
- `ijksdl/video/ijksdl_vout_overlay_ohos_avcodec.c:95-144,190-197`
- `libavcodec/ohdec.c:92-101,145-201,365-428,433-462,492-528,530-590,592-686,688-724`
- `libavutil/hwcontext_oh.h:27-32`
- `ijkplayer/CMakeLists.txt:11-33`

### 10.3 历史文档

- `DOC/DEVLOG.md:60-64`（v383 P1-P3 + Closure）
- `DOC/DEVLOG.md:79-85`（v393 FlashCatch / FlickerFix / Revert / Closure / v394）
- `DOC/DEVLOG.md:93-95`（v399 DragTakeover / S3 / 回退）
- `DOC/PERFORMANCE-BASELINE.md:14-16,61-68,122-127`
- `DOC/evidence/v399/fullscreen-surface-audit.md:144-223`
- `DOC/evidence/v400/ijk-surface-feasibility.md:67-201`
- `DOC/evidence/v401/p4-audit.md:31-51`
- `DOC/V402/T-08.md:58-78`（SURFACE 保留的 `isUseNativeBuffer 1` 真机证据）
- `D:\devlop\ZplayerDemo\DOC\V402\SIDECAR-REFACTOR-BLUEPRINT.md:511-551`（当前崩溃链 C1-C9）

### 10.4 二进制核对命令

```powershell
$ndk="C:\Program Files\Huawei\DevEco Studio\sdk\default\openharmony\native\llvm\bin"
& "$ndk\llvm-readelf.exe" --dyn-syms entry\oh_modules\@ohos\ijkplayer\libs\arm64-v8a\libijksdl.so |
  Select-String "av_ohcodec_release_buffer"
& "$ndk\llvm-readelf.exe" --dyn-syms entry\oh_modules\@ohos\ijkplayer\libs\arm64-v8a\libavcodec.so |
  Select-String "av_ohcodec_release_buffer|OH_VideoDecoder_SetSurface|OH_VideoDecoder_RenderOutputBuffer"
& "$ndk\llvm-readelf.exe" -d entry\oh_modules\@ohos\ijkplayer\libs\arm64-v8a\libavcodec.so |
  Select-String "NEEDED|SONAME"
& "$ndk\llvm-strings.exe" entry\oh_modules\@ohos\ijkplayer\libs\arm64-v8a\libavcodec.so |
  Select-String "h264_ohcodec|hevc_ohcodec|ohcodec: release output buffer|Set surface failed"
```

已核对结果：

- `libijksdl.so`：`UND av_ohcodec_release_buffer@LIBAVCODEC_62`
- `libavcodec.so`：`av_ohcodec_release_buffer@@LIBAVCODEC_62`；`UND OH_VideoDecoder_SetSurface`；`UND OH_VideoDecoder_RenderOutputBuffer`
- `libavcodec.so` NEEDED：`libnative_media_vdec/venc/codecbase/core`、`libnative_window`
- `libavcodec.so` 含 `h264_ohcodec`、`hevc_ohcodec`、`ohcodec: release output buffer index=%u failed %d`、`Set surface failed, %d, %s`

### 10.5 哈希/版本

- Zplayer HEAD：`edeadea8cb8673a952ce173b44ded6ec9c167b56`（2026-10-27）
- P4 提交：`eb2f2b3`（SURFACE 开关）、`e1c1f44`（vendor 官方 `libavcodec.so`）、`a655858`（.so 走 LFS）
- `entry/oh_modules/.../libavcodec.so` = `tools/.../prebuilt/arm64-v8a/libavcodec.so` = SHA-256 `67876E843D86…`，12,565,832 B
- `libijkplayer.so` = `F1F48C92EE32…`，307,168 B
- `libijksdl.so` = `38978365B63D…`，303,480 B
- `libijkplayer_napi.so` = `62A6EC6A370C…`，130,992 B
- 对照自定义构建：`D:\devlop\ZplayerDemo\v402-libs\arm64-v8a\libavcodec.so` = `061BE8E7347F…`，12,496,056 B（Zplayer HAP 未使用）

---

## 11. 一句话给重构

**SURFACE 零拷贝不是一个可删除的 .so，而是“第三方 IJK fork 的 `ff_ffplay` 接缝 + `libijksdl` 的 HAVC 直出 + FFmpeg `ohcodec` 的 `output_to_window`”三件套；Zplayer 只在最外层用 `VideoCodecMode.SURFACE` 这一个标准开关把它打开。Sidecar 要切的不是库，而是 `OHNativeWindow*` 的所有权与代际——当前它同时裸存于 NAPI map、proxy 全局、pipeline、vout 和 decoder 五处，且没有任何引用计数。**
