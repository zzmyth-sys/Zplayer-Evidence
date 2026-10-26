- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/PreloadPrepareFlow.ets`
- 行号区间: `7-14`（业务分支：无 if/switch。文件共 14 行。）

```ts
export function registerPreloadPrepareFlow(bus: IntentBus, deps: FlowDeps): void {
  bus.subscribe(UiIntentKind.PreloadAdjacent, (raw: UiIntent): void => {
    const it: PreloadAdjacentIntent = raw as PreloadAdjacentIntent;
    runFlow(deps, async (): Promise<void> => {
      await deps.player.prepareAdjacent(it.targetIndex);
    });
  });
}
```
