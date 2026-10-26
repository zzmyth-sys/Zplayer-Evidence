- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/flows/playback/FlowErrorBoundary.ets`
- 行号区间: `7-16`（业务分支：无；L8-15 仅 try/catch 统一出口。文件共 16 行。）

```ts
export async function runFlow(deps: FlowDeps, task: () => Promise<void>): Promise<void> {
  try {
    await task();
  } catch (e) {
    deps.events.dispatch({
      kind: DomainEventKind.ErrorRaised,
      error: { code: -1, level: ErrorLevel.Recoverable, message: (e as Error).message ?? '' }
    });
  }
}
```
