- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/smb/media_source/SmbAdapter.ets`
- 行号区间: `31-43`（`open` 函数；文件共 68 行。）

```ts
  async open(req: SourceRequest, ref: SourceRef): Promise<PlayableSource> {
    if (req.purpose === 'preload') {
      const aux: StreamHandle | null =
        await this.smb.openAuxStream(this.smb.currentSession(), req.item.uri, req.item.mtime);
      if (aux === null) {
        throw this.fail(ref, req, 'smb: aux unavailable');
      }
      return this.wrap(aux, ref);
    }
    const h: StreamHandle = await this.smb.openPlayStream(this.smb.currentSession(), req.item.uri,
      req.item.mtime, req.gen, req.deadline);
    return this.wrap(h, ref);
  }
```
