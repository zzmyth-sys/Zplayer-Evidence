# SIDECAR-REFACTOR-BLUEPRINT — SMB IO 与 SURFACE 渲染的进程内 Sidecar 解耦蓝图

- 日期：2026-10-27
- 状态：**设计稿，不含任何补丁 / diff / 业务代码改动**
- 前置纪律：`0020-smb2-async-prefetch-ring.patch`（949 行）**冻结**，F-1 ~ F-4 一律停止修补；本文件只描述边界、接口、状态机、演进路径与工作量。
- 输入证据：
  - `DOC/V402/0020-RING-RISK-AUDIT.md`（868 行 `libsmb2.c` 的代码级审计）
  - `native/src/ohos_ffmpeg/libavformat/libsmb2.c`（现行实现）
  - `native/src/ohos_ijkplayer/.../ijkavformat/{ijkio.c, ijkiomanager.c, ijkioprotocol.c}`
  - `native/src/ohos_ijkplayer/.../napi/ijkplayer_napi.cpp`
  - `native/src/ohos_ijkplayer/.../proxy/ijkplayer_napi_proxy.cpp`
  - `native/src/ohos_ijkplayer/.../ijksdl/video/ijksdl_vout_android_nativewindow.c`
- 标记约定：**[观察]** 本轮读到的源码事实；**[推断]** 由事实推出；**[待验证]** 依赖设备/API 版本，设计与估算均按最坏情况留余量。

---

## 0. 结论先行

### 0.1 这套设计是否“彻底阻断 SMB 和 SURFACE 对 IJK 主干的侵入式依赖”？

分两半回答，不模糊：

| 领域 | 判定 | 边界说明 |
|---|---|---|
| **SMB IO** | **是（业务逻辑层彻底阻断）** | 868 行 ring/prefetch/FFmpeg 回调耦合全部迁出 `libavformat`。Sidecar 核心只依赖 `libsmb2` + pthread；对 IJK 的唯一 ABI 面是 `IjkURLProtocol` 的 6 个函数指针，且该适配器位于 IJK 侧 shim（5 行 URL 归一化 + 8 行注册 + ≤120 行头文件适配），不编进 Sidecar 核心库。`libavformat` 的读路径不再含 ring/prefetch/块缓存（可保留一个 ≤150 行、无预取的 `smb_direct` 作为回退；非读路径可收缩为 ≤150 行 `smbmeta`），`ff_ffplay.c` 零改动（主路径）。 |
| **SURFACE 生命周期 / NAPI 回调** | **是（可彻底阻断）** | Bridge 核心独占 `OHNativeWindow` 的引用与 generation；UI 单向驱动 attach/detach；`sb_core` 不保存 `napi_env`/`napi_value`/`OH_NativeXComponent*`，player 的 NAPI 适配器只在实例生命周期内持有 `env`，不再在 XComponent 回调里 `delete player`。野指针与生命周期竞态从“靠运气”变成“结构上不可能”。 |
| **SURFACE 的硬崩溃（SIGSEGV / VDEC 驱动崩溃）** | **否（进程内物理不可能）** | 鸿蒙沙箱禁止独立渲染进程，进程内也无法捕获 SIGSEGV。Bridge 能做到的是：把错误码、超时、VDEC 失败收敛为可重试事件；**不能承诺“解码器段错误绝不拖垮 IJK 进程”**。任何声称能做的方案都是不诚实的，见 §4.6。 |
| **IJK 主干的侵入度** | **SMB：0 行业务逻辑；SURFACE：≤80 行接缝 + 1 个工厂替换** | 解码、时钟、同步、seek 主循环、FFmpeg 打开/关闭流程均不碰。 |

**一句话**：SMB 可以做成真正的“乐高件”；SURFACE 可以做成“生命周期防腐层”，但不能做成“进程内崩溃隔离舱”。

### 0.2 关键事实核查（避免设计画饼）

1. **FFmpeg 上游没有公开的运行时 `URLProtocol` 注册 API**。`URLProtocol` 表由 `CONFIG_*_PROTOCOL` 在编译期生成；`ffurl_register_protocol` 只在本仓库的 `libavformat/url.h`（内部头）和 `async.c:708` 出现，公开的 `libavformat/avio.h` 不导出它。因此“Sidecar 自己 dlopen 后注册一个 `smb` 协议、完全不碰任何宿主”在当前 FFmpeg 上**不成立**。
2. IJK 自带一条官方扩展点 `ijkio:`：`URLProtocol ijkimp_ff_ijkio_protocol`（`ijkio.c:131-139`）→ `ijkio_manager_io_*` → `IjkURLProtocol` inner vtable（`ijkiourl.h:43-53`）。这条路径**不需要改 `libavformat`**。当前构建已启用：`config_components.h:2240 CONFIG_IJKIO_PROTOCOL 1`，`protocol_list.c:21` 注册 `ff_ijkio_protocol`；`ijkmp_set_ijkio_inject_opaque` 在 `ijkplayer_napi_proxy.cpp:318` 调用，`ff_ffplay.c:4908` 创建 manager。
3. 但 `ijkio_alloc_url` 目前是硬编码 `if/else`（`ijkioprotocol.c:32-65`），只认 `cache:/ffio:/httphook:/androidio:`。要挂新后端，**必须在 IJK 的 `ijkavformat` 适配层加一个 ≤20 行的分支（5 行 URL 归一化 + 8 行注册）**。这是 SMB 主路径唯一允许改 IJK 的地方。
4. 备路径是 FFmpeg 公开 API 的 **Custom AVIO**（`avio_alloc_context` + `AVFMT_FLAG_CUSTOM_IO`），只在 `ff_ffplay.c` 的 `avformat_open_input` 前加一个 ≤30 行的 probe 分支。它不依赖 libavformat 内部符号，**libavformat 升级无碍**，但 IJK native 源码升级时需要重放这 30 行。
5. `ijkio` 已把 FFmpeg 的 `AVIOInterruptCB` 传到 manager（`ijkio.c:58`、`ijkiomanager.c:416`），Sidecar 可从 `IjkURLContext->ijkio_app_ctx->ijkio_interrupt_callback` 取得中断回调，**不需要 FFmpeg 私有头**。
6. SURFACE 侧当前确有代码级崩溃链，不是“感觉不稳”，见 §4.1 的逐行证据。

---

## 1. 现状与目标

### 1.1 现状职责堆叠

| 层 | 现状文件 | 实际承担 | 后果 |
|---|---|---|---|
| SMB 传输 | `libsmb2.c:136-162` | `smb2_pread` + 1 MB 切片 + `net_lock` | 与 ring 绑定，无法单测 |
| 预取环 | `libsmb2.c:177-254` | 4 MB 块、gen 校验、memmove、eof/err | 4 MB 不可分割，F-1；无中断，F-2 |
| 块缓存 | `libsmb2.c:259-318` | LRU、锁外写、锁内发布 | H-3a 脏数据雷 |
| FFmpeg 语义 | `libsmb2.c:561-690` | `pos`、`eof`、`AVERROR`、等待 25 s | 协议层与 demuxer 语义缠绕 |
| 状态 | 散落 7 个 int（`inited/thread_started/quitting/prefetch_started/eof/err/err_logged`） | 无统一状态机 | 一次错误终身失败（S-1/F-4） |
| SURFACE | `ijkplayer_napi.cpp:1097-1265` | XComponent 回调、全局 map、直接 delete | 裸指针 + 回调线程 delete |
| SURFACE 数据面 | `ijksdl_vout_android_nativewindow.c:124-267` | 裸 `native_window` 指针、release 被注释 | window 销毁后继续 display |

### 1.2 设计目标（可测，不写口号）

1. **SMB 读路径业务逻辑零进入 `libavformat`**：ring/prefetch/块缓存/中断调度全部迁出；`libavformat` 只保留（或裁掉）一个不含预取的 `smbmeta` 小实现来处理目录/删除/移动等非读路径，`seek.c` 的 `smb` 字符串特判可删除。
2. **Sidecar 可主机单测**：不依赖设备、不依赖 FFmpeg、不依赖真实网络；用 fake transport + 虚拟时钟跑完子块发布、中断、50 次 seek 风暴。
3. **SURFACE 无跨线程裸指针**：任何 `OHNativeWindow*` 必须带 generation + refcount 才能被渲染线程使用。
4. **升级可退**：Sidecar 缺失/ABI 不匹配时，播放器自动回退官方路径，不崩溃、不黑屏死锁。
5. **性能不退化**：LAN 下 seek 首字节 p95 < 300 ms；50 次跨距风暴 0 次 `ETIMEDOUT`；release p95 < 1 s。

