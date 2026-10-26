- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/recovery/SourceDowngradeCtl.ets`
- 行号区间: `34-70`（deps 签名 + 字段声明：tried/inFlight/latestAttempt/attempts）、`100-182`（run + noteOpen：isCurrent/inFlight/tried/gen）、`249-268`（isV2/currentAttempt/isLatestKey）；文件共 281 行

```ts
export interface SourceDowngradeDeps {
  publish: EventPublisher;
  isCurrent: (gen: number, uri: string) => boolean;
  beginGen: (uri: string) => number;
  prepare: (item: MediaItem, gen: number, deadline: Deadline, reqId: number,
    avoid: SourceKind[]) => Promise<PrepareOutcome>;
  // v402：销毁本次降级持有的死实例（由调用方在失败时刻捕获）；失败不得阻断降级。
  // v402-007：必须释放传入引用，不得读“当前后端”（用户切源后当前已换人）。
  releaseBackend?: (dead: PlayerBackend) => Promise<void>;
  // 009a'-1：完整键开关（默认取 HLS_DEGRADE_KEY_V2）；attempt 只由 009a'-2 接线递增。
  keyV2?: boolean;
  attempt?: (uri: string, gen: number) => number;
  // 009a'-2 / C2：测试注入。返回 false 表示“删除被静默吞掉”，用于复现异常被吞。
  clearHook?: (key: string) => boolean;
}

export class SourceDowngradeCtl {
  private deps: SourceDowngradeDeps;
  // 同一「uri + gen」只降级一次：同一次尝试的重复 fatal 被吸收，重新打开会拿到新 gen 而放行。
  private tried: Set<string> = new Set<string>();
  // 009a'-1：仅 keyV2 开启时使用；键为 uri#gen#attempt（完整键）。
  private inFlight: Set<string> = new Set<string>();
  private latestAttempt: Map<string, number> = new Map<string, number>();
  // 009c / T9：同 uri prepare 串行链（FIFO）。
  private prepareChain: Map<string, Promise<void>> = new Map<string, Promise<void>>();
  // 009a'-2：attempt 由 noteOpen 递增（仅 user|selection 的 open 调用）。
  private attempts: Map<string, number> = new Map<string, number>();
  private epoch: number = 0;
  private clearedKeys: number = 0;
  private clearFail: number = 0;
  private totalClearedKeys: number = 0;
  private totalClearFail: number = 0;
  private lastClearWarned: boolean = false;

