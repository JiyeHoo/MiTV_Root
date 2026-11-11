# 07 Root 管理 App（rtu root manager）

> 接手成果：在持久化 root（内核补丁 + setuid su）之上，补齐 SuperSU 式的**权限管理层**：
> 电视遥控器友好的 UI，root 总开关 + 每软件 allow/deny/ask 授权 + 提权日志 + 内核态一键修复。
> 本文档自包含：架构 / 使用 / 部署 / 回滚 / 重建。

## 1. 组成

| 产物 | 位置 | 说明 |
|---|---|---|
| 托管 su | `/system/xbin/su`（6755，armv7 静态） | 源码 `tools/src/rtu_su.c`，构建产物 `tools/bin/rtu_su`。**替换**了原 mini_su（备份在 `/system/xbin/su.mini.bak`） |
| 管理 App | `com.tvroot.manager` | 源码 `app/src/`，APK `app/build/TvRootManager.apk` |
| 策略库 | `/data/rtu/` | root 安装创建，结构见下 |
| 部署脚本 | `deploy/deploy.sh` | 一键部署（含内核态恢复） |
| roothelper 载荷 | `deploy/roothelper.sh` + `deploy/install_root_cmd.sh` | 安装动作本体（coredump 管道执行） |
| 构建脚本 | `tools/build_all.sh` | zig 编 su + aapt/javac/dx/apksigner 出 APK |

## 2. 工作原理

### 2.1 su 的决策流程（每次有人调 `su`）

```
调用者 uid ──┬─ 已是 root                 → 直接放行
             ├─ == 管理器 uid             → 直接放行（App 自检/修复用）
             ├─ /data/rtu/switch != "1"   → 一律拒绝（总开关）
             └─ 查策略 /data/rtu/policy/u<uid>
                  ├─ allow → 放行
                  ├─ deny  → 拒绝
                  └─ ask/未配置
                       ├─ 写 /data/rtu/req/<uid>_<pid>.req（0644）
                       ├─ am start 唤起 App 的 RequestActivity（TV 弹窗）
                       └─ 轮询 /data/rtu/resp/<uid>_<pid>.resp，60 秒超时=拒绝
放行 = setuid(0) + capset 全量（受 bounding set 限制）+ exec /system/bin/sh
```

所有决策写 `/data/rtu/log`（时间 | 决策 | uid | 包名 | 命令）。

**安全约束**：policy/resp 文件属主必须是管理器 uid（从 `/data/system/packages.list`
解析）或 root，防止第三方 app 伪造授权。管理器自身与 root 不受限。

### 2.2 策略库结构

```
/data/rtu/
├── switch            0666  "1"=总开关开（App 直接改写，不需要 root）
├── log               0666  提权日志（App 读取展示）
├── policy/           0777  u<uid> 文件，内容 allow/deny/ask，0644
├── req/              0777  su 写的授权请求（uid_pid.req）
└── resp/             0777  管理器写的决策（uid_pid.resp）
```

目录 0777 是刻意的：su（可能只有 0xc0 caps，见 03-踩坑清单 C2）不需要任何
额外能力就能读写。属主校验承担防伪造职责。

### 2.3 与 root 基础设施的关系

- **不依赖 core_pattern/slots**：su 提权只靠内核补丁保证的 SELinux Permissive
  + setuid 位，重启即用（`/system` 是持久的）。
- **依赖 slots 的只有"修复内核态"按钮**：执行 `gpu_arb_write 1f2b790 …`
  （slots→memcpy）+ `kfunc246 zero`（selinux_enforcing）——即原 rootify.sh 的
  逻辑，用于 rootify 钩子失效时的手动兜底。
- **roothelper 通道保留**：core_pattern 劫持仍是 /system 写入的正规通道
  （deploy.sh 第 5 步就是用它装 su）。adb shell 的 su 仍然只有 0xc0 caps，
  写 /system 一律走 root_cmd 机制（02-复刻手册 阶段 7）。

## 3. App 功能（TV UI，遥控器操作）

- **首页**：Root 总开关（一键开/关，立即生效）、su/SELinux/策略状态、
  「修复内核态」按钮、授权管理与日志入口。
- **软件 Root 授权管理**：全部已装软件 + ADB Shell 列表，点击选择
  永久允许 / 每次询问 / 永久拒绝。已配置的排前面。
