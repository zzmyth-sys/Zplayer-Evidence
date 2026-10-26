- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/domains/player/command/CommandAdmissionCtl.ets`
- 行号区间: `1-29`（全文）

```ts
// domains/player/command/CommandAdmissionCtl.ets —— v401 / P2：命令准入策略（可单测）
// 只读 status + releasing 标志；不产生命令事件。

import { PlaybackCommand, PlaybackStatus } from '../store/PlaybackEventSink';

export type AdmissionVerdict = 'Admit' | 'Reject' | 'DeferPrepare';

export class CommandAdmissionCtl {
  check(cmd: PlaybackCommand, state: PlaybackStatus, releasing: boolean): AdmissionVerdict {
    if (state === 'Released' && cmd.kind !== 'Prepare' && cmd.kind !== 'Adopt' && cmd.kind !== 'Release') {
      return 'Reject';
    }
    if (state === 'Error' && (cmd.kind === 'Play' || cmd.kind === 'Pause')) {
      return 'Reject';
    }
    if (!releasing) {
      return 'Admit';
    }
    // Release 幂等由 CommandCenter 的 pendingRelease 槽负责，这里只放行。
    if (cmd.kind === 'Release') {
      return 'Admit';
    }
    // Release 在飞：Prepare 延迟铸 epoch；Adopt 必须与 pool.commit 同 tick，一律拒绝。
    if (cmd.kind === 'Prepare') {
      return 'DeferPrepare';
    }
    return 'Reject';
  }
}
```
