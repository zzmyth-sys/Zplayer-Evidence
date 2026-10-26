- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/PlayerAssembly.ets`
- 行号区间: `194-243`（makeDowngrade 的 releaseBackend(dead) 捕获/传递 + makeHlsDegrade 的 dead()/lastPrepareErrorCode/lastProtocol 注入 + failHlsStream 转发）；文件共 563 行
- 相关未拷区间: `246-258`（makeDeadlock.releaseCurrent 另有 dead 捕获）

```ts
  private makeDowngrade(): SourceDowngradeCtl {
    return new SourceDowngradeCtl({
      // 009a'-1：开关 false = 旧 uri#gen 行为零变化；attempt 由 009a'-2 接线。
      keyV2: HLS_DEGRADE_KEY_V2,
      publish: this.publish,
      isCurrent: (gen: number, uri: string): boolean => this.intent.isCurrent(gen, uri),
      beginGen: (uri: string): number => this.intentOps.beginIntent(uri),
      // v402：拉流阶段的死实例必须先销毁，再由新 prepare 重建（native 只允许一次 release）。
      // v402-007：只释放调用方持有的 dead 引用；若用户已切源（this.backend 已换人），不得回退覆盖新后端。
      releaseBackend: async (dead: PlayerBackend): Promise<void> => {
        try {
          await dead.release();
        } catch (e) {
          hilog.error(0x0000, TAG, 'hls downgrade release fail %{public}s',
            (e as Error).message !== undefined ? (e as Error).message : '');
        }
        if (this.backend === dead) {
          this.setBackend(this.pool.activeBackend());
        }
      },
      prepare: async (item: MediaItem, gen: number, deadline: Deadline,
        reqId: number, avoid: SourceKind[]): Promise<PrepareOutcome> => {
        const r: PrepareRunResult = await this.storePrep.prepareDetailed(item, gen, deadline, reqId, avoid);
        // v402-041: forward errorCode so a Failed downgrade can report the real cause.
        this.rt.lastPrepareErrorCode = r.errorCode;
        return r.outcome;
      }
    });
  }

  // v402 / 009b：唯一降级入口只做转发，分支全部在 HlsDegradeCtl 内（T-02 §6/§7）。
  failHlsStream(reason: string, code: number, message: string): Promise<boolean> {
    return this.hlsDegrade.fail(reason, code, message);
  }

  private makeHlsDegrade(): HlsDegradeCtl {
    return new HlsDegradeCtl({
      lastItem: (): MediaItem | null => this.rt.lastItem, lastUrl: (): string => this.rt.lastUrl,
      lastProtocol: (): SourceProtocol => this.rt.lastProtocol, gen: (): number => this.intent.gen(),
      lastPrepareErrorCode: (): number => this.rt.lastPrepareErrorCode,
      currentReqId: (): number => this.init.currentReqId(), dead: (): PlayerBackend => this.backend,
      isCurrent: (gen: number, uri: string): boolean => this.intent.isCurrent(gen, uri),
      downgrade: (item: MediaItem, gen: number, reqId: number, reason: string,
        dead: PlayerBackend): Promise<DowngradeResult> =>
        this.downgrade.downgradeFromHlsDetailed(item, gen, reqId, reason, dead),
      publish: this.publish,
      armStallWatchdog: (scope: RequestScope): void => { this.playback.armStallWatchdogScope(scope, smbStreamHint()); },
      engineEpoch: (): number => this.engine.getEpoch()
    });
  }
```
