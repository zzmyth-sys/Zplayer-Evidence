- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/PlaybackLifecycleFlow.ets`
- 行号区间: `7-15`（业务分支：无 if/switch。文件共 15 行。）

```ts
export function registerPlaybackLifecycleFlow(bus: IntentBus, deps: FlowDeps): void {
  bus.subscribe(UiIntentKind.Release, (raw: UiIntent): void => {
    const it: ReleaseIntent = raw as ReleaseIntent;
    runFlow(deps, async (): Promise<void> => {
      deps.player.setPresentation('ExitFullscreen');
      await deps.player.release(it.reason);
    });
  });
}
```
