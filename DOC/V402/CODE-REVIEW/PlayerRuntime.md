- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/PlayerRuntime.ets`
- 行号区间: `1-26`（全文；lastPrepareErrorCode 声明在 15）
- 唯一写入点: `entry/src/main/ets/domains/player/PlayerAssembly.ets:218`（附于本文件末尾）

```ts
// domains/player/PlayerRuntime.ets —— v401 / P3：运行期可变状态（无后端类型分支）
// AV 热切换已物理删除：恒 IJK，不存在后端类型运行期分支。

import { MediaItem, StreamHandle } from '../../kernel/types';
import { SourceProtocol } from '../../kernel/media_source';

export class PlayerRuntime {
  videoMuted: boolean = false;
  lastItem: MediaItem | null = null;
  lastUrl: string = '';
  lastHandle: StreamHandle | null = null;
  // 最后一次取源协议（HLS 失败判定 / 降级线）。
  lastProtocol: SourceProtocol = 'HTTP';
  // v402-041: last downgrade prepare error code (-13 = beyond device decode)
  lastPrepareErrorCode: number = 0;
  // v370 / 任务1（W3）：最近一次 commit 产生的活动租约。
  activeLeaseFresh: boolean = false;
  // v381：渲染全屏上下文。
  fullscreenRender: boolean = false;
  // v401 / P3：域层全屏舞台状态（UI 不再写全局存储）。
  fullscreenStage: boolean = false;
  // v386 / A2：ItemChanged 去重键（gen|uri）。
  publishedKey: string = '';
  releaseHook: (() => void) | null = null;
  releaseAuxIfHook: ((h: StreamHandle) => Promise<void>) | null = null;
}
```

写入点（PlayerAssembly.ets:214-220）：

```ts
      prepare: async (item: MediaItem, gen: number, deadline: Deadline,
        reqId: number, avoid: SourceKind[]): Promise<PrepareOutcome> => {
        const r: PrepareRunResult = await this.storePrep.prepareDetailed(item, gen, deadline, reqId, avoid);
        // v402-041: forward errorCode so a Failed downgrade can report the real cause.
        this.rt.lastPrepareErrorCode = r.errorCode;
        return r.outcome;
      }
```
