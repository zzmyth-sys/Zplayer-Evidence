- 主仓 commit: `52ba806a0d13540ceedac1642cc91ec097c68845`
- 原路径: `entry/src/main/ets/kernel/player_contract.ets`
- 行号区间: `108-208`（文件共 208 行；按 200 行阈值只拷命令/DTO 契约段，1-107 为几何/渲染/失败契约，未纳入）

```ts
// v389：PlaybackStore 对 UI/层的只读契约（kernel 不依赖领域实现）。
export type PlaybackCommandKind = 'Prepare' | 'Adopt' | 'Play' | 'Pause' | 'Seek' | 'SetRate' | 'Release';
export type PlaybackStatus = 'Idle' | 'Prepared' | 'Playing' | 'Paused' | 'Error' | 'Released';
export type PlaybackErrorSeverity = 'recoverable' | 'fatal';
export type Unsubscribe = () => void;

export interface PlaybackFailure {
  code: number;
  message: string;
  severity: PlaybackErrorSeverity;
  atMs: number;
}

export interface PlaybackSnapshot {
  revision: number;
  positionRevision: number;
  uri: string;
  streamId: string;
  epoch: number;
  state: PlaybackStatus;
  stateSinceMs: number;
  pendingKind: PlaybackCommandKind | '';
  pendingUri: string;
  positionMs: number;
  durationMs: number;
  positionSampledAtMs: number;
  rate: number;
  ended: boolean;
  buffering: boolean;
  lastError: PlaybackFailure | null;
  // v392：首帧上屏旁路事实（不参与 status/revision 推进，仅诊断与 UI 对齐）。
  firstFrameUri: string;
  firstFrameRevision: number;
  // v400：SFF 拼接裁切（原图像素框；null = 不裁切）。必填，禁止 optional + 运行时缺省。
  sourceCrop: SourceCrop | null;
}

export interface PlaybackUiView {
  getSnapshot(): PlaybackSnapshot;
  subscribeState(fn: (cur: PlaybackSnapshot, prev: PlaybackSnapshot) => void): Unsubscribe;
  subscribePosition(fn: (positionMs: number, durationMs: number) => void): Unsubscribe;
}

// v398 / S1：播放初始化层的只读投影（kernel 只定义形状；推导是纯函数，唯一实现在 domains/player/init）。
export type InitPhase = 'Idle' | 'Preparing' | 'Nominated' | 'Committing'
  | 'Settled' | 'Canceled' | 'Failed' | 'Superseded';
export type InitOutcome = 'Ok' | 'Superseded' | 'Canceled' | 'PrepareFailed' | 'GateTimeout';

export interface InitPhaseView {
  phase: InitPhase;
  uri: string;
  state: PlaybackStatus;
  pendingKind: PlaybackCommandKind | '';
  lastOutcome: InitOutcome | '';
  lastErrorCode: number;
}

// v399 / S2：初始化层的不可变输入（跨 Port 边界，故定义在 kernel；唯一实现在 domains/player/init）。
export type InitSource = 'user' | 'restore' | 'step' | 'auto' | 'selection' | 'retry' | 'recovery' | 'promote-fallback' | 'deadlock';
export type PresentationAspect = 'contain' | 'cover';

// UI 手势 token 的结构快照（与 pages/player/FullscreenCtl.TargetToken 结构等价，避免 domain → pages 依赖）。
export interface InitTargetToken {
  uri: string;
  gen: number;
}

export interface PresentationIntent {
  fullscreenStage: boolean;    // 舞台语义（fc.active，含横屏）
  fullscreenRender: boolean;   // 域语义（fc.active && portrait，Surface 优先级）
  aspect: PresentationAspect;
  panelIndex: number;          // -1 = 未检测/单面板
  panels: PanelRect[];         // 长度 ≥1，缺省整帧
  panelRevision: number;       // 检测结果版本（换 uri 才推进）
}

// 不可变输入值对象：不含 slot / lease / handle / gen。
export interface InitIntent {
  targetToken: InitTargetToken;
  mediaItem: MediaItem;
  uri: string;
  deadlineMs: number;
  source: InitSource;
  presentation: PresentationIntent;
  hasPreloadCandidate: boolean;
}

// v390：预加载资产接管视图（方案 A）——UI 只读；判定唯一实现在域内。
export type TakeoverVerdict = 'ready' | 'arming' | 'idle' | 'lost';

export interface PreloadTakeoverView {
  verdict: TakeoverVerdict;
  preloadGen: number;        // 预加载任务代次
  surfaceGen: number;        // job.gen（SurfaceRegistry 归属代次）
  backendState: string;      // 'idle' | 'initialized' | 'prepared' | 'playing' | 'paused' | 'completed' | 'error' | 'released'
  prepareAgeMs: number;      // 预加载槽本次 prepare 已耗时；未知 = -1
  hasPageSurface: boolean;   // 预加载槽 registry 是否有当前 commit gen 的 (uri, gen) 页面
  leaseReady: boolean;       // holder 已持有或该流无 lease
  fenced: boolean;           // 该 (uri, gen) 已被 fence
  reason: string;            // 诊断：'ok' | 'no_job' | 'failed' | 'fenced' | 'intent_moved' | ...
}
```
