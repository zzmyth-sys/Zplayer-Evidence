# OPS · SFF 守护脚本测试期改造归档（V402-056 停服开关）

> 性质：运维变更归档（只动启动/守护，未动 SFF 业务代码）
> 对象：`<SFF_HOST>` 上 `C:\ZP-Accelerator\run-service.cmd`
> 变更时间：2026-10-27 01:16–01:18（设备/主机本地时间）
> 结论：测试期"干净停服 ≥2 min"与"一键恢复"均已实测通过；本改造属临时运维手段，生产必须以版本化配置取代

## 1. 为什么要改

- 原脚本是 `:run` 死循环：`uvicorn` 退出后 `restart in 5s` 立刻自拉起。测试中"停 SFF"只成立约 5 s，导致：
  - 降级/停服类测试窗口不可控；
  - 上一轮实测把一次 5 s 中断误判为"停服"，产生过一条无效 B 侧数据（见 `T-15.md` §2）。
- 计划任务方案（`schtasks /disable`）不可用：`schtasks /query /tn ZPAccelV402` 返回"任务不存在"，本机只有一条孤儿 `:run` 循环在自拉起。

## 2. 改了什么（仅 `run-service.cmd`）

| 项 | 变更 |
|---|---|
| 双开关 | ① 环境变量 `ZPACCEL_NORESTART=1`；② 标志文件 `C:\ZP-Accelerator\ZPACCEL_NOSTART` 存在。两者任一成立，循环打印原因后 `goto end` 退出，**不再拉起 uvicorn** |
| 检查点 | 放在 `:run` 循环顶部，即每次 uvicorn 退出、`ping` 等待结束后重新判断 → 停服后不会被自拉起 |
| 日志分离 | 停服/退出事件写 `C:\ProgramData\ZP-Accelerator\logs\svc-lifecycle.log`；运行日志仍为 `svc-out.log`/`svc-err.log`。原因：uvicorn 运行期持有 `svc-out.log` 句柄，外部追加会报 "file is being used by another process" |
| 备份 | `C:\ZP-Accelerator\run-service.cmd.bak-20261027-011642`（原文件 815 B） |
| 其他 | 增加 `mkdir "%LOG%"` 兜底；将 `exited ... restart in 5s` 改为 `check stop flags` 的措辞；其余 env/端口/参数一字未改 |

脚本新增逻辑（摘要，非全文）：

```text
:run
if "%ZPACCEL_NORESTART%"=="1"       → log "loop stop: ZPACCEL_NORESTART=1"       → goto end
if exist "%~dp0ZPACCEL_NOSTART"     → log "loop stop: ZPACCEL_NOSTART flag present" → goto end
...
:end
log "loop exited"
```

## 3. 测试期停服（清理后必须执行）

```powershell
# 1) 置标志（此后循环不再自拉起）
Set-Content C:\ZP-Accelerator\ZPACCEL_NOSTART -Value stop -Encoding ascii
# 2) 停止服务进程（正常"停服务"动作，非掩盖式手杀）
Stop-Process <uvicorn 的 python 进程>
# 3) 确认：health 连续无响应 ≥2 min
for ($i=0; $i -lt 12; $i++) { curl.exe -s -m 3 -o NUL -w "%{http_code}`n" http://127.0.0.1:8612/health; Start-Sleep 10 }
```

## 4. 恢复生产

```powershell
# 方案一（推荐，回到生产循环）
Remove-Item C:\ZP-Accelerator\ZPACCEL_NOSTART -Force
Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='cmd /c C:\ZP-Accelerator\run-service.cmd'}

# 方案二（彻底回滚脚本）
Copy-Item C:\ZP-Accelerator\run-service.cmd.bak-20261027-011642 C:\ZP-Accelerator\run-service.cmd -Force
# 然后按原方式启动
```

## 5. 实测验证（本次）

| 步骤 | 观测 |
|---|---|
| 置标志 + 停 uvicorn | `01:18:09.05 loop stop: ZPACCEL_NOSTART flag present`（`svc-lifecycle.log`） |
| 停服确认 | `poll t=10s..120s code=000`（12/12 全 000，连续 **120 s** 无响应）；`py_left=0` |
| 环境变量路径 | `ZPACCEL_NORESTART=1` → `exit=0`；`01:20:47.26 loop stop: ZPACCEL_NORESTART=1`；health 仍 000（未起服务） |
| 恢复 | 删标志 + 启动后 `restore t=7s/14s/21s code=200`（3/3）；`uvicorn_procs=2` |
| 当前状态 | SFF 运行中（health 200），标志文件已删 |

## 6. 归档声明（硬约束）

1. **本改造仅限测试期**：目的是获得可重复的"停服窗口"，不是长期部署形态。
2. **`run-service.cmd` 必须纳入版本控制**（当前它不在任何仓库里，只存在于 `<SFF_HOST>` 磁盘）。建议落 `ZP-Accelerator/deploy/run-service.cmd`，与 `deploy/setup-service.ps1` 同目录，由部署脚本安装。
3. **生产部署不得手工改该文件**：任何开关（是否自拉起、重启间隔、日志路径）应走配置项/服务参数，禁止 SSH 上去改脚本。
4. 本次未改 SFF 业务代码、未改端口/环境变量默认值、未新增常驻进程。
5. 遗留风险：若测试中断未删标志文件，服务重启后会**静默不自拉起**。排障第一步检查 `ZPACCEL_NOSTART` 与 `svc-lifecycle.log`。