### 1.3 明确不做的事

- 不重写 demuxer、不改解码时序、不改 IJK 的 clock/serial/seek 主循环。
- 不做独立进程、不做沙箱外 `dlopen`、不做 `LD_PRELOAD` 符号拦截。
- 不承诺捕获 SIGSEGV / 驱动级 VDEC 崩溃。
- **本期 Sidecar 只覆盖读路径** `open/read/seek/close/interrupt`；SMB 目录浏览、删除、移动、写（`open_dir/read_dir/delete/move/write`）不在范围。主项目若需要扫片单，保留一个 ≤150 行、无预取的 `smbmeta` 协议；不要把这些语义塞进 IOManager。
- 不在本阶段改任何 C 代码；本文所有接口均为设计草案。

---

## 2. 总体架构

### 2.1 层级图

```
┌──────────────────────────────────────────────────────────────────────┐
│ ArkTS / HAP（沙箱内）                                                 │
│   IjkMediaPlayer.setDataSource("smb://…")                            │
│   XComponent(type=SURFACE, libraryname="ijkplayer_napi")              │
│   aboutToAppear / aboutToDisappear  ── 单向驱动 ──┐                    │
└───────────────────────────┬──────────────────────┼───────────────────┘
                            │ NAPI                 │ XComponent callbacks
┌───────────────────────────▼──────────────────────▼───────────────────┐
│ libijkplayer_napi.so + libijkplayer.so（同进程，随 HAP 打包）         │
│  ┌────────────────────┐        ┌──────────────────────────────────┐  │
│  │ SMB shim（在 IJK 的  │        │ SurfaceBridge 的 NAPI 事件入口    │  │
│  │ ijkavformat，≤20 行） │        │ attach/detach → sb_*（不存 env）  │  │
│  │ ijkio inner 注册    │        │                                   │  │
│  └─────────┬──────────┘        └──────────────┬───────────────────┘  │
│            │ IjkURLProtocol vtable             │ C ABI                 │
│  ┌─────────▼──────────┐        ┌──────────────▼───────────────────┐  │
│  │ libzsmb_io.so      │        │ libsurface_bridge.so             │  │
│  │ Transport          │        │ window ref + generation          │  │
│  │ IOManager (Ring,   │        │ state machine + VDEC retry       │  │
│  │  BlockCache, State)│        │ 不持有 napi_env / XComponent 指针 │  │
│  │ 零 FFmpeg 依赖      │        │ 零 IJK 依赖                       │  │
│  └─────────┬──────────┘        └──────────────┬───────────────────┘  │
│            │ libsmb2                          │ OH_NativeWindow      │
│  ┌─────────▼──────────┐        ┌──────────────▼───────────────────┐  │
│  │ libsmb2.so.1       │        │ libijksdl (vout 接缝，≤80 行)     │  │
│  └─────────┬──────────┘        └──────────────┬───────────────────┘  │
└────────────┼──────────────────────────────────┼──────────────────────┘
             │ SMB/TCP                          │ EGL / OHNativeWindow
        ┌────▼────┐                       ┌─────▼──────┐
        │ SMB 服务器│                       │ XComponent │
        └─────────┘                       └────────────┘
```

要点：

- **两个 Sidecar 互不依赖**，只共享“进程内 + 随 HAP 打包”的部署约束。
- 两者都由 `libijkplayer_napi.so` 的宿主链接并随 HAP 一起加载，属于**进程内 NAPI 模块的一部分**，不是独立进程、不是沙箱外插件。SMB 只是不走 NAPI 的数据面。
- SMB 走 `ijkio` 的 inner vtable，**不经过 NAPI**，数据路径全在 native，无 ArkTS 拷贝。
- SURFACE Bridge 直接面对 NAPI/XComponent，但**只接受事件，不持有 UI 对象**。
- 两个库的符号默认 `hidden`，唯一导出面见 §3.3 / §4.3；`libijkplayer.so` 通过 `DT_NEEDED` 链接，系统加载器在应用沙箱内完成加载（满足“禁止沙箱外动态加载”）。

### 2.2 目录与构建（建议结构）

```
native/src/sidecar/smb/                # → libzsmb_io.so（即 libsmb2_sidecar.so；核心零 IJK / 零 FFmpeg）
  include/zsmb.h                       # 稳定 C ABI（opaque）
  src/zsmb_transport.c                 # libsmb2 封装，唯一 include <smb2/...>
  src/zsmb_iom.c                       # Ring + BlockCache + 状态机 + worker
  src/zsmb_state.c                     # 迁移表与不变量（可被 iom 内联）

native/src/sidecar/surface/            # → libsurface_bridge.so（零 IJK / 零 NAPI）
  include/sb.h                         # 稳定 C ABI（opaque）
  src/sb_core.c                        # slot/ref/gen/状态机
  src/sb_vout.c                        # SDL_Vout 接缝

native/src/ohos_ijkplayer/.../ijkavformat/   # IJK 侧 shim（不属于 Sidecar 核心）
  zsmb_ijkio.c                         # IjkURLProtocol 适配器（include ijkiourl.h + zsmb.h）
  zsmb_avio.c                          # Custom AVIO 适配器（备路径）

native/src/ohos_ijkplayer/.../napi/
  sb_napi_bridge.cpp                   # XComponent 事件 → sb_*（唯一 include napi 的地方）

native/src/sidecar/tests/
  fake/zsmb_fake_transport.c
  fake/fake_window.c
  test_publish.c test_interrupt.c test_seek_storm.c test_surface_ref.c ...
```

构建约束：

- `libzsmb_io.so`：链接 `smb2` + `pthread` + `libc`，**不链接 `av*`，不 include IJK 头**。`zsmb_ijkio.c` 编进 IJK 的 shim，不编进本库。
- `libsurface_bridge.so`：`sb_core.c/sb_vout.c` 不 include NAPI；只有 `sb_napi_bridge.cpp` 在 IJK 侧包含 `napi.h`。
- `libijkplayer.so`：新增 `target_link_libraries(ijkplayer zsmb_io surface_bridge)`；HAP 的 `libs/arm64-v8a/` 随包带上两个 `.so`。
- `libavformat`：**移除读路径的 `libsmb2.o`**（`Makefile:737` 的 `OBJS-$(CONFIG_LIBSMB2_PROTOCOL)` 不再指向魔改文件）；用 `smb_direct.c`（≤150 行、无 ring/prefetch/块缓存，编译开关 `SIDECAR_SMB_FALLBACK=1`）作为 Sidecar 缺失时的回退；若主项目仍需 SMB 目录/删除/移动，另建 `smbmeta.c`（≤150 行、无预取），不要把非读路径重新塞回 Sidecar 核心；`seek.c:192-194` 的 `smb` 特判删除。

---

## 3. SMB IO Sidecar

### 3.1 组件边界

| 组件 | 只做什么 | 绝不做什么 |
|---|---|---|
| `Transport` | 建会话、认证、`pread`（≤1 MB/请求）、`stat`、`cancel`、`close` | 不认识 ring/cache/FFmpeg/seek 语义 |
| `IOManager` | Ring、BlockCache、状态机、预取 worker、子块发布、错误预算 | 不 include `libav*`；不返回 `AVERROR`；不碰 NAPI |
| `ijkio Adapter`（IJK 侧 shim） | 把 `IjkURLProtocol` 的 4 个回调转成 `zsmb_*`；映射错误码 | 不含调度逻辑；不持锁跨回调；不编进 `libzsmb_io.so` |
| `avio Adapter`（备，IJK 侧 shim） | 把 `AVIOContext` 的 read/seek 转成 `zsmb_*`；桥接 `interrupt_callback` | 同上 |

### 3.2 挂载方案：主路径 + 备路径

#### 主路径 A：IJK `ijkio` inner protocol（推荐）

