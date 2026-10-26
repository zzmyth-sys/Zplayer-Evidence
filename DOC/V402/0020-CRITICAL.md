# 0020-CRITICAL · V402-049 C 线路径二前置审查（仅三类核心代码）

> 对象：`ZplayerDemo/native/src/ohos_ffmpeg/libavformat/libsmb2.c`（0020 最终 868 行）
> DEMO commit：`650b310`；审查范围仅三类，其他 766 行不贴。
> 代码行号为该文件当前行号；引用约 102 行。

## ① 是否绕过 PlaybackCommandCenter 直接下命令

`libsmb2.c:376-397`（`url_open` 入口；只接 `av_application_*`，无 ArkTS/CommandCenter）：

```c
static av_cold int libsmb2_open(URLContext *h, const char *uri, int flags)
{
    int ret = 0;
    LIBSMB2Context *libsmb2 = h->priv_data;
    libsmb2->filesize = -1;
    libsmb2->pos = 0;
    h->max_packet_size = 0;

    pthread_mutex_init(&libsmb2->lock, NULL);
    pthread_mutex_init(&libsmb2->net_lock, NULL);
    pthread_cond_init(&libsmb2->cond, NULL);
    libsmb2->inited = 1;
    libsmb2->ctx = smb2_init_context();
    libsmb2->app_ctx = (AVApplicationContext *)av_dict_strtoptr(libsmb2->app_ctx_intptr);

    av_application_will_http_open(libsmb2->app_ctx, (void *)h, uri);

    if (!libsmb2->ctx) {
        av_log(h, AV_LOG_ERROR, "smb2 create context failed: %s.\n", smb2_get_error(libsmb2->ctx));
        ret = AVERROR(ENOMEM);
        goto failed;
    }

    const char *smb_url = av_strireplace(uri, "smb2", "smb");
    struct smb2_url *url = smb2_parse_url(libsmb2->ctx, smb_url);
```

`libsmb2.c:839-854`（URLProtocol 注册；没有命令门禁/回调）：

```c
const URLProtocol ff_libsmb2_protocol = {
    .name = "smb",
    .flags           = URL_PROTOCOL_FLAG_NETWORK,
    .priv_data_size  = sizeof(LIBSMB2Context),
    .priv_data_class = &libsmb2lient_context_class,
    .url_open        = libsmb2_open,
    .url_read        = libsmb2_read,
    .url_write       = libsmb2_write,
    .url_seek        = libsmb2_seek,
    .url_close       = libsmb2_close,
    .url_delete      = libsmb2_delete,
    .url_move        = libsmb2_move,
    .url_open_dir    = libsmb2_open_dir,
    .url_read_dir    = libsmb2_read_dir,
    .url_close_dir   = libsmb2_close_dir,
};
```

**判定：不命中。** 868 行内无 `PlaybackCommandCenter`/`CommandCenter`/`PlaybackCommand` 引用；它是 FFmpeg 的 `URLProtocol` 下层，由 IJK `setDataSource` 触发。集成风险不在 0020，而在主项目是否只允许经 CommandCenter 的 `Prepare` 调 `setDataSource(smb://)`。

## ② 是否破坏 SURFACE 零拷贝（额外 buffer 拷贝/回读）

`libsmb2.c:231-250`（预取线程：`scratch → ring` 一次 `memcpy`）：

```c
        gen = s->gen;
        pthread_mutex_unlock(&s->lock);
        rd = smb2_pread_full(s, s->scratch, want, off);
        pthread_mutex_lock(&s->lock);

        if (s->quitting)
            break;
        if (gen != s->gen) {
            /* window was moved while this read was in flight */
        } else if (rd < 0) {
            s->err = smb2_errno_to_averror(rd);
            pthread_cond_broadcast(&s->cond);
        } else if (rd == 0) {
            s->eof = 1;
            pthread_cond_broadcast(&s->cond);
        } else {
            memcpy(s->ring + s->ring_len, s->scratch, rd);
            s->ring_len += rd;
            pthread_cond_broadcast(&s->cond);
        }
```

