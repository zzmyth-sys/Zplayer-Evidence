# T-01-REALDEVICE · V402-049 A 线 T-01 收尾（真机原始 hilog 片段）

> 测试 commit：主项目 `d67e179`（HAP `zplayer-v402-036-baseline.hap`；T-12 §13.1。日志本身不内嵌 commit）
> 设备：MLR-AL00 / API 26（真机，序列号不入库）
> SFF：夹具 commit `f789260`；后端 8613、夹具 8612
> 说明：仅将 IP、HLS id、片名、端口/大小替换为 `<...>`；其余字段逐字保留。

## 原始 hilog（一次 504 → 降级完成）

```text
10-26 22:57:33.588 17682 17682 E A00000/com.example.zplayer/ZPLAYER/IjkPlayerBackend: ijk error what=-10000 extra=0
10-26 22:57:33.589 17682 17682 W A00000/com.example.zplayer/ZPLAYER/IjkPlayerBackend: ijk transport fail url=http://<SFF_HOST>:8612/hls/<HLS_ID>/index.m3u8 code=-10000/0
10-26 22:57:33.589 17682 17682 W A00000/com.example.zplayer/ZPLAYER/PlayerBootstrapper: STREAM_FAIL stage=hls-segment code=-10000 msg=ijk:-10000/0
10-26 22:57:33.589 17682 17682 W A00000/com.example.zplayer/ZPLAYER/HlsDegrade: HLS_STREAM_FAIL reason=stream-fail code=-10000 uri=<SAMPLE>
10-26 22:57:33.603 17682 17682 I A00000/com.example.zplayer/ZPLAYER/Lease: LEASE_CREATE leaseId=L1 streamId=s1 port=<PORT> url=http://127.0.0.1:<PORT>/video.mp4
10-26 22:57:33.607 17682 17682 I A00000/com.example.zplayer/ZPLAYER/SmbHttpProxy: listening 127.0.0.1:<PORT> size=<SIZE>
10-26 22:57:33.607 17682 17682 I A00000/com.example.zplayer/ZPLAYER/SmbOpenOrchestrator: open path=<SAMPLE> url=http://127.0.0.1:<PORT>/video.mp4 size=<SIZE>
10-26 22:57:33.855 17682 17682 I A00000/com.example.zplayer/ZPLAYER/SmbHttpProxy: served start=<OFF> declared=<LEN> sent=<LEN>
10-26 22:57:33.930 17682 17682 I A00000/com.example.zplayer/ZPLAYER/Lease: LEASE_HOLDER_ADD leaseId=L1 holder=slot0 holders=1
10-26 22:57:33.934 17682 17682 W A00000/com.example.zplayer/ZPLAYER/SourceDowngrade: DOWNGRADE_OK trigger=stream-fail uri=<SAMPLE>
10-26 22:57:34.193   817   817 I C02105/intell_voice_service/TriggerHelper: [OnRendererStateChange:1217]: state change, usage:10, isPlaying:1
```

## 逐项对照

| 要求 | 证据 | 判定 |
|---|---|---|
| `isHlsTransportFailure` 判定 true | 22:57:33.589 `ijk transport fail ... code=-10000/0`；该行位于 `IjkPlayerBackend.ets:311-318` 的 `isHlsTransportFailure(...)` true 分支内 | **间接成立**；hilog 未打印函数返回值，只有分支日志 |
| 出现 `streamFailure`，无 `fatalReport` | 22:57:33.589 `STREAM_FAIL` + `HLS_STREAM_FAIL`；同窗口 `fatal/Fatal/fatalReport` 检索 0 条 | **成立** |
| `SourceDowngradeCtl.run` 新 gen + `avoid:['SFF']` | 22:57:33.603 `LEASE_CREATE` + 33.607 `SmbOpenOrchestrator open` + 33.934 `DOWNGRADE_OK`；代码 `SourceDowngradeCtl.ets:143-145,193` 先 `beginGen` 再 `serialPrepare(... ['SFF'])` | **代码成立，hilog 未打印 avoid/gen** |
| 新 backend 落到 `PlayerState.Playing` | 22:57:33.855 代理 `served` + 34.193 音频 `isPlaying:1`；**hilog 无 `PlayerState.Playing` 行**（`PROBE_LOGS=false`） | **未取得直接证据** |

## 结论与缺口

- 降级链已走完：`STREAM_FAIL → SMB open → DOWNGRADE_OK`，代理持续 `served`，未见 Fatal；符合“可解码素材降级成功”的行为。
- **不能作为 T-01 四项全满足的关闭证据**：① `isHlsTransportFailure` 只有分支日志；② `avoid/gen` 未入 hilog；③ `PlayerState.Playing` 无运行期日志。
- 若要把三项变成直接证据，需要 `PROBE_LOGS=true` 的测试构建补 `[PROBE-STORE] state=Playing` / `[PROBE-GEN] gen=`，或把 `SourceDowngradeCtl` 的 gen/avoid 打进 hilog；本轮未改主仓代码，故未补。
- 本轮设备已恢复主项目 HAP；未留在 DEMO。