调用链（**[观察]** 自现有代码）：

```
FFmpeg URLProtocol "ijkio"            (ijkio.c:131-139)
  → ijkio_manager_io_open/read/seek   (ijkiomanager.c:380/448/470)
    → ijkio_alloc_url(url)            (ijkioprotocol.c:32-65)   ← 唯一需要加分支的地方
      → IjkURLProtocol "zsmb"         (Sidecar 导出)
        → zsmb_open/read/seek/close
```

URL 形式：`ijkio:zsmb://user:pass@host/share/movie.mp4`

URL 归一化（≤5 行，放在 `IjkMediaPlayer_setDataSource` / `ijkplayer_napi_proxy.cpp:338`）：

```c
if (!strncmp(url, "smb://", 6)) {
    snprintf(buf, sizeof(buf), "ijkio:zsmb://%s", url + 6);
    url = buf;
}
```

这样 ArkTS 仍然只传 `smb://…`，不需要知道 `ijkio` 的存在；代价是第三处、也是最小的 shim（5 行）。

`zsmb_open` 收到 `zsmb://…` 后，只在 Transport 里把前缀替换为 libsmb2 认识的 `smb://…`（不重新解析凭据）；这一处归一化在 Sidecar 内部完成。

IJK 侧改动（示意，**本次不实施**）：

```c
/* ijkioprotocol.c，仅 +8 行；另有 5 行 URL 归一化见下文 */
else if (!strncmp(url, "zsmb:", strlen("zsmb:"))) {
    h = (IjkURLContext *)calloc(1, sizeof(IjkURLContext));
    h->prot = &zsmb_ijkio_protocol;                 /* Sidecar 导出 */
    h->priv_data = calloc(1, h->prot->priv_data_size);
}
```

Sidecar 导出（**接口定义，不是补丁**）：

```c
/* zsmb_ijkio.h —— 唯一依赖 IJK 的头，单独编译单元 */
#include "ijkavformat/ijkiourl.h"

extern IjkURLProtocol zsmb_ijkio_protocol;   /* name="zsmb" */
```

`IjkURLProtocol` 的字段在 `ijkiourl.h:43-53`，IJK 侧 shim 只填：

```c
static int     zsmb_i_url_open2(IjkURLContext *h, const char *url, int flags, AVDictionary **opts);
static int     zsmb_i_url_read (IjkURLContext *h, unsigned char *buf, int size);
static int64_t zsmb_i_url_seek (IjkURLContext *h, int64_t pos, int whence);
static int     zsmb_i_url_close(IjkURLContext *h);

IjkURLProtocol zsmb_ijkio_protocol = {
    .name = "zsmb",
    .url_open2 = zsmb_i_url_open2,
    .url_read  = zsmb_i_url_read,
    .url_seek  = zsmb_i_url_seek,
    .url_close = zsmb_i_url_close,
    .priv_data_size = sizeof(ZsmbAdapter),
};
```

中断回调从 IJK 已有的传递链取得（**[观察]**），通过 `zsmb_open` 的 `abort_cb` 参数注入，不新增接口：

```c
/* zsmb_i_url_open2 内 */
static int on_abort(void *user)
{
    AVIOInterruptCB *cb = user;
    return cb && cb->callback ? cb->callback(cb->opaque) : 0;
}

AVIOInterruptCB *cb = h->ijkio_app_ctx
                    ? h->ijkio_app_ctx->ijkio_interrupt_callback
                    : NULL;
a->io = zsmb_open(smb_url, &a->cfg, on_abort, cb,
                  a->event_cb, a, err, sizeof(err));
```

`zsmb_i_url_close` 先 `zsmb_interrupt(a->io)` 再 `zsmb_close(a->io)`，保证 `url_close` 不会无限等。

`zsmb_i_url_read` 的返回值直接是“读到的字节数或负值”，不需要 `AVERROR`；`zsmb_i_url_seek` 只处理 `AVSEEK_SIZE` 与三种 whence，其余返回 `ZSMB_E_INVAL`。

**优点**：`libavformat` 零改动；不依赖私有符号；升级 `libavformat.so` 无影响。
**代价**：IJK native 升级需重放 5 行 URL 归一化 + 8 行分支；`ijkio` 层一次函数跳转（**[推断]** <1% 开销）。

#### 备路径 B：Custom AVIO（不推荐作默认）

在 `ff_ffplay.c:3867` 之前加：

```c
/* 设计示意，不实施 */
if (zsmb_url_probe(is->filename)) {
    AVIOContext *pb = NULL;
    int ret = zsmb_avio_create(&pb, is->filename, ffp->interrupt_cb);
    if (ret < 0) goto fail;
    ic = avformat_alloc_context();
    ic->pb = pb;
    ic->flags |= AVFMT_FLAG_CUSTOM_IO;
    if ((ret = avformat_open_input(&ic, "", is->iformat, &ffp->format_opts)) < 0)
        goto fail;
}
```

坑：

- `avformat_open_input` 在 `CUSTOM_IO` 下不会自己建 `AVIOContext`，但会尝试 probe；`ic->pb` 必须在调用前设置。
- 关闭顺序必须是 `avformat_close_input` → `avio_context_free`，且 `AVIOContext->opaque` 的释放只能由 Sidecar 的 close 回调完成。
- 需要处理 `avio_size`/`seekable`/`read_packet` 的短读语义。
- 这是 FFmpeg 公开 API，**libavformat 升级 ABI 稳定**，但 `ff_ffplay.c` 的 30 行需要跟随 IJK native 升级重放。

**结论**：默认走 A；B 作为“ijkio 被上游移除”时的逃生门，两者共用同一个 `zsmb_*` 核心，切换成本只在一个适配器文件。

### 3.3 公共 C 接口（`zsmb.h`）

```c
/* zsmb.h — 稳定 ABI v1；核心零 FFmpeg 依赖 */
#ifndef ZSMB_H
#define ZSMB_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

#define ZSMB_ABI_VERSION 1u

typedef struct zsmb zsmb_t;          /* opaque，禁止 sizeof */

typedef enum {
    ZSMB_OK          =  0,
    ZSMB_E_ABORT     = -1,           /* 被 interrupt 取消 */
    ZSMB_E_TIMEOUT   = -2,
    ZSMB_E_IO        = -3,
    ZSMB_E_AUTH      = -4,
    ZSMB_E_NOTFOUND  = -5,
    ZSMB_E_NOMEM     = -6,
    ZSMB_E_STATE     = -7,
    ZSMB_E_INVAL     = -8,
    ZSMB_E_FATAL     = -9            /* 需上层换实例/换源 */
} zsmb_status;

typedef enum {
    ZSMB_EV_BUFFERING = 1,           /* 进入/退出缓冲，arg=percent */
    ZSMB_EV_SEEK_SETTLED,            /* arg=ticket(gen)，arg2=pos */
    ZSMB_EV_ERR_TRANSIENT,           /* 已自动重试，上层仅记录 */
    ZSMB_EV_ERR_FATAL,               /* arg=zsmb_status */
    ZSMB_EV_CLOSED
} zsmb_event;

typedef struct zsmb_config {
    uint32_t abi_version;            /* 必须 == ZSMB_ABI_VERSION */
    uint32_t ring_cap;               /* 0=默认 8 MiB */
    uint32_t unit_normal;            /* 0=默认 1 MiB */
    uint32_t unit_urgent;            /* 0=默认 256 KiB */
    uint32_t history;                /* 0=默认 2 MiB */
    uint32_t block_size;             /* 0=默认 256 KiB */
    uint32_t block_count;            /* 0=默认 8 */
    uint32_t io_timeout_ms;          /* 0=默认 2000，clamp [1000,5000] */
    uint32_t close_timeout_ms;       /* 0=默认 2500 */
    uint32_t err_budget;             /* 0=默认 3 */
} zsmb_config;

typedef int  (*zsmb_abort_cb)(void *user);              /* 非 0 = 取消 */
typedef void (*zsmb_event_cb)(void *user, int ev,
                              int64_t arg, int64_t arg2);

int      zsmb_abi_version(void);

/* 打开；失败返回 NULL，err 写入可读原因。不阻塞超过 io_timeout。 */
zsmb_t  *zsmb_open(const char *url,
                   const zsmb_config *cfg,
                   zsmb_abort_cb abort_cb, void *abort_user,
                   zsmb_event_cb event_cb, void *event_user,
                   char *err, size_t err_size);

/* 读；*got 为实际字节数。返回 zsmb_status。 */
int      zsmb_read(zsmb_t *io, void *buf, uint32_t size, uint32_t *got);

/* 立即受理的 seek；返回 ticket(>=1)，ticket 单调递增。不等待 I/O。 */
int64_t  zsmb_seek(zsmb_t *io, int64_t pos);

/* 文件大小；未知返回 -1。 */
int64_t  zsmb_size(zsmb_t *io);

/* 幂等；任意线程可调；只置取消并唤醒等待者，不做 join。 */
void     zsmb_interrupt(zsmb_t *io);

/* 有界关闭；返回 0 或 ZSMB_E_TIMEOUT（内部已 detach 回收）。 */
int      zsmb_close(zsmb_t *io);

#ifdef __cplusplus
}
#endif
#endif
```

