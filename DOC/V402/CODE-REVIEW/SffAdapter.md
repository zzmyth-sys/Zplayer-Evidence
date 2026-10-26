- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/smb/media_source/SffAdapter.ets`
- 行号区间: `32-113`（`open` 函数；文件共 146 行。）

```ts
  async open(req: SourceRequest, ref: SourceRef): Promise<PlayableSource> {
    // v400 体检：预加载必须走 MediaSourcePort(purpose='preload')，SFF 侧未实现槽位交接，
    // 显式让位给 SMB aux（不建服务端会话），行为与迁移前一致。
    if (req.purpose === 'preload') {
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: preload → smb aux');
    }
    const share: string = this.shareName().trim();
    if (share === '') {
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: empty share name');
    }
    // v400：熔断中直接让位（0 等待），不再每次点播都等连接超时。
    if (this.transport.isCircuitOpen()) {
      if (PERF_LOGS) {
        hilog.info(0x0000, 'ZPLAYER/SffAdapter',
          '[PERF-V4] event=sff-breaker action=skip ms=0 uri=%{public}s', req.item.uri);
      }
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: circuit open');
    }
    const tVfs: number = Date.now();
    let localPath: string;
    try {
      localPath = await this.transport.resolveLocalPath(share, req.item.uri, req.deadline);
    } catch (e) {
      throw this.transportFail(ref, req, 'vfs', tVfs, e);
    }
    const tProbe: number = Date.now();
    let probe: SffProbeInfo;
    try {
      probe = await this.transport.probe(localPath, req.deadline);
    } catch (e) {
      throw this.transportFail(ref, req, 'probe', tProbe, e);
    }
    const tStart: number = Date.now();
    // 契约 v1.7 只允许 direct|remux|transcode；规范化后再路由，未知值直接回落 SMB。
    const method: string = probe.method.trim().toLowerCase();
    const layout: string = probe.splicing !== undefined && probe.splicing !== null
      ? probe.splicing.layout : 'single';
    const needCrop: boolean = layout === 'triple' || layout === 'double';
    if (method === SFF_METHOD_DIRECT && !needCrop) {
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: direct → play smb');
    }
    if (method !== 'transcode' && method !== 'remux' && method !== 'direct') {
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: unsupported method ' + probe.method);
    }
    // direct + 拼接：源本身可直读，但裁切事实只在 /probe 里；走 SFF HLS（服务端 segment_method=transcode）
    // 才能在客户端按 crop_rect 渲染。非拼接 direct 仍回落 SMB（性能优先）。
    const startMethod: string = method === SFF_METHOD_DIRECT ? 'transcode' : method;
    let stream: SffStreamInfo;
    try {
      stream = await this.transport.start(localPath, startMethod, req.deadline);
    } catch (e) {
      throw this.transportFail(ref, req, 'start', tStart, e);
    }
    if (stream.streamId === '') {
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: empty streamId');
    }
    // 取源总预算已过：不留服务端会话。
    if (req.deadline.expired()) {
      await this.transport.stop(stream.streamId).catch((): void => {
      });
      throw this.fail(ref, req, SourceFailCode.AttemptFallback, 'sff: open deadline');
    }
    const splicing: SffSplicing | null = probe.splicing !== undefined && probe.splicing !== null
      ? probe.splicing : null;
    // v400 终局：IJK 直连 SFF HLS（实测起播 87–140ms），本地中继已物理删除。
    const url: string = this.hlsUrl(stream.streamId);
    if (PERF_LOGS) {
      hilog.info(0x0000, 'ZPLAYER/SffAdapter',
        '[PERF-SOURCE-ACQ] stage=sff ms=%{public}d vfs=%{public}d probe=%{public}d start=%{public}d method=%{public}s id=%{public}s',
        Date.now() - tVfs, tProbe - tVfs, tStart - tProbe, Date.now() - tStart,
        startMethod, stream.streamId);
    }
    const payload: SffPayload = {
      url: url, size: req.item.size, mtime: req.item.mtime,
      sffStreamId: stream.streamId, localPath: localPath, splicing: splicing
    };
    const crop: SourceCrop | null = cropOfSplicing(splicing);
    const out: PlayableSource = {
      url: url, size: req.item.size, mtime: req.item.mtime, source: ref, payload: payload, crop: crop
    };
    return out;
  }
```
