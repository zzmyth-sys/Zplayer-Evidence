- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/recovery/HlsDegradeCtl.ets`
- 行号区间: `1-74`（全文，含 fail 与 inFlight）

```ts
import { hilog } from '@kit.PerformanceAnalysisKit';
import { DomainEventKind, EventPublisher } from '../../../kernel/events';
import { ErrorLevel, MediaItem } from '../../../kernel/types';
import { isHlsUrlPath } from '../../../kernel/stream_failure';
import { SourceProtocol } from '../../../kernel/media_source';
import { PlayerBackend } from '../PlayerBackend';
import { RequestScope } from '../request/RequestScope';
import { DowngradeResult } from './SourceDowngradeCtl';
import { SOURCE_UNSUPPORTED_CODE, SOURCE_UNSUPPORTED_MESSAGE } from '../../media_probe/ClipCapability';
const TAG: string = 'ZPLAYER/HlsDegrade';
export interface HlsDegradeDeps {
  lastItem: () => MediaItem | null;
  lastProtocol: () => SourceProtocol;
  lastPrepareErrorCode: () => number;
  lastUrl: () => string;
  gen: () => number;
  currentReqId: () => number;
  dead: () => PlayerBackend;
  isCurrent: (gen: number, uri: string) => boolean;
  downgrade: (item: MediaItem, gen: number, reqId: number, reason: string,
    dead: PlayerBackend) => Promise<DowngradeResult>;
  publish: EventPublisher;
  armStallWatchdog: (scope: RequestScope) => void;
  engineEpoch: () => number;
}
export class HlsDegradeCtl {
  private deps: HlsDegradeDeps;
  // 装配层重入吸收：一次只允许一个降级在飞（T5/T7 的 InFlight 静默）。
  private inFlight: boolean = false;
  constructor(deps: HlsDegradeDeps) {
    this.deps = deps;
  }
  // S1 裁决（V402-020）：早退路径依赖在 this.inFlight = true 之前返回；新增早退分支
  // 必须确保在置位前退出，否则 inFlight 会残留 true，导致降级永久被吸收。
  async fail(reason: string, code: number, message: string): Promise<boolean> {
    if (this.inFlight) { return false; }
    const item: MediaItem | null = this.deps.lastItem();
    if (item === null) { return false; }
    if (this.deps.lastProtocol() !== 'HLS' && !isHlsUrlPath(this.deps.lastUrl())) {
      this.deps.publish({
        kind: DomainEventKind.ErrorRaised,
        error: { code: code, level: ErrorLevel.Fatal, message: message }
      });
      return false;
    }
    hilog.warn(0x0000, TAG, 'HLS_STREAM_FAIL reason=%{public}s code=%{public}d uri=%{public}s',
      reason, code, item.uri);
    this.inFlight = true;
    try {
      const gen: number = this.deps.gen(); const reqId: number = this.deps.currentReqId();
      const result: DowngradeResult = await this.deps.downgrade(item, gen, reqId, reason, this.deps.dead());
      if (result === 'Ok') {
        // 降级成功没有 init ticket：补一份 scope，停滞看门狗否则会永久缺席。
        const scope: RequestScope = new RequestScope(reqId, this.deps.gen(), item.uri);
        scope.bindEpoch(this.deps.engineEpoch());
        this.deps.armStallWatchdog(scope);
        return true;
      }
      if (result === 'Failed' && this.deps.isCurrent(gen, item.uri)) {
        // v402-041: prefer downgrade prepare error (-13 = unsupported), else trigger error.
        const prep: number = this.deps.lastPrepareErrorCode();
        const shownCode: number = prep === SOURCE_UNSUPPORTED_CODE ? prep : code;
        const shownMsg: string = prep === SOURCE_UNSUPPORTED_CODE ? SOURCE_UNSUPPORTED_MESSAGE : message;
        this.deps.publish({
          kind: DomainEventKind.ErrorRaised,
          error: { code: shownCode, level: ErrorLevel.Fatal, message: shownMsg }
        });
      }
      return false;
    } finally {
      this.inFlight = false;
    }
  }
}
```