设计要点：

- **不把 `AVERROR`、`AVIOInterruptCB`、`URLContext` 暴露给核心**；适配器负责双向映射。
- `zsmb_seek` 返回 **ticket**，这是给主项目 epoch/ack 归属用的稳定票据（不暴露 FFmpeg serial）。事件 `ZSMB_EV_SEEK_SETTLED` 携带同一 ticket。
- `zsmb_close` 有界：超时后内部转入“自回收”，句柄立即失效；详见 §3.6。
- `zsmb_config` 只增字段、不改语义；`abi_version` 不匹配直接失败。

### 3.4 内部接口（Transport / IOManager）

```c
/* 仅 Sidecar 内部可见 */
typedef struct zt_cancel {
    volatile int  *flag;             /* 原子置位 */
    int64_t        deadline_ms;      /* 绝对时间，单调时钟 */
} zt_cancel;

typedef struct zt_ops {
    int  (*open)  (void **self, const char *url, const zsmb_config *,
                   char *err, size_t err_size);
    int  (*stat)  (void *self, uint64_t *size);
    /* 单次最多 1 MiB；got=0 且 ok 表示 EOF；必须可被 cancel 打断在请求边界 */
    int  (*pread) (void *self, uint64_t off, void *buf, uint32_t want,
                   uint32_t *got, const zt_cancel *cancel);
    void (*cancel)(void *self);      /* 只置标志，不阻塞 */
    void (*close) (void *self);
} zt_ops;

/* IOManager 只吃 ops，不认识 libsmb2；单测用 fake_ops 注入 */
int  iom_create (const zsmb_config *cfg, const zt_ops *ops, void *self,
                 zsmb_event_cb ev, void *ev_user, zsmb_t **out);
int  iom_read   (zsmb_t *io, void *buf, uint32_t size, uint32_t *got);
int64_t iom_seek(zsmb_t *io, int64_t pos);
void iom_interrupt(zsmb_t *io);
int  iom_close  (zsmb_t *io);
```

`Transport` 的 `pread` 内部就是现在的 `smb2_pread_full`，但做两件事：

1. 每 1 MB 请求前检查 `cancel->flag` 与 `deadline_ms`；命中返回 `ZSMB_E_ABORT`/`ZSMB_E_TIMEOUT`。
2. 只持有 transport 自己的 `net_lock`；**绝不持 IOManager 的 `M`**。

### 3.5 状态机

用户给定的 `Idle / Connecting / Buffering / Seeking / Error / Closed` 保留，**增加 `Closing`**：`join` 必须有可观测的过渡态，否则 `Closed` 无法同时表达“句柄已失效”和“worker 仍在退出”。

```
                 zsmb_open()
     [Idle] ───────────────────► [Connecting]
        ▲                            │ ok
        │ open fail(fatal)           ▼
        │                      [Buffering] ◄──────────────┐
        │                        │    ▲                   │
        │              seek()    │    │ urgent subblock   │
        │                        ▼    │ published         │
        │                    [Seeking]┘                   │
        │                        │                         │
        │        transient err   ▼                         │
        │                    [Error] ── recover (budget) ─┘
        │                        │ budget exhausted
        │                        ▼
        └──── close() ────── [Closing] ── worker joined ──► [Closed]
```

转移表：

| From | 事件 | To | 动作（全部在 `M` 内做迁移，I/O 在锁外） |
|---|---|---|---|
| Idle | `open` | Connecting | 分配、初始化、建 worker |
| Connecting | 认证/打开成功 | Buffering | `prefetch_started=1`，发首个 urgent 子块 |
| Connecting | 不可恢复错误 | Closing → Closed | 释放 transport，回调 `ERR_FATAL` |
| Buffering | `seek(p)` | Seeking | `pos=p`；`gen++`；ring 清空；`urgent=1`；broadcast |
| Seeking | 首个 urgent 子块发布 | Buffering | `SEEK_SETTLED(ticket, p)` |
| Seeking | 又收到 `seek` | Seeking | 合并：只保留最新 `pos`，`gen++`，首块继续 256 KiB |
| Buffering/Seeking | transport 瞬时错误 | Error | 退避 100/300/900 ms，`consecutive++` |
| Error | 任一读/预取成功 | Buffering | `consecutive=0`；`ERR_TRANSIENT` 已发 |
| Error | `consecutive >= err_budget` | Closing | `ERR_FATAL`，不再原地重试 |
| 任意 | `interrupt()` | 不变（置 aborted） | 读/预取在检查点返回 `ZSMB_E_ABORT`；状态由调用方决定是否 close |
| 任意 | `close()` | Closing | 置取消、broadcast、有界等 worker |
| Closing | worker 已退出 / 超时 detach | Closed | 释放 ring/cache；`EV_CLOSED`；句柄失效 |

`Error` 不是“粘滞”的：`seek` 与任何一次成功的 `pread` 都会清零 `consecutive`；这与 F-4 的“一次错误终身失败”正好相反。**自愈路径**有两条，必须在设计里写死：

1. 预取线程退避后重试；成功后自动回 `Buffering`。
2. 若消费者正在等待且 `ring_len == 0`，`iom_read` 允许一次 **bypass 直读**（≤`unit_normal`，直接走 Transport，不经 ring）；成功则清零预算、广播并回 `Buffering`，失败则计数，达到预算才转 `FATAL`。这样“预取线程被一次瞬时错误卡住”不会把整场播放判死。

错误**分级**而不是一个 `int err`：

- `TRANSIENT`：超时、短读无进展、`STATUS_IO_TIMEOUT` → 可重试。
- `FATAL`：认证失败、文件不存在、会话不可重建 → 交给上层换实例。
- `ABORT`：用户/FFmpeg 取消 → 不是错误，返回 `AVERROR_EXIT`。

### 3.6 生命周期与 seek/release 时序

#### seek（不阻塞、不 join）

```
FFmpeg read_thread          IOManager(M)               PrefetchWorker        Transport
      │ zsmb_seek(p)             │                          │                    │
      │─────────────────────────►│ lock: pos=p, gen++,      │                    │
      │                          │ ring.reset(urgent=1)     │                    │
      │                          │ broadcast                │                    │
      │◄──── ticket ─────────────│ unlock                   │                    │
      │                          │                          │ (在途 1 MiB 完成)   │
      │                          │                          │◄───────────────────│
      │                          │                          │ gen 变了 → 丢弃     │
      │                          │                          │ 下一子块 256 KiB    │
      │                          │                          │───────────────────►│
      │ zsmb_read(32 KiB)        │                          │                    │
      │─────────────────────────►│ wait(cv_data)            │                    │
      │                          │◄── publish 256 KiB ──────│                    │
      │◄──── 32 KiB ─────────────│                          │                    │
```

关键点：

- `seek` 只改账本，**不等待任何 I/O**；因此 120 ms 合并窗的上层可以连续发 seek，不会阻塞命令队列。
- 子块粒度 = `unit_urgent(256 KiB)` / `unit_normal(1 MiB)`；在途块最大 1 MiB，作废成本从 4 MiB 降到 ≤1 MiB。
- 预取在**每个子块边界**复查 `gen`，只丢未发布部分（F-1）。

#### close（有界、可 detach）

