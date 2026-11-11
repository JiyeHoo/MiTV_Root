# 小米电视 4A（matrix）Mali Utgard 提权项目 · 交接总入口

> 本文是接手入口。**先读完这一页**，再按需展开 `01`~`06`。
> 历史时间线记录仍保留在 [`RESEARCH_NOTES.md`](../RESEARCH_NOTES.md)（按阶段追加，内容与本文重叠，仅作归档）。

---

## 一页速览

| 项 | 内容 |
|---|---|
| 目标 | 给小米电视 4A（`matrix`，Amlogic S905L）拿到 **root**；备选方案是自定义桌面接管（用户已明确「更倾向 root」） |
| 方案来源 | [`Yeuoly/mitv4-utgard-lpe`](https://github.com/Yeuoly/mitv4-utgard-lpe)（MiTV4 实测 PoC，利用 Mali Utgard 驱动未校验物理地址） |
| 设备 | 192.168.2.105:5555，MiTV4A / matrix / Android 6.0.1 / `6.0.1.1275`（显示 `8.8.20.1275`）/ 内核 `3.14.29-g4f3e19b`（2018-08-20） |
| 已达成 | **任意物理写原语（128 字节/次）实机验证通过**；驱动漏洞、ioctl ABI、job 布局、崩溃现场利用全部逆向落地 |
| 当前卡点 | 缺 **2 个固件常量**：`kallsyms_lookup_name` 虚地址、`sys_call_table[244]` 物理地址。取常量的两条路都堵着：固件包已被 CDN 清理；用 GPU 造「任意读」卡在**这一代 DDK 的 shader 指令编码**（Lima 公开 ISA 不适用） |
| 设备状态 | 网络可达、5555 端口开放，但 adb 传输层为 `offline`（需电视端确认/唤醒，见 [04 · adb 的坑](04-复现与操作手册.md)） |
| 下一步 | 见 [05-路线与待办](05-路线与待办.md)：P0 修复 adb → P1 判定是否走「读原语」或「再找固件」→ P2 执行提权链 |

---

## 文档地图

| 文档 | 回答什么问题 |
|---|---|
| [01-项目总览与结论](01-项目总览.md) | 这项目要干什么、做到哪一步、结论是什么、还差什么 |
| [02-技术事实与实验矩阵](02-技术事实.md) | 已经**证实**的技术事实（设备/驱动/写原语/job 布局/DDK 逆向）、**做过的实验及其结论**、已排除的路径、未解问题 |
| [03-产物清单](03-产物清单.md) | 工作区里每个文件是什么、设备上推了什么、哪些脚本还有用 |
| [04-复现与操作手册](04-复现与操作手册.md) | adb 怎么连（含已知坑）、怎么采证据、怎么写/读、固件怎么验、安全红线 |
| [05-路线与待办](05-路线与待办.md) | 三条路线怎么选、替代方案（不用固件包行不行）、按优先级排的待办与验收标准 |
| [06-会话与决策日志](06-会话与决策日志.md) | 上一段会话发生了什么、5 轮对话的诉求演进、为什么中断、有哪些悬空问题 |

---

## 30 秒上手

```powershell
$ws = "$env:USERPROFILE\Documents\deepseek-harness\default-workspace"
$adb = "$ws\tools\platform-tools\adb.exe"

# 1) adb 必须重定向 TEMP，否则 daemon 起不来（C:\...\Temp\adb.log 权限拒绝）
$env:TEMP = "$ws\tools\tmp"; $env:TMP = "$ws\tools\tmp"
& $adb start-server
& $adb connect 192.168.2.105:5555
& $adb devices -l          # 期望：192.168.2.105:5555  device
```

关键文件（相对 `$ws`）：

- 漏洞利用源码：`mitv4-utgard-lpe-main/src/{gpu_arb_write,gp_read,pp_read,kfunc}.c`
- 编译产物（未 strip，可直接推设备）：`mitv4-utgard-lpe-main/{gpu_arb_write,gp_read,pp_read,kfunc}`
- 驱动/用户态样本：`mali.ko`、`libGLES_mali.so`
- 现场证据：`dmesg.txt`、`dmesg2.txt`、`logcat.txt`
- 固件解析产物：`fw/`（含 L49M5-AZ eMMC 提取结果）

---

## 红线（每次动手前看一遍）

1. **`panic_on_oops=1`**：任何一次非法内核写入 = 立即 panic + watchdog 5 秒重启。重启本身无损坏（已多次实测），但会中断现场。
2. **只对「已离线验证过」的物理地址写入**。历史事故：向物理地址 0 写 128 字节导致重启（当时是哑参数）。
3. **全链成功前不碰 `/system`**；roothelper 落地后回滚方式是 `mount -o rw,remount /system && rm /system/xbin/su` + `setprop persist.adb.tcp.port ""`。
4. 校准模式（`gpu_arb_write 0 0`，phys=0）只写自己申请的 ion 页，**可安全反复跑**；真实地址写入前先跑它。

> **注意**：本目录（archive/）是 root 攻克**之前**的阶段文档（2026-11-09/10 前期研究），
> 描述的是当时的卡点与计划，其中"待办/路线"均已由后来的方案实现或证伪。
> 当前状态与权威做法一律以 `../01-原理详解.md` ~ `../06-回滚与恢复.md` 为准。