- **授权弹窗**（su 触发）：永久允许 / 仅本次允许 / 仅本次拒绝 / 永久拒绝，
  60 秒无操作自动拒绝。
- **提权日志**：时间、决策、uid、包名、命令。

默认策略：部署脚本预置 `u2000=allow`（adb 工作流不受影响）；其余软件未配置
= 每次询问；管理器自身 = 永远放行。

## 4. 部署 / 卸载 / 回滚

```bash
# 一键部署（电视开机、与 PC 同网段；网络调试已开）
deploy/deploy.sh                 # 默认 192.168.2.105:5555，可传别的目标

# 单独重装 APK（改了 App 后）
tools/platform-tools/adb.exe -s 192.168.2.105:5555 install -r app/build/TvRootManager.apk

# 回滚到 mini_su（无管理界面，直接放行）
adb shell "ulimit -c unlimited; kill -SEGV \$$"   # 先让 core_pattern 生效（见 deploy.sh 3~5 步）
#   并把 deploy/install_root_cmd.sh 改为: cp /system/xbin/su.mini.bak /system/xbin/su
# 彻底卸载 App:
adb shell pm uninstall com.tvroot.manager
```

App 本身可随意卸载重装（策略库在 /data/rtu，不受影响；但卸载后 policy/resp
属主校验会找不到管理器 uid → 按 root 属主放行/继续工作）。

## 5. 重建（改代码后）

```bash
# 需要一次性就位的工具链（下载解压）
tools/zig/            zig-windows-x86_64-0.9.1
tools/jdk8/           Temurin JDK 8 (Windows x64)
tools/bt/android-9/   build-tools r28.0.3
tools/platform-23/    android.jar (platform-23)
tools/platform-tools/ adb

tools/build_all.sh    # 产 tools/bin/rtu_su + app/build/TvRootManager.apk
deploy/deploy.sh      # 部署
```

源码：`tools/src/rtu_su.c`（su）、`app/src/com/tvroot/manager/*.java`（App，
纯代码 UI 无 XML 布局）、`app/AndroidManifest.xml`。

## 6. 已知边界（含 2026-11-11 实机验证结论）

- adb shell 里 `su` 只有 0xc0 caps（bounding set 限制，无法绕过）——读
  /proc、cat、reboot 没问题；写 /data/local/tmp 之外的文件用 root_cmd 通道。
- App 进程树 bounding set 是全量，App 内的 su 能力完整（可用 CapEff 全开）。
- 总开关关闭只是"su 层面拒绝"；内核补丁、SELinux Permissive、core_pattern
  不受影响——这是特性（可随时一键恢复），不是漏洞面收窄手段。
- 设备无公开 OTA，/system 改动前确认 `images/system_backup.img` 完整
  （1073741824 字节），见 06-回滚与恢复.md。

### 实机部署踩坑补充（小米电视 4A，2026-11-11 实测）

1. **adb 安装 APK 会弹电视端「安装确认」**：pm install 停在 `pkg: ...` 不动 =
   电视屏幕在等遥控器确认「确定」。勾选「应用到全部」后本机后续安装免确认。
   远程无人值守可走 root pm（root_cmd 里已导出 ANDROID_DATA/ANDROID_ROOT，
   否则 pm 直接 Aborted exit=134）。
2. **策略/日志文件属主必须是管理器 uid**：App 才能改写策略。install/
   upgrade_root_cmd.sh 里有 `chown $(stat -c %u /data/data/com.tvroot.manager)`
   ——**重装 App 会换 uid，之后必须重跑 upgrade_root_cmd.sh**。
3. **adb 会话断开会杀掉设备侧 su**（SIGHUP）：远程触发 ask 流程测试时用
   `sh -c 'trap "" HUP; su ... &'` 或保持会话。孤儿 su 的 getppid()=1，
   caller_pkg 会兜底显示 shell(adb)。
4. **zig 版本**：0.9.1 在本机构建 musl CRT 失败（unable to build C object），
   用 **0.13.0**（tools/zig13/）。
5. dexopt 很慢，pm install 至少给 3~5 分钟；`adb install -r` 换签名后报
   INSTALL_FAILED_UPDATE_INCOMPATIBLE，先 uninstall 再 install。