```
zsmb_close()
  1. lock(M): state=Closing; abort=1; broadcast; unlock
  2. transport.cancel()                 // 只置标志 + 唤醒，不 join
  3. wait(worker_done, close_timeout)   // cv_timedwait，不持 M 等 join
  4a. 已退出: join; transport.close(); free
  4b. 超时:   worker 持 core 引用，退出时自回收；句柄立即失效
  5. state=Closed; EV_CLOSED（锁外回调）
```

规则：

- `zsmb_close` 的调用方（FFmpeg）之后不得再用句柄；超时 detach 时，core 由 worker 最后 `refcount--` 释放。
- `refcount` 是原子：handle 1 个 + worker 1 个 + 在途 read 1 个（如有）。**没有裸指针跨线程**。
- `transport.close()` 只由“最后一个持有者”执行；worker 卡在 `pread` 时，close 不抢 `net_lock`，避免把调用方也拖住。

### 3.7 锁纪律（摘要）

| 资源 | 保护 | 访问者 |
|---|---|---|
| `pos`、`ring_base/len/tail`、`consume`、`gen` | 单一 `M` | read / seek / worker（全部短临界区） |
| `state`、`abort`、`refcount` | 原子（跨线程取消/迁移） | 任意线程 |
| `blocks[i].state/off/len/stamp` | `M` | read / 任意块填充者 |
| slot 数据缓冲 | `FILLING` 状态屏障 | 填充者在锁外写，发布在锁内 |
| SMB 会话 | `Transport.net_lock` | 仅 worker / 块填充 / close |

三条硬规则：

1. **`M` 永不跨 I/O、永不跨 join、永不跨回调**。因此任何线程都能在微秒级拿到 `M`，不存在“seek 等不到锁”。
2. **锁序只有 `M → Transport` 不存在**；需要 I/O 时先释放 `M`。`net_lock` 永不反向获取 `M`。
3. ring 用**真环**（`tail/len` + 两段 memcpy），不再在持锁时 `memmove` 12 MiB（S-8）；块缓存用 `FREE/FILLING/READY` 三态，彻底消除 H-3a 的“锁外写、锁内发布”。

> 本章即锁纪律的最小充分集；更细的子块发布伪代码与 12 条并发不变量可在进入 W2 时单独立项，不阻塞本蓝图。

### 3.8 内存与性能预算

| 项 | 默认 | 上限 | 说明 |
|---|---|---|---|
| ring | 8 MiB | 16 MiB | 16 MiB 是配置上限，不是硬编码；DEMO 可用 16 MiB 与现状对齐，主项目默认 8 MiB 降 PSS；真环无 memmove |
| block cache | 8 × 256 KiB = 2 MiB | 16 × 256 KiB = 4 MiB | 片源相关，按 hit rate 调 |
| publisher/scratch | 1 MiB | 1 MiB | 单子块 |
| transport | ~0 | — | libsmb2 内部 |
| **合计** | **~11 MiB** | **~21 MiB** | 现状 22 MiB；默认降 11 MiB |
| 每实例线程 | 1 worker | 1 | 不随 seek 增长 |

---

## 4. SURFACE Bridge

### 4.1 现状崩溃链（逐行证据）

| # | 位置 | 现象 | 后果 |
|---|---|---|---|
| C1 | `ijkplayer_napi.cpp:1097-1113` | `nativeXComponentMap_` / `nativeWindowMap_` 直接读写，无 `mapMutex_` | 与 NAPI 线程并发 → map 结构破坏 |
| C2 | `ijkplayer_napi.cpp:1181-1221` | `OnSurfaceDestroyed` 在 XComponent 回调线程直接 `delete instance` / `delete proxy` | 其它线程持有的 player/vout 立即悬空 |
| C3 | `ijkplayer_napi.cpp:1252-1255` | `onSurfaceDestroyed` 空实现 | player 不知道 window 已失效，继续 display |
| C4 | `ijkplayer_napi_proxy.cpp:311` | `GLOBAL_NATIVE_WINDOW` 全局裸指针 | 多实例/重建时互相覆盖 |
| C5 | `ijkplayer_napi_proxy.cpp:303-319` | window 只在 `native_setup` 绑定一次 | surface 重建后仍是旧指针；无重新 attach 通路 |
| C6 | `ijksdl_vout_android_nativewindow.c:124-127` | `ANativeWindow_release` / `NativeLayerHandle` 被注释 | window 引用从不归还，且销毁后指针仍被使用 |
| C7 | `ijksdl_vout_android_nativewindow.c:142-193` | `display_overlay` 直接解引用 `opaque->native_window` | detach 与 display 并发 → 野指针 |
| C8 | `ijkplayer_android.c:55-68` | `SDL_VoutAndroid_SetAndroidSurface` 传裸指针 | 没有 acquire/refcount，生命周期无主 |
| C9 | `ijkplayer_napi_proxy.cpp:484` | 只有 `reset()` 才把 surface 置空 | 正常 `release()`/页面退出路径未统一 |

`[观察]` 以上均为当前源码事实；`[推断]` C2+C7 是“高频切换/销毁时崩溃”的最短路径：UI 回调删实例 → vout 线程下一次 `display_overlay` 解引用已释放 window。

### 4.2 设计原则：单向驱动 + 引用快照

```
ArkTS XComponent 事件（创建/变化/销毁）
        │  只能调用
        ▼
sb_attach / sb_detach / sb_destroy      ← 唯一入口，UI 线程
        │  只改 Bridge 内部状态，绝不 delete player
        ▼
SurfaceBridge（单锁 + 原子 refcount + generation）
        │  sb_acquire / sb_release
        ▼
渲染线程（vout / display）               ← 只拿“带 ref 的快照”，不拿裸指针
```

反向约束（写进代码评审清单）：

1. Bridge 核心 **不持有** `napi_env`、`napi_value`、`napi_ref`、`OH_NativeXComponent*`；只有 IJK 侧的 `sb_napi_bridge.cpp` 可以在 player 实例生命周期内持有 `napi_env`，且不得跨 `release` 存活。
2. Bridge **不调用** `OH_NativeXComponent_*` 回写 UI，不调用 `delete player`。
3. Bridge 的事件只能通过注入的 `sb_post_event` 函数指针单向发出；需要到 ArkTS 时由 NAPI 层转 `napi_threadsafe_function`。
4. 所有 `OHNativeWindow*` 必须在 `attach` 时 `OH_NativeWindow_NativeObjectReference`，在最后一个 ref 释放时 `Unreference`。**[待验证]** 目标 API 版本的两个符号名与可用性，需在第一步用最小 demo 验证。
5. `detach` 只做“摘链 + gen++”，不等待渲染线程；window 的最终 `Unreference` 由 refcount 归零触发。

### 4.3 公共 C 接口（`sb.h`）

