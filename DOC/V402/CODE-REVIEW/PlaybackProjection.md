- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/pages/player/PlaybackProjection.ets`
- 行号区间: `1-92`（全文；@Trace 字段声明在 11-19）

```ts
// pages/player/PlaybackProjection.ets —— v401 / P3：UI 唯一渲染缓存（只读镜像）
// 只由 PlayerView 订阅回调写入；seekPreviewMs 仅影响显示，不参与真值。
// 纯逻辑在 ProjectionCore（可单测）；本类只负责 @Trace 镜像。

import { PlaybackSnapshot, Unsubscribe } from '../../kernel/player_contract';
import { PlaybackSessionView, PlayerView } from '../../kernel/player_view';
import { ProjectionCore } from './ProjectionCore';

@ObservedV2
export class PlaybackProjection {
  @Trace state: string = 'Idle';
  @Trace positionMs: number = 0;
  @Trace durationMs: number = 0;
  @Trace committedUri: string = '';
  @Trace activeSlot: number = 0;
  @Trace muted: boolean = false;
  @Trace errorText: string = '';
  @Trace renderRole: string = 'none';
  @Trace seekPreviewMs: number = -1;
  private core: ProjectionCore = new ProjectionCore();
  private unsubState: Unsubscribe | null = null;
  private unsubPosition: Unsubscribe | null = null;
  private unsubSession: Unsubscribe | null = null;

  bind(view: PlayerView): void {
    this.dispose();
    const snap: PlaybackSnapshot = view.getSnapshot();
    this.applySnapshot(snap);
    this.applySession(view);
    this.unsubState = view.subscribeState((cur: PlaybackSnapshot, _prev: PlaybackSnapshot): void => {
      this.applySnapshot(cur);
    });
    this.unsubPosition = view.subscribePosition((positionMs: number, durationMs: number): void => {
      this.core.applyPosition(positionMs, durationMs);
      this.mirror();
    });
    this.unsubSession = view.subscribeSession((): void => {
      this.applySession(view);
    });
  }

  dispose(): void {
    if (this.unsubState !== null) { this.unsubState(); this.unsubState = null; }
    if (this.unsubPosition !== null) { this.unsubPosition(); this.unsubPosition = null; }
    if (this.unsubSession !== null) { this.unsubSession(); this.unsubSession = null; }
  }

  displayMs(): number {
    // 读 @Trace 字段：位置 tick 只让真正读取它的叶子组件重算。
    return this.seekPreviewMs >= 0 ? this.seekPreviewMs : this.positionMs;
  }

  setSeekPreview(ms: number): void {
    this.core.setSeekPreview(ms);
    this.seekPreviewMs = this.core.seekPreviewMs;
  }

  clearSeekPreview(): void {
    this.core.clearSeekPreview();
    this.seekPreviewMs = -1;
  }

  private mirror(): void {
    this.state = this.core.state;
    this.positionMs = this.core.positionMs;
    this.durationMs = this.core.durationMs;
    this.seekPreviewMs = this.core.seekPreviewMs;
  }

  private applySnapshot(s: PlaybackSnapshot): void {
    this.core.applySnapshot(s);
    this.mirror();
    if (s.lastError !== null) {
      this.errorText = s.lastError.message;
    }
  }

  private applySession(view: PlayerView): void {
    const session: PlaybackSessionView = view.getSession();
    this.core.committedUri = session.committedUri;
    this.core.activeSlot = session.activeSlot;
    this.core.muted = session.muted;
    this.core.renderRole = session.renderRole;
    this.committedUri = session.committedUri;
    this.activeSlot = session.activeSlot;
    this.muted = session.muted;
    this.renderRole = session.renderRole;
    if (session.visibleError !== null) {
      this.errorText = session.visibleError.message;
    }
  }
}
```