`libsmb2.c:599-613`（读路径：`ring → AVIO buf` 一次 `memcpy`）：

```c
        if (pos >= libsmb2->ring_base &&
            pos < (uint64_t)libsmb2->ring_base + libsmb2->ring_len) {
            off = (uint32_t)(pos - libsmb2->ring_base);
            avail = libsmb2->ring_len - off;
            n = FFMIN((uint32_t)(size - out), avail);
            memcpy(buf + out, libsmb2->ring + off, n);
            out += n;
            libsmb2->pos = pos + n;
            if (off + n > libsmb2->consume_off)
                libsmb2->consume_off = off + n;
            if (libsmb2->consume_off >= libsmb2->ring_cap / 2)
                pthread_cond_broadcast(&libsmb2->cond);
            libsmb2->miss_net_bytes = 0;
            wait_start = 0;
            continue;
        }
```

**判定：按 SURFACE 定义不命中“零拷贝破坏”，但“额外 buffer 拷贝”命中 2 处。** 这两处是**网络缓冲拷贝**（scratch→ring→AVIO），文件内无 `AVFrame`/`surface`/`glReadPixels`/readback；解码→Surface 的帧路径未被 0020 触碰。若判据把任何额外 memcpy 都算破坏，则这两处要计；若按 SURFACE 零拷贝（帧不回读）定义，则不破坏。代价是 CPU/带宽，需在 A/B CPU 里计入。

## ③ 是否引入 `if (newWay === null)` 走旧逻辑的运行时双路径

`libsmb2.c:568-587`（`!ring` 时退回直接 `smb2_pread_full` 的旧读路径）：

```c
    if (!libsmb2->ring) {
        uint32_t max_read = 0;
        int rd;

        if (libsmb2->ctx)
            max_read = smb2_get_max_read_size(libsmb2->ctx);
        if (max_read == 0)
            max_read = 1024 * 1024;
        if (libsmb2->err)
            return libsmb2->err;
        rd = smb2_pread_full(libsmb2, buf, FFMIN((uint32_t)size, max_read),
                             libsmb2->pos);
        if (rd < 0)
            return smb2_errno_to_averror(rd);
        if (rd == 0)
            return AVERROR_EOF;
        libsmb2->pos += rd;
        av_application_did_io_tcp_read(libsmb2->app_ctx, (void *)h, rd);
        return rd;
    }
```

`libsmb2.c:642-650`（前跳只重置环，不是旧逻辑分支）：

```c
        if (pos > (uint64_t)libsmb2->ring_base + libsmb2->ring_len) {
            ring_reset(libsmb2, pos);
            wait_start = 0;
            continue;
        }

        /* Behind the sequential window: small stashed track or a real
         * backward seek.  Serve it from the block cache so the sequential
         * prefetch stream is not disturbed. */
```

**判定：形式命中 1 处，低风险，非致命。** `if (!libsmb2->ring)` 是第二条读路径（直接 pread），但文件 `open` 在 `:491-495` 对 `ring/scratch` 分配失败直接 `goto failed`，正常文件打开不会进入该分支；它不是 `newWay===null` 的业务旧逻辑，只是防御性 fallback。建议改成断言/删除，或在注释中明确“仅目录/异常路径”。

## 结论（只裁致命项）

| 类别 | 命中 | 致命性 |
|---|---|---|
| 绕过 CommandCenter | **否**（0 引用） | 无 |
| 破坏 SURFACE 零拷贝 | **否**（仅网络 memcpy，无 frame readback） | 无 |
| 运行时双路径 | **1 处形式命中**（`!ring` 直接 pread；正常不可达） | 低风险，非致命 |

**净结论：0020 无致命项；1 处防御性双路径建议清理。**