  constructor(deps: SourceDowngradeDeps) {
    this.deps = deps;
  }
```

```ts
  // disposeFirst：运行期死实例必须先销毁；prepare 期失败由流水线 rollback 负责，不重复动后端。
  private async run(item: MediaItem, gen: number, reqId: number, reason: string,
    disposeFirst: boolean, dead: PlayerBackend | null): Promise<DowngradeResult> {
    const v2: boolean = this.isV2();
    if (!this.deps.isCurrent(gen, item.uri)) {
      hilog.warn(0x0000, TAG, 'DOWNGRADE_SKIP trigger=%{public}s reason=stale-gen gen=%{public}d uri=%{public}s',
        reason, gen, item.uri);
      return 'Skipped';
    }
    const baseKey: string = item.uri + '#' + gen.toString();
    const attempt: number = v2 ? this.currentAttempt(item.uri, gen) : 0;
    const key: string = v2 ? baseKey + '#' + attempt.toString() : baseKey;
    // 完整键判据：迟到的旧 attempt 回调在入口即被判 stale（不依赖 isCurrent）。
    if (v2 && !this.isLatestKey(item.uri, gen, key)) {
      hilog.warn(0x0000, TAG, 'DOWNGRADE_SKIP trigger=%{public}s reason=stale-attempt gen=%{public}d attempt=%{public}d',
        reason, gen, attempt);
      return 'Skipped';
    }
    if (v2) {
      if (this.inFlight.has(key)) {
        hilog.warn(0x0000, TAG, 'DOWNGRADE_SKIP trigger=%{public}s reason=inflight gen=%{public}d attempt=%{public}d',
          reason, gen, attempt);
        return 'InFlight';
      }
      this.inFlight.add(key);
      this.latestAttempt.set(baseKey, attempt);
    }
    if (this.tried.has(key)) {
      return 'Skipped';
    }
    this.tried.add(key);
    try {
      // v402：死实例先释放（native 只允许一次 release，重复调用是幂等的），
      // 再走新 gen 取源；这里失败不能阻断降级（新源仍可能起来）。
      if (disposeFirst && dead !== null) {
        await this.releaseBackend(dead);
        // await 前后复检：释放期间用户可能已切源，放弃本次降级（不取源、不报错）。
        if (!this.deps.isCurrent(gen, item.uri) || (v2 && !this.isLatestKey(item.uri, gen, key))) {
          hilog.warn(0x0000, TAG, 'DOWNGRADE_SKIP trigger=%{public}s reason=stale-after-release gen=%{public}d',
            reason, gen);
          return 'Skipped';
        }
      }
      const nextGen: number = this.deps.beginGen(item.uri);
      // 降级是一次全新的取源尝试：给完整预算，不继承已消耗的旧 deadline。
      const outcome: PrepareOutcome = await this.serialPrepare(item, nextGen, reqId);
      // 迟到结果：取源期间已切源/被新 attempt 接管 → 不认领、不发布错误。
      if (!this.deps.isCurrent(nextGen, item.uri) || (v2 && !this.isLatestKey(item.uri, gen, key))) {
        hilog.warn(0x0000, TAG, 'DOWNGRADE_SKIP trigger=%{public}s reason=stale-after-prepare gen=%{public}d/%{public}d',
          reason, gen, nextGen);
        return 'Skipped';
      }
      if (outcome === 'Ok') {
        hilog.warn(0x0000, TAG, 'DOWNGRADE_OK trigger=%{public}s uri=%{public}s', reason, item.uri);
        return 'Ok';
      }
      this.tried.delete(key);
      hilog.error(0x0000, TAG, 'DOWNGRADE_FAIL trigger=%{public}s outcome=%{public}s uri=%{public}s',
        reason, outcome, item.uri);
      this.deps.publish({
        kind: DomainEventKind.PlaybackError,
        error: { code: -4, level: ErrorLevel.Recoverable, message: 'hls downgrade failed' }
      });
      return 'Failed';
    } finally {
      if (v2) {
        this.inFlight.delete(key);
        if (this.latestAttempt.get(baseKey) === attempt) {
          this.latestAttempt.delete(baseKey);
        }
      }
    }
  }

  // 009a'-2：仅 open(user|selection) 调用；同 uri+gen 重开 attempt+1。
  noteOpen(uri: string, gen: number): void {
    if (!this.isV2()) {
      return;
    }
    const base: string = uri + '#' + gen.toString();
    const cur: number | undefined = this.attempts.get(base);
    this.attempts.set(base, cur === undefined ? 0 : cur + 1);
  }
```

```ts
  private isV2(): boolean {
    return this.deps.keyV2 === undefined ? HLS_DEGRADE_KEY_V2 : this.deps.keyV2;
  }

  private currentAttempt(uri: string, gen: number): number {
    if (this.deps.attempt !== undefined) {
      return this.deps.attempt(uri, gen);
    }
    return this.attempts.get(uri + '#' + gen.toString()) ?? 0;
  }

  private isLatestKey(uri: string, gen: number, key: string): boolean {
    const latest: number | undefined = this.latestAttempt.get(uri + '#' + gen.toString());
    if (latest === undefined) {
      return true;
    }
    const parts: string[] = key.split('#');
    const incoming: number = Number(parts[parts.length - 1]);
    return incoming >= latest;
  }
```