```c
/* sb.h — SurfaceBridge 稳定 ABI v1；零 NAPI / 零 IJK 依赖 */
#ifndef SB_H
#define SB_H

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

#define SB_ABI_VERSION 1u

typedef struct sb_ctx sb_ctx;

typedef enum {
    SB_OK          =  0,
    SB_E_STATE     = -1,
    SB_E_NO_WINDOW = -2,
    SB_E_NOMEM     = -3,
    SB_E_TIMEOUT   = -4,
    SB_E_VDEC      = -5,
    SB_E_VDEC_FATAL= -6
} sb_status;

typedef enum {
    SB_EV_ATTACHED = 1,
    SB_EV_DETACHED,
    SB_EV_LOST,            /* window 失效，等待重新 attach */
    SB_EV_VDEC_RETRY,      /* arg=第几次重试 */
    SB_EV_VDEC_FATAL,      /* 上层应切软解/换实例 */
    SB_EV_DESTROYED
} sb_event;

typedef struct sb_config {
    uint32_t abi_version;
    uint32_t vdec_retry_max;      /* 0=2 */
    uint32_t vdec_retry_backoff_ms;/* 0=300 */
    uint32_t frame_stall_ms;      /* 0=500，超时视为 VDEC stall */
} sb_config;

/* 渲染线程取到的窗口快照；必须成对 acquire/release */
typedef struct sb_frame {
    void    *native_window;       /* OHNativeWindow*，已持有引用 */
    uint64_t generation;          /* 与当前 attach 代际比对 */
    int32_t  width;
    int32_t  height;
} sb_frame;

typedef void (*sb_post_event)(void *user, int ev, int64_t arg);

int  sb_abi_version(void);
int  sb_create(const sb_config *cfg, sb_post_event post, void *user, sb_ctx **out);

/* UI 线程：XComponent onSurfaceCreated / onSurfaceChanged。
 * xcomponent_id 可为 NULL：表示该 sb_ctx 只服务一个 player/surface；
 * 多实例场景传 id，用于校验事件归属。 */
int  sb_attach(sb_ctx *sb, const char *xcomponent_id, void *native_window,
               int32_t width, int32_t height);

/* UI 线程：XComponent onSurfaceDestroyed；非阻塞，不等待渲染线程 */
int  sb_detach(sb_ctx *sb, const char *xcomponent_id);

/* 渲染线程：取窗口快照；无窗口返回 SB_E_NO_WINDOW */
int  sb_acquire(sb_ctx *sb, sb_frame *out);
void sb_release(sb_ctx *sb, sb_frame *frame);

/* 解码/渲染结果回报；负值只做计数/退避/发事件，
 * 真正的 codec 重建由 IJK 的 vdec 接缝执行（Bridge 不拥有 codec）。 */
int  sb_on_vdec_result(sb_ctx *sb, int32_t code);

/* 仅 UI/播放器 release 驱动；幂等，有界；返回后指针失效 */
int  sb_destroy(sb_ctx *sb);

#ifdef __cplusplus
}
#endif
#endif
```

### 4.4 内部结构与竞态消解

```c
typedef struct {
    void     *win;        /* OHNativeWindow*，已 Reference */
    uint64_t  gen;
    int32_t   w, h;
    int32_t   refs;       /* 原子；acquire++ / release-- */
    int       retired;    /* detach 后等待 refs==0 */
} sb_slot;

struct sb_ctx {
    pthread_mutex_t m;    /* 唯一锁：slot/state/事件队列 */
    sb_slot         slot;
    volatile int    state;      /* DETACHED/ATTACHED/LOST/DEGRADED/DEAD */
    uint64_t        gen;
    int32_t         refs;       /* 全局在途帧引用 */
    sb_post_event   post;
    void           *user;
    /* 事件队列（可选，避免在锁内回调） */
};
```

`detach`：

```
lock(m)
  if (id != NULL && 与当前 id 不匹配) { unlock; return SB_E_STATE; }
  slot.retired = 1; slot.win = NULL; gen++;
  state = DETACHED;
unlock
if (slot.refs == 0) unref(window)      // 无在途帧，立即归还
```

`release`：

```
lock(m)
  if (--slot.refs == 0 && slot.retired) { w = retired_win; retired_win=NULL; }
unlock
if (w) unref(w)
```

`acquire`：

```
lock(m)
  if (state != ATTACHED || !slot.win) { unlock; return SB_E_NO_WINDOW; }
  slot.refs++; out = {slot.win, slot.gen, w, h};
unlock
```

**不变量**：

- I1：`slot.win != NULL` 时，其引用计数 ≥1 + `slot.refs`。
- I2：`detach` 后，任何新 `acquire` 都失败；已发出的快照仍安全，直到 `release`。
- I3：`gen` 单调递增；渲染线程可比较 `frame.generation` 与当前 `gen`，决定是否丢帧。
- I4：`destroy` 只置 `DEAD` + `detach` + 等 `refs==0`（有界），绝不 `delete` 仍在使用的 player。

### 4.5 线程模型与事件

| 线程 | 允许调用 | 禁止 |
|---|---|---|
| UI（XComponent 回调 / ArkTS） | `sb_attach/sb_detach/sb_destroy` | 直接改 vout / 调 `sb_release` |
| 渲染 / vout | `sb_acquire/sb_release/sb_on_vdec_result` | `sb_destroy`、触碰 NAPI |
| NAPI 事件线程 | `napi_threadsafe_function` 投递 | 调 `sb_acquire` |
| Bridge 事件线程（可选） | `post()` 回调 | 在锁内调用 |

`post` 的线程标注写进 `sb.h` 注释：**回调在 Bridge 的受控上下文执行，不得阻塞超过 1 ms，不得回调 IJK 的锁**。

NAPI 适配器持有的 `napi_threadsafe_function` 必须在 player `release` 前 `napi_release_threadsafe_function(..., napi_tsfn_abort)`；事件回调里只投递、不直接调用 ArkTS，避免 `napi_env` 跨线程失效。

### 4.6 VDEC 异常：能做什么、不能做什么

**能做的（策略与状态隔离）**：

| 异常形态 | Bridge 行为 | 谁执行重建 |
|---|---|---|
| 解码返回负值（码流/参数/资源） | `sb_on_vdec_result(code<0)`：计数、退避、发 `SB_EV_VDEC_RETRY(n)`；达到 `vdec_retry_max` 后发 `SB_EV_VDEC_FATAL` | **IJK 的 vdec 接缝**（`ffpipenode_ffplay_vdec.c` / pipeline）执行 session 重建；Bridge 不拥有 codec，不越权 |
| 渲染 job 超时（`frame_stall_ms`） | 标记 stall、丢帧、发 `RETRY`；连续 stall 触发 `FATAL` | 同上 |
| window 失效（`SB_EV_LOST`） | 拒绝新帧，保留最后一帧；等重新 `attach` 后恢复 | 无 codec 重建，仅 surface 恢复 |
| EGL surface 创建失败 | 释放 EGL/slot 状态，发 `LOST`；重试一次 | vout 接缝 |

**不能做的（必须写进产品预期）**：

- SIGSEGV / SIGBUS / 驱动内核崩溃：进程内无法捕获，会直接杀进程；Bridge 只能通过“无裸指针、无跨线程 delete、window refcount”降低触发概率，**不能承诺隔离**。
- 硬件解码器的固件卡死：只能靠超时检测 + 重建，不能保证一定恢复。
- 如果产品硬性要求“渲染崩溃不影响播放进程”，唯一路径是系统级独立渲染进程（与本次“禁止独立进程”的约束冲突），需要在平台侧另行立项。

### 4.7 与 IJK 的接缝（≤80 行）

| 接缝 | 文件 | 设计 |
|---|---|---|
| vout 工厂 | `ijksdl_vout_android_surface.c:30-33` | `SDL_VoutAndroid_CreateForAndroidSurface()` 改为返回 `sb_vout_create()` 的 `SDL_Vout` 实例（保留原实现作为 `SIDECAR_SURFACE=0` 回退） |
| window 获取 | `ijksdl_vout_android_nativewindow.c:142` | `opaque->native_window` 改为每次 `sb_acquire()`；`IJK_EGL_display` 后 `sb_release()` |
| window 设置 | `ijkplayer_android.c:55-68` | `SDL_VoutAndroid_SetAndroidSurface` 只转调 `sb_attach/sb_detach`，不再直接写裸指针 |
| 初始化 | `ijkplayer_napi_proxy.cpp:303-320` + `ijkplayer_android.c:38` | `sb_ctx` 由 player 实例创建一次，指针存入 pipeline opaque；vout 工厂取用同一实例，**不在工厂内部 new**，否则 NAPI 事件入口拿不到同一对象 |
| NAPI | `ijkplayer_napi.cpp:1140-1221` | 回调只 `sb_attach/sb_detach`；删除 `delete instance` 逻辑，实例销毁交给 `ijkplayer_napi_proxy::IjkMediaPlayer_release` |
| 全局指针 | `ijkplayer_napi_proxy.cpp:311` | 删除 `GLOBAL_NATIVE_WINDOW`；window 只存在于 Bridge |
| 释放 | `ijkplayer_napi_proxy.cpp:465-475` | `release` 先 `sb_detach`（停止新帧）再 `sb_destroy`（有界），最后 `ijkmp_shutdown`；顺序不可颠倒 |

**不碰**：`ff_ffplay.c`、`ffpipenode_ffplay_vdec.c`、解码器、时钟、seek 主循环。

### 4.8 SURFACE 状态机

