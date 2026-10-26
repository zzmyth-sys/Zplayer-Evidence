- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/prepare/SourceResolver.ets`
- 行号区间: `28-82`（指令写 `domains/smb/media_source/SourceResolver`，主仓实际位于 `prepare/`；`open/fallback`对应实体为 `resolve` / `resolveRequest` / `release`，文件共 82 行。）

```ts
export class SourceResolver {
  private source: MediaSourcePort;
  private handleOf: (src: PlayableSource) => StreamHandle;

  constructor(source: MediaSourcePort, handleOf: (src: PlayableSource) => StreamHandle) {
    this.source = source;
    this.handleOf = handleOf;
  }

  async resolve(item: MediaItem, gen: number, deadline: Deadline, avoid: SourceKind[]): Promise<PlayableSource> {
    return this.resolveRequest({ item: item, gen: gen, purpose: 'play', deadline: deadline, avoid: avoid }, deadline);
  }

  // v400 体检：预加载取源同一入口（purpose 区分）；句柄所有权仍归 PreloadCtl/SMB aux 槽。
  async resolvePreload(item: MediaItem, gen: number, deadline: Deadline): Promise<PlayableSource> {
    return this.resolveRequest({ item: item, gen: gen, purpose: 'preload', deadline: deadline, avoid: [] }, deadline);
  }

  private async resolveRequest(req: SourceRequest, deadline: Deadline): Promise<PlayableSource> {
    const t0: number = Date.now();
    try {
      const src: PlayableSource = await withDeadline(this.source.acquire(req), deadline, 'source-acquire');
      if (PERF_LOGS) {
        hilog.info(0x0000, TAG, '[PERF-SOURCE-ACQ] result=ok ms=%{public}d kind=%{public}s purpose=%{public}s uri=%{public}s',
          Date.now() - t0, src.source.kind, req.purpose, req.item.uri);
      }
      return src;
    } catch (e) {
      const err: Error = e as Error;
      if (PERF_LOGS) {
        hilog.info(0x0000, TAG, '[PERF-SOURCE-ACQ] result=fail ms=%{public}d purpose=%{public}s err=%{public}s uri=%{public}s',
          Date.now() - t0, req.purpose, err.message !== undefined ? err.message : '', req.item.uri);
      }
      if (err.name === 'DeadlineExceededError') {
        throw new SourceResolveError(-12, 'timeout', err.message);
      }
      if (err.message === 'play open canceled') {
        throw new SourceResolveError(0, 'canceled', err.message);
      }
      const fail: SourceFailureError = e as SourceFailureError;
      const code: number = fail.code !== undefined ? fail.code : -3;
      throw new SourceResolveError(code, 'open', err.message);
    }
  }

  // 透明 handle 引用：具体形状由装配层决定（player 不解释 SMB 租约字段）。
  handleRef(src: PlayableSource): StreamHandle {
    return this.handleOf(src);
  }

  // v400：能力闸门拒绝时在取源阶段就归还源（未进入 surface/lease，rollback 无此步）。
  release(src: PlayableSource, reason: string): Promise<void> {
    return this.source.release(src, reason);
  }
}
```
