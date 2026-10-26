# T-14-EXCERPT · V402-049R 裁判取件（T-14 三区间原文）

> 性质：只读摘录；内容仅为 T-14 原文，不改写、不压缩；不含代码。
> 来源：`Zplayer-Evidence/DOC/V402/T-14.md`（commit `657f768`）。

## 索引

| 段 | 行号 | 一句话 |
|---|---|---|
| §A | 14–27 | 0020 的文件级 877 行构成与“单独应用、勿叠 0019”关系 |
| §B | 29–49 | 15–25 人日的逐项人日表与合计依据 |
| §C | 51–61 | 2.0.13 vs 2.0.11 仅到版本/lock 风险条目；无展开分析 |

## §A 0020 的 877 行构成

来自 T-14 §2 行 49–58（引 51–58）：

> | patch | 文件 / 行数 | 类型 | 作用与合并含义 |
> |---|---|---|---|
> | `0019-smb2-ohos-n8.0.patch` | configure +5、Makefile +1、libsmb2.c +415、protocols.c +1；4 文件 +422 | **新增** | 新增 libsmb2 后端的 `smb` URLProtocol；上游 0019 与 debugly n8.1.x 同一份，仅 2 处 OHOS 适配（include 路径、协议名 `smb2`→`smb`） |
> | `0020-smb2-async-prefetch-ring.patch` | configure +5、Makefile +1、libsmb2.c +868、protocols.c +1、seek.c +3/-1；5 文件 +877/-1 | **整合/覆盖** | 以新文件形式给出最终 868 行 `libsmb2.c`（含 16 MB 环/4 MB 预取/块缓存），并加 `seek.c` 的 `smb` 白名单；**应单独应用，不要再叠 0019**（否则重复/冲突） |
> | `0021-ijkplayer-har-smb2-link.patch` | build-profile×2、CMakeLists×2；4 文件 +12/-6 | **修改构建** | 把 `libsmb2.so.1` 作为 IMPORTED 链入 `libijkplayer`/HAR；`arm64-v8a` only |
>
> - 与主线上游关系：**新增**（0019/0020 的 protocol 与 `libsmb2.c`）、**修改**（configure/Makefile/protocols/seek/构建配置）；主线没有 FFmpeg 源码可“覆盖”，只能重编。
> - 0019 的两个版本都在 DEMO `native/patches/`；合并以 OHOS 版/0020 为准，0019 作来源参考。

**该段说明**：T-14 只给出文件级行数与“0020 独立应用”的结论，未把 877 行进一步拆成“整合/新增/hack”三类；本摘录不补造该分类。

## §B 15–25 人日的计算依据

来自 T-14 §7 行 110–128（引 112–128）：

> | # | 层 | 项 | 类 | 预估人日 | 主要风险 |
> |---:|---|---|---|---:|---|
> | 1 | libsmb2 | 复用 4.0.0 动态库 | **A** | 0.1–0.5 | 两个模块重复打包同名 `.so.1` |
> | 2 | FFmpeg 补丁 | 0019 protocol（若单独用） | **B** | 1–2 | FFmpeg base/configure 漂移；与 0020 重复 |
> | 3 | FFmpeg 补丁 | 0020 预取环 + seek 白名单 | **C** | 3–5 | 本地自持读模型正确性、16 MB 环、seek 语义 |
> | 4 | 构建 | 0021 HAR link libsmb2 | **B** | 0.5–1 | HAR 打包、arm64 only、重复 `.so` |
> | 5 | FFmpeg 构建 | 重编（ohcodec+libsmb2+openssl/shared） | **B** | 2–3 | 旧 CI 工具链/SDK12 → SDK24；x86_64 缺失 |
> | 6 | IJK 构建 | 重链 `libijkplayer`/`libijksdl` | **B** | 1–2 | `libijksdl` 体积差异、SURFACE 回归 |
> | 7 | HAR | 版本/结构/ABI 适配（2.0.13→2.0.11 线） | **C** | 1–2 | oh-package lock、module.json SDK、x86_64 |
> | 8 | 主项目 A 路径 | ports/flows/CommandCenter 重写 | **C** | 3–5 | DEMO 代码不能直接搬；latest-wins/降级边界 |
> | 9 | A/B 共存 | libsmb2 去重 + 并发 | **C** | 1–2 | 同名 SONAME、双 context 未验证 |
> | 10 | 回归 | V401 SURFACE/双槽/CPU | **C** | 2–3 | `libijksdl`/libavcodec 重建导致回退 |
> | 11 | ETS/NAPI | 直接复用 | **A** | 0 | 版本/包名变化 |
>
> - 合计约 **15–25 人日**（含回归；估算，未实施）。
> - **A 2 / B 4 / C 5 / D 0**（当前 SMB 合并面）；若未来改 `ff_ffplay.c`，新增 1 项 D。
> - 最大风险：**0020 读模型 + 主项目 A 路径重写**；ABI 不是最大风险。

## §C 2.0.13 vs 2.0.11 的 lock/并存/替换/升级风险

来自 T-14 §5 行 79–98（引 93）、§7 行 110–128（引 120）、§8.1 行 132–138（引 135）：

> | `oh-package.json5` | version `2.0.11` | version `2.0.13`，含 metadata/nativeComponents，compatibleSdkVersion 24 | 版本/元数据不同 |

> | 7 | HAR | 版本/结构/ABI 适配（2.0.13→2.0.11 线） | **C** | 1–2 | oh-package lock、module.json SDK、x86_64 |

> 2. **IJK/HAR**：同源重链 `libijkplayer`/`libijksdl`，加 `libsmb2.so.1` 依赖；本地化 HAR，保持 ETS/NAPI 不变；包版本/`oh-package`/`module.json` 对齐主项目。

**该段无展开**：T-14 只登记了版本差异与 “oh-package lock” 风险条目，未展开 lock 冲突细节、2.0.13 与 2.0.11 的并存/替换策略、升级覆盖风险；本摘录不补写。
