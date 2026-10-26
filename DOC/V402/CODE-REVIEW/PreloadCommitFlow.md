- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/PreloadCommitFlow.ets`
- 行号区间: `9-26`（业务分支：L14 `if (out.kind === 'Committed')`、L18 `if (out.kind === 'Fallback')`。文件共 26 行。）

```ts
export function registerPreloadCommitFlow(bus: IntentBus, deps: FlowDeps): void {
  bus.subscribe(UiIntentKind.CommitSelection, (raw: UiIntent): void => {
    const it: CommitSelectionIntent = raw as CommitSelectionIntent;
    runFlow(deps, async (): Promise<void> => {
      const out: CommitOutcome = await deps.player.commitPrepared(it.item);
      if (out.kind === 'Committed') {
        await deps.settings.update({ lastPlayingItem: out.item.uri });
        return;
      }
      if (out.kind === 'Fallback') {
        await deps.player.open(out.item, 'selection');
        await deps.settings.update({ lastPlayingItem: out.item.uri });
        return;
      }
      // NotReady：域已发 PreloadRejected / ErrorRaised，flow 不再补业务分支。
    });
  });
}
```
