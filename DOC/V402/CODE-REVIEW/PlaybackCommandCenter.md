- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/command/PlaybackCommandCenter.ets`
- 行号区间: `21-39`（成员声明）、`45-68`（submit）、`80-104`（enqueue）、`106-110`（advanceEpoch）、`166-175`（preemptRunning）；文件共 239 行，其余未拷

```ts
export class PlaybackCommandCenter {
  private runner: CommandRunner;
  private view: PlaybackCommandView;
  private budget: CommandBudget;
  private admission: CommandAdmissionCtl = new CommandAdmissionCtl();
  private queue: CommandQueue<QueuedCommand> = new CommandQueue<QueuedCommand>();
  private epoch: EpochLedger = new EpochLedger();
  private timer: HardTimer = new HardTimer();
  // 唯一执行槽。
  private running: QueuedCommand | null = null;
  private pumpQueued: boolean = false;
  private releasing: boolean = false;
  private pendingRelease: QueuedCommand | null = null;
  constructor(executor: PlaybackCommandExecutor, events: PlaybackEventPort, view: PlaybackCommandView,
    budget?: CommandBudget) {
    this.view = view;
    this.budget = budget !== undefined ? budget : DEFAULT_BUDGET;
    this.runner = new CommandRunner(executor, events, view, this.timer);
  }
```

```ts
  submit(cmd: PlaybackCommand): Promise<CommandResult> {
    const verdict: AdmissionVerdict = this.admission.check(cmd, this.view.getState(), this.releasing);
    if (verdict === 'Reject') {
      if (this.releasing && cmd.kind === 'Adopt') {
        hilog.error(0x0000, TAG, 'STORE_ADOPT_REJECTED reason=releasing uri=%{public}s gen=%{public}d',
          cmd.plan !== undefined ? cmd.plan.uri : '',
          cmd.plan !== undefined && cmd.plan.gen !== undefined ? cmd.plan.gen : 0);
      }
      return Promise.resolve('Rejected');
    }
    // Release 幂等：重复提交只挂同一个槽，不铸新代次、不丢在拆卸体。
    if (this.releasing && cmd.kind === 'Release' && this.pendingRelease !== null) {
      return this.pendingRelease.promise;
    }
    if (verdict === 'DeferPrepare') {
      // 此刻不铸 epoch，否则在飞 Release 的 settle 对不上代；执行前由 execute() 补铸。
      this.dropQueued(this.queue.dropPrepare());
      const deferred: QueuedCommand = makeQueuedCommand(cmd, 0, true);
      this.queue.push(deferred);
      this.schedulePump();
      return deferred.promise;
    }
    return this.enqueue(cmd);
  }
```

```ts
  private enqueue(cmd: PlaybackCommand): Promise<CommandResult> {
    let epoch: number = this.epoch.current();
    if (cmd.kind === 'Prepare' || cmd.kind === 'Adopt') {
      const prev: number = epoch;
      epoch = this.advanceEpoch();
      this.dropQueued(this.queue.dropPrepare());
      this.cancelRunningForSuccessor();
      this.runner.scheduleAbort(prev);
    } else if (cmd.kind === 'Release') {
      const prev: number = epoch;
      this.releasing = true;
      epoch = this.advanceEpoch();
      this.preemptRunning();
      this.dropQueued(this.queue.clear());
      this.runner.scheduleAbort(prev);
    } else {
      this.dropQueued(this.queue.dropKind(cmd.kind));
    }
    const entry: QueuedCommand = makeQueuedCommand(cmd, epoch, false);
    if (cmd.kind === 'Release') { this.pendingRelease = entry; }
    this.queue.push(entry);
    // 无 native 的 Adopt 保持同 tick 启动；其余命令一律让出事件栈。
    if (cmd.kind === 'Adopt' && this.running === null) { this.pumpNow(); } else { this.schedulePump(); }
    return entry.promise;
  }

  private advanceEpoch(): number {
    const next: number = this.epoch.advance();
    this.view.setGateEpoch(next);
    return next;
  }
```

```ts
  // 同步清槽 + 旧执行体 orphan；native abort 延后到宏任务。
  private preemptRunning(): void {
    const cur: QueuedCommand | null = this.running;
    if (cur === null) { return; }
    this.running = null;
    cur.orphaned = true;
    this.clearHardTimer(cur);
    cur.resolve('Superseded');
    cur.resolveReleaseResult('failed');
    this.runner.scheduleAbort(cur.epoch);
  }
```
