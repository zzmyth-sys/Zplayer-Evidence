- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/PlaybackEntryFlow.ets`
- 行号区间: `9-44`（业务分支：L22 `if (next === null)`；neighborOf L33 `if (videoIdx >= 0)`、L37 for 循环、L40 三元。文件共 44 行。）

```ts
export function registerPlaybackEntryFlow(bus: IntentBus, deps: FlowDeps): void {
  bus.subscribe(UiIntentKind.PlayItem, (raw: UiIntent): void => {
    const it: PlayItemIntent = raw as PlayItemIntent;
    runFlow(deps, async (): Promise<void> => {
      await deps.player.open(it.item, it.origin);
      await deps.settings.update({ lastPlayingItem: it.item.uri });
    });
  });

  bus.subscribe(UiIntentKind.StepItem, (raw: UiIntent): void => {
    const it: StepItemIntent = raw as StepItemIntent;
    runFlow(deps, async (): Promise<void> => {
      const next: MediaItem | null = neighborOf(deps, it.uri, it.delta);
      if (next === null) {
        return;
      }
      await deps.player.open(next, 'step');
      await deps.settings.update({ lastPlayingItem: next.uri });
    });
  });
}

function neighborOf(deps: FlowDeps, uri: string, delta: number): MediaItem | null {
  const videoIdx: number = deps.mediaLibrary.videoIndexOf(uri);
  if (videoIdx >= 0) {
    return deps.mediaLibrary.peekVideoAt(videoIdx + delta);
  }
  const items: MediaItem[] = deps.mediaLibrary.getItems();
  for (let i = 0; i < items.length; i++) {
    if (items[i].uri === uri) {
      const t: number = i + delta;
      return t >= 0 && t < items.length ? items[t] : null;
    }
  }
  return null;
}
```