```
        sb_attach(win)
[DETACHED] ──────────────► [ATTACHED] ── sb_detach ──► [DETACHED]
     ▲                        │  ▲
     │                        │  │ sb_attach(new)
     │   sb_detach            │  ▼
     │                     [LOST] ── timeout ──► [DEGRADED]
     │                        ▲                    │
     │                        └── recover ─────────┘
     └──────── sb_destroy ── [DEAD] ◄── sb_destroy ──┘
```

| From | 事件 | To | 动作 |
|---|---|---|---|
| DETACHED | `attach(win)` | ATTACHED | Reference(win)，gen++，发 `ATTACHED` |
| ATTACHED | `attach(win2)` | ATTACHED | 旧 win 标记 retired，新 win Reference，gen++ |
| ATTACHED | `detach` | DETACHED | 摘链，gen++；refs==0 时 Unreference |
| ATTACHED | window 失效/EGL lost | LOST | 拒绝新 acquire，发 `LOST` |
| LOST | `attach(win)` | ATTACHED | 同首次 attach；渲染线程恢复 |
| LOST | 超时 | DEGRADED | 上报 `VDEC_FATAL` 或等待上层换源 |
| 任意 | `destroy` | DEAD | 幂等；有界等待在途帧；绝不 delete player |

---

## 5. 升级与合并兼容矩阵

| 升级动作 | Sidecar 本体 | IJK shim | `libavformat` | 结论 |
|---|---|---|---|---|
| 换 HAR（ArkTS 包） | 零改动 | 零改动 | 不涉及 | 完全兼容 |
| 换纯净 `libavformat.so` | 零改动 | 零改动 | 官方版 | 完全兼容（Sidecar 不依赖内部符号） |
| 升级 IJK native 源（`ff_ffplay.c`/`ijkavformat`） | 零改动 | 重放 5 行 URL 归一化 + 8 行 `ijkio` 分支（或 30 行 AVIO 分支） | — | 需一次 rebase；Sidecar 无感 |
| 升级 `libsmb2` ABI | 仅 `zsmb_transport.c` 适配 | 零改动 | — | 隔离在 Transport |
| 换 XComponent/窗口 API | 仅 `sb_core.c` 的 Reference/Unreference | 零改动 | — | 隔离在 Bridge |

**降级安全**：若 `zsmb_ijkio_protocol` 未注册（shim 丢失/Sidecar 缺失），`ijkio_alloc_url("zsmb:...")` 返回 NULL；播放器在 URL 归一化层捕获该失败，回退到 `smb_direct`（无预取直读）或官方 HTTP/本地路径，不会崩溃。SURFACE 侧 `SIDECAR_SURFACE=0` 编译开关回退旧 vout，行为与今天一致（包括已知风险）。

---

## 6. 演进与验证路径

### 6.1 第一步：剥离 SMB，建立独立组件库

**目标**：证明“插件化 SMB 能达到现有性能”，且 `libavformat` 恢复纯净。

1. 冻结基线：用当前 `0020` 版跑 DragHarness `Single/Burst/FarBack/BackForth`，保存 CSV、首帧、seek 首字节、PSS、hilog。
2. 建 `libzsmb_io.so`：Transport + IOManager + Ring + BlockCache + 状态机；先只实现 `open/read/seek/close/interrupt`。
3. 回滚 `libavformat`：读路径删除 ring/prefetch/块缓存，只保留“无预取直读”作为基线和回退；非读路径（目录/删除/移动）如仍需，收缩为 `smbmeta.c`；删除 `seek.c` 的 `smb` 特判。
4. 挂主路径：`ijkio:zsmb:`；加 5 行 URL 归一化 + 8 行 `ijkio_alloc_url` 分支；跑通播放。
5. 对照验收：
   - 同一片源、同一网络，首帧不慢于基线 10%。
   - `seekTo` 首字节 p50/p95：目标 <300 ms（LAN）。
   - 连续 20 次 seek 无 `ETIMEDOUT`。
   - `release` p95 <1 s（正常网络）。
   - PSS ≤ 基线 + 2 MiB。
6. 回滚点：URL 前缀切回 `smb://` + `SIDECAR_SMB=0`。

**退出条件**：功能/性能/内存三项均达标，且 `libavformat` 无 `libsmb2` 业务代码。

### 6.2 第二步：SURFACE Bridge 标准化

1. 实现 `libsurface_bridge.so`：slot/ref/gen/状态机/事件。
2. 替换 vout 工厂与 window 获取点（§4.7 的 5 个接缝）。
3. 改造 NAPI：XComponent 回调只 `sb_attach/sb_detach`；删除 `delete instance`；删除 `GLOBAL_NATIVE_WINDOW`。
4. 压力矩阵：
   - 页面进出 1000 次（attach/detach）。
   - 播放中 detach 50 次、立即 re-attach 50 次。
   - `release()` 与 `onSurfaceDestroyed` 并发 200 次。
   - 旋转/分屏/后台切换各 100 次。
   - 弱网 + seek 风暴中 detach（组合场景）。
5. 判据：0 崩溃、0 ASan/UBSan 报告、window ref 平衡（Reference/Unreference 计数为 0 差）、detach 后 200 ms 内无 display、re-attach 后 1 s 内恢复画面。
6. 回滚点：`SIDECAR_SURFACE=0` 回旧 vout。

### 6.3 第三步：DragHarness 50 次跨距风暴

在干净解耦的组件上重新压测，证明“架构隔离没有牺牲性能”。

新增/复用脚本：

| 脚本 | 参数 | 目的 |
|---|---|---|
| `SeekStorm50-A` | 50 次，0↔90%，间隔 60 ms | 极限合并窗压力 |
| `SeekStorm50-B` | 50 次，0↔90%，间隔 100 ms | 接近 `SeekGate` 120 ms |
| `SeekStorm50-C` | 50 次，随机 5%–95%，间隔 120 ms | 非均匀跨距 |
| `SeekStormRelease` | 风暴中第 25 次后 `release` | 发布会话生命周期 |
| `SurfaceChurn` | 播放中 detach/attach ×50 | 渲染桥稳定性 |

打点（复用现有 CSV）：

- `firstByteMs`：`seekTo` → 首次 `onInfo(10008/10100)` 或首帧回调。
- `droppedBytes`：Sidecar 作废字节（`gen` 丢弃 + 未发布子块）。
- `urgentHits`：urgent 256 KiB 首块命中率。
- `closeMs`：`release` 到 native 完成。
- `blockHitRate`：块缓存命中率（S-7 片源相关）。
- `windowRefs`：Bridge 的 Reference/Unreference 净值。

判据（与审计 7.3 对齐并加严）：

1. 50 次风暴：0 次 `ETIMEDOUT`，0 次 `ERR_FATAL`，最终 `pos` = 最后一次目标。
2. seek 首字节 p95 <300 ms（LAN）；p50 <150 ms。
3. `release` p95 <1 s；断网/半死连接 <2.5 s（若产品要求 <1 s，需先验证 `shutdown(fd)` 能力，见 §8）。
4. PSS 增量 <15 MiB；无单调增长。
5. `windowRefs` 净值 = 0；无 `LOST` 残留。
6. 全程 0 崩溃；ASan/TSan（host 单测）0 报告。

---

## 7. 单测矩阵（主机侧，无设备/无网络）

### 7.1 SMB（fake transport + 虚拟时钟）

| ID | 用例 | 注入 | 断言 |
|---|---|---|---|
| SMB-01 | 子块发布可见性 | 延迟 50 ms/子块 | 第一个 256 KiB 发布后读线程即可返回，不等 1 MiB |
| SMB-02 | gen 作废边界 | 30 ms 时 seek | 只丢在途子块；新首块 ≤256 KiB；`droppedBytes` 有界 |
| SMB-03 | urgent 首块 | seek 后 | 第一次 `pread.want <= 256 KiB` |
| SMB-04 | 50 次风暴 | 60/100/120 ms | 无超时；ticket 单调；最终 pos 正确 |
| SMB-05 | 中断时延 | 每次 pread 2 s | `interrupt` 后 100 ms 内返回 `ABORT`（请求边界） |
| SMB-06 | close 有界 | 半死 transport | `close` ≤ `close_timeout_ms`；无 UAF（ASan） |
| SMB-07 | 错误自愈 | 连续 3 次 `ETIMEDOUT` 后成功 | 不粘滞；`ERR_TRANSIENT` 发出；随后可读 |
| SMB-08 | 短读不误 EOF | `got<want` 多次 | 只有 `off+got==filesize` 才 EOF |
| SMB-09 | 块缓存并发 | 2 读者同 miss | 仅 1 次 transport 请求；两读者数据一致 |
| SMB-10 | ring 回绕 | 消费/预取跨 cap | 字节序列与文件一致（SHA256） |
| SMB-11 | 真环无 memmove | 大窗口滑动 | 持锁时间 <1 ms（instrumentation） |
| SMB-12 | trace 等价 | 录制 read/seek 序列回放 | 输出字节流 SHA256 与基线一致 |

