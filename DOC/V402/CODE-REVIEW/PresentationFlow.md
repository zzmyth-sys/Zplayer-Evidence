- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/PresentationFlow.ets`
- 行号区间: `6-11`（业务分支：无 if/switch（也无 runFlow 包装）。文件共 11 行。）

```ts
export function registerPresentationFlow(bus: IntentBus, deps: FlowDeps): void {
  bus.subscribe(UiIntentKind.Presentation, (raw: UiIntent): void => {
    const it: PresentationIntentMsg = raw as PresentationIntentMsg;
    deps.player.setPresentation(it.op);
  });
}
```
