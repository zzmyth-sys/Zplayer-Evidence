# DEMO-SEEK · V402-049 B 线 SeekArbiter 真机拖拽记录

> DEMO commit：`650b310`（时钟注入 + 纯逻辑测试；夹具 `c08a324`）
> 设备：MLR-AL00 / API 26（真机，序列号不入库）
> 素材：4K SMB 直读（DEMO 默认 4K 样本，代号 `<4K_SAMPLE>`）
> 说明：仅掩码 host/share/路径；hilog 字段逐字保留。

## 原始 hilog（启动夹具，prepare 阶段失败）

```text
10-27 00:42:21.845 57005 57005 I A00000/com.example.zplayer/SMBPOC: [SMBPOC] create mode=direct host=<SMB_HOST> share=<SHARE> path=<SAMPLE>
10-27 00:42:22.012 57005 57005 I C03925/com.example.zplayer/AceRouter: [(100000:100000:scope)] Page router manager is creating page[1]: url: fixture/DragHarnessPage path: fixture/DragHarnessPage.js, recoverable: yes, namedRouter: no
10-27 00:42:22.033 57005 57005 I A00000/com.example.zplayer/DRAGHARNESS: harness_boot mode=direct path=<SAMPLE>
10-27 00:42:22.083 57005 57005 I A00000/com.example.zplayer/DRAGHARNESS: prepare_called
10-27 00:42:22.154 57005 57005 I A00000/com.example.zplayer/DRAGHARNESS: error what=-10000 extra=0
```

第二次改用 `--ps share <SHARE_ALT>`，同样在 `prepare_called` 后 44 ms 报 `error what=-10000 extra=0`；无 `fire` / `seek_complete` / `frame` / `plan_stall` 行。

## 结果

| 项 | 结果 |
|---|---|
| 有效快拖样本 N | **0**（未进入 prepared） |
| 每次 Seek 完成耗时 | **无数据** |
| 黑帧 / 回跳 | **无数据** |
| CPU | **无数据** |
| 结论行“CPU __%、播放正常、拖动不卡” | **不写**（未达标/无证据） |

## 阻塞原因

- DEMO `PocCfg.pass` 为空；本次未提供 SMB 凭据，主项目已存凭据未解码/未复用。
- `error -10000` 为 IJK 通用错误；无 `libsmb2` 细错误日志，不能区分认证失败与 share/路径错误。
- 未做任何伪造/转述；未把 host 预检 9/9 当作真机数据。
- 设备已恢复主项目 HAP（`zplayer-v402-036-baseline.hap`），未留在 DEMO。

## 参考（非真机证据）

- `SeekArbiter` 纯逻辑 Node 预检：9/9（T-13 §Q11）；模拟器未运行，**不是**真机拖拽证据。