### 7.2 SURFACE（host 侧用 fake window + 设备侧补充）

| ID | 用例 | 断言 |
|---|---|---|
| SB-01 | attach/detach 1000 次 | ref 净值 0；gen 单调 |
| SB-02 | display 中 detach | 在途帧仍可 release；之后无新帧 |
| SB-03 | destroy 与 display 并发 | 无 UAF；destroy 有界 |
| SB-04 | re-attach 新 window | 渲染恢复；旧 window 已 Unreference |
| SB-05 | 重复 detach / destroy | 幂等，返回 `SB_E_STATE` |
| SB-06 | NAPI 不持 env | 静态检查：`sb_core.c` 无 `napi_*` 符号 |
| SB-07 | VDEC 负值重试 | 第 3 次失败发 `VDEC_FATAL`；不无限重试 |
| SB-08 | 长稳 2 h 切换 | 无泄漏、无崩溃 |

---

## 8. 风险与待验证

| # | 风险 | 影响 | 处置 |
|---|---|---|---|
| R1 | `ijkio_alloc_url` 是硬编码，主路径必须改 IJK 约 13 行（URL 归一化 + 注册） | IJK native 升级需重放 | 备路径 Custom AVIO；两适配器共用核心 |
| R2 | `OH_NativeWindow_NativeObjectReference/Unreference` 的符号/语义待验证 | window 引用方案可能需换成 `OH_NativeWindow_NativeWindowHandleOpt` 或 XComponent 生命周期内保证 | 第一步先用最小 demo 验证；不行则退化为“只在回调有效期内使用 window + 渲染线程 gen 校验” |
| R3 | 硬崩溃不可捕获 | 无法承诺“VDEC 崩溃不拖垮进程” | 明确产品预期；降低触发概率；警示“独立渲染进程”是唯一硬隔离方案 |
| R4 | 半死 TCP 的 `pread` 不能在进程内强杀 | release 可能到 `io_timeout` 上限 | 优先 `smb2_set_timeout` + 1 MiB 粒度；验证 libsmb2 是否暴露 fd 以 `shutdown()`；否则把 `io_timeout` 降到 1 s 并接受误超时 |
| R5 | `ijkio` 的 `AVIOInterruptCB` 传递是 IJK 私有约定 | 升级可能断 | 适配器同时支持“无回调”模式（仅靠 `zsmb_interrupt`）；单测覆盖 |
| R6 | 块缓存对片源敏感（S-7） | 跨轨 >2 MiB 时抖动 | 默认 8×256 KiB，可配 16；用 `blockHitRate` 指标驱动，不拍脑袋加内存 |
| R7 | 一次 `ijkio` 跳转 + 可能的 adapter memcpy | 吞吐下降 | 实测 <2% 可接受；否则把 IOManager 的 read 直接映射到 `IjkURLContext->priv_data`，零额外拷贝 |
| R8 | 主项目 `SeekGate` epoch 与 Sidecar ticket 的语义对齐 | ack 归属仍有歧义 | `zsmb_seek` 返回 ticket，`SEEK_SETTLED` 带同一 ticket；主项目只认 ticket，不认 IJK 的 10008/600 |

---

## 9. 工作量拆解（人日）

假设：1 名熟悉 IJK/FFmpeg 的 native 工程师；不含主项目接入、HAR 发布流程、硬解驱动厂商问题；主机为 Windows/Linux，设备为 HarmonyOS arm64。

| 工作包 | 内容 | 人日 |
|---|---|---|
| W0 基线冻结 | 现有 DragHarness 全量跑、CSV/日志归档、指标脚本 | 2 |
| W1 SMB 核心 | Transport（libsmb2 封装 / cancel / timeout） | 3 |
| W2 SMB 核心 | IOManager（ring / blockcache / 状态机 / worker） | 6 |
| W3 SMB 核心 | 子块发布、urgent、错误预算、ticket | 2 |
| W4 SMB 挂载 | `ijkio` 适配器 + 5 行 URL 归一化 + 8 行分支 + 错误映射 | 2 |
| W5 SMB 备挂载 | Custom AVIO 适配器（可后置） | 2 |
| W6 SMB 单测 | fake transport、虚拟时钟、12 用例、ASan/TSan | 5 |
| W7 SMB 集成 | `libavformat` 回滚、`seek.c` 清理、构建/打包 | 2 |
| W8 SURFACE Bridge | slot/ref/gen/状态机/事件 | 5 |
| W9 SURFACE 接缝 | vout 工厂、window 获取、NAPI 改造 | 3 |
| W10 SURFACE 单测 | fake window、并发、1000 次 attach/detach | 4 |
| W11 DragHarness | 新增风暴/释放/切换脚本与打点 | 3 |
| W12 集成调优 | 性能回归、内存、日志、回退开关 | 3 |
| W13 文档评审 | 接口冻结、合并说明、风险复盘 | 2 |
| **小计** | | **44** |
| 风险缓冲 | 20% | **9** |
| **合计** | | **53 人日** |

分阶段：

| 阶段 | 工作包 | 人日 |
|---|---|---|
| 第一步 SMB 剥离 | W0–W7 | 24 |
| 第二步 SURFACE 标准化 | W8–W10 | 12 |
| 第三步 风暴与收口 | W11–W13 | 8 |
| 缓冲 | — | 9 |
| **总计** | | **53** |

并行建议：

- 2 人：SMB 与 SURFACE 可并行，约 **6 周**（含联调）。
- 3 人（SMB 2 + SURFACE 1）：约 **4 周**，前提是接口在 W0 后冻结。
- 单人顺序：约 **10–11 周**。

估算口径：上表为 P50；乐观 −20%，悲观 +30%（主要取决于 `OH_NativeWindow` 引用 API 与 `ijkio` 兼容性两项待验证结论）。

不含：主项目 `SeekGate`/`PlaybackCommandCenter` 改造（约 3–5 人日，另行评估）、HAR 发版、硬件解码器问题定位。

---

## 10. 交付清单

本阶段（设计）交付：

1. 本文档。
2. `zsmb.h` / `sb.h` 接口冻结稿（见 §3.3、§4.3）。
3. 状态机与锁纪律（见 §3.5、§3.7、§4.8）。
4. 挂载方案与事实核查（见 §0.2、§3.2）。
5. 升级兼容矩阵（见 §5）。
6. 分阶段验收与工作量（见 §6、§7、§9）。

明确不做：

- 不提交任何 C/C++ 代码改动。
- 不修改 `0020` patch、不修 F-1~F-4。
- 不调整现有 DragHarness 业务逻辑（只规划新增脚本）。

---

## 11. 附：一页纸决策摘要

```
SMB：  libavformat 纯净  ←  13 行 shim（URL 归一化 + ijkio 注册）  ←  libzsmb_io.so（核心零 FFmpeg）
       备路：30 行 Custom AVIO 注入（仅 ff_ffplay.c）
       数据：子块 256 KiB/1 MiB 发布；单锁 M + 原子 cancel；close 有界
SURFACE：UI 单向 attach/detach  ←  SurfaceBridge（ref + gen）  →  渲染线程 acquire/release
       禁止：napi_env / XComponent 指针 / 回调线程 delete / 裸 window
       能：生命周期、野指针、错误重试；不能：SIGSEGV 硬隔离
升级：  换 HAR / libavformat.so → Sidecar 零改动
       换 IJK native 源 → 重放 8~30 行 shim，Sidecar 零改动
验收：  50 次跨距 seek 0 超时；首字节 p95 <300 ms；release p95 <1 s；0 崩溃
```
