- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/PlaybackControlFlow.ets`
- 行号区间: `7-23`（业务分支：无 if/switch（L12 仅等值比较生成 playing）。文件共 23 行。）

```ts
export function registerPlaybackControlFlow(bus: IntentBus, deps: FlowDeps): void {
  bus.subscribe(UiIntentKind.Control, (raw: UiIntent): void => {
    const it: ControlIntent = raw as ControlIntent;
    runFlow(deps, async (): Promise<void> => {
      await deps.player.control(it.op, it.value);
      const playing: boolean = deps.playerView.getView().getSnapshot().state === 'Playing';
      await deps.shell.updatePlaybackState(playing);
    });
  });

  bus.subscribe(UiIntentKind.SeekGesture, (raw: UiIntent): void => {
    const it: SeekGestureIntent = raw as SeekGestureIntent;
    runFlow(deps, async (): Promise<void> => {
      await deps.player.seekGesture(it.phase, it.ms);
    });
  });
}
```
