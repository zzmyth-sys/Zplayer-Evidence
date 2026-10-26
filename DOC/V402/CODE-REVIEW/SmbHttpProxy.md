- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/smb/SmbHttpProxy.ets`
- 行号区间: `77-143`（`start` 函数；文件共 479 行。）

```ts
  async start(deadline?: Deadline, priority?: PortPriority): Promise<number> {
    try {
      this.size = this.low
        ? await this.queue.runLow<number>((): Promise<number> => {
          return Promise.resolve(smbStatSize(this.sid, this.smbPath));
        })
        : await this.queue.run<number>((): Promise<number> => {
          return Promise.resolve(smbStatSize(this.sid, this.smbPath));
        });
    } catch (e) {
      if (PROBE_LOGS) {
        hilog.error(0x0000, TAG, '[PROBE] start fail statSize err=%{public}s path=%{public}s', (e as Error).message ?? '', this.smbPath);
      }
      throw (e as Error);
    }
    // v73 / B1.4：native statSize 对缺失文件返回 0（不抛异常）→ 视作失败，避免静默黑屏
    if (this.size <= 0) {
      if (PROBE_LOGS) {
        hilog.error(0x0000, TAG, '[PROBE] start fail statSize<=0 path=%{public}s', this.smbPath);
      }
      throw new Error('SMB stat failed or empty (size=' + this.size.toString() + ')');
    }
    this.parts.size = this.size;
    const portDeadline: Deadline = deadline !== undefined
      ? deadline.child(this.low ? PORT_WAIT_AUX_MS : PORT_WAIT_PLAY_MS)
      : Deadline.after(this.low ? PORT_WAIT_AUX_MS : PORT_WAIT_PLAY_MS);
    const pri: PortPriority = priority !== undefined ? priority : (this.low ? 'Aux' : 'Play');
    const lease: PortLease = await gPortPool.acquire(portDeadline, pri);
    const port: number = lease.port;
    this.portLease = lease;
    this.port = port;
    // P1-1：createLease 紧跟 acquire 成功点；此后任何 close 都必须经 lease 撤销。
    if (this.leaseRegistry !== null && this.provisioner !== null) {
      const grant: LeaseGrant = this.provisioner(port, 'http://127.0.0.1:' + port.toString() + '/video.mp4');
      this.leaseId = grant.leaseId;
      this.leaseStreamId = grant.streamId;
    }
    this.server = socket.constructTCPSocketServerInstance();
    try {
      await this.server.listen({ address: '127.0.0.1', port: port, family: 1 });
    } catch (e) {
      if (PROBE_LOGS) {
        hilog.error(0x0000, TAG, '[PROBE] start fail listen port=%{public}d err=%{public}s', port, (e as Error).message ?? '');
      }
      try {
        await this.server.close();
      } catch (e2) {
      }
      await this.rollbackLease('listenFail');
      throw (e as Error);
    }
    // 仅 listen 成功后注册 kill 回调：listen 失败路径只做零 holder 回滚 + PORT_RETURN。
    if (this.leaseRegistry !== null && this.leaseId !== '') {
      this.leaseRegistry.onRevoked(this.leaseId, (): Promise<void> => this.killByLease(this.leaseId));
    }
    this.server.on('connect', (client: socket.TCPSocketConnection) => {
      this.clients.push(client);
      this.onConnect(client);
    });
    this.alive = true;
    hilog.info(0x0000, TAG, 'listening 127.0.0.1:%{public}d size=%{public}d', port, this.size);
    // v317：小文件低优全量预取（<=10MB），播放起播后可秒开后续块
    if (this.allowPreload && this.size <= PRELOAD_MAX_BYTES) {
      this.startPreload();
    }
    return port;
  }
```
