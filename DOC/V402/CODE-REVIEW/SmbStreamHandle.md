- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/smb/SmbStreamHandle.ets`
- 行号区间: `60-77`（`static async create`；文件共 128 行。）

```ts
  static async create(sid: number, smbPath: string, queue: SmbOpQueue, sessionId: string,
    cache: ProxyCache, low: boolean, mtime: number, allowPreload: boolean,
    lease?: LeaseSource, deadline?: Deadline, priority?: PortPriority): Promise<SmbStreamHandleImpl> {
    const source: LeaseSource | null = lease !== undefined ? lease : null;
    const proxy: SmbHttpProxy = new SmbHttpProxy(sid, smbPath, queue, cache, low, mtime, allowPreload);
    if (source !== null) {
      proxy.setLeaseProvisioner(source.registry, source.provision);
    }
    const port: number = await proxy.start(deadline, priority);
    const handle: SmbStreamHandleImpl = new SmbStreamHandleImpl(sid, smbPath, queue, cache, low, mtime,
      allowPreload, sessionId, proxy, port, proxy.getSize(), source);
    const grant: LeaseGrant | null = proxy.leaseGrant();
    if (grant !== null) {
      handle.streamId = grant.streamId;
      handle.leaseId = grant.leaseId;
    }
    return handle;
  }
```
