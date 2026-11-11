> ⚠️ **归档说明**：本文件是**按阶段追加上去的历史记录**（第一~第四阶段），保留原始时间线与当时的表述，不再维护。
> 最新、结构化、可检索的整理版请看 **[`docs/README.md`](docs/README.md)**（总览 / 技术事实 / 产物 / 操作手册 / 路线待办 / 会话日志）。
> 两处若有出入，以 `docs/` 为准。

---

# MiTV4A 43寸 (matrix) Mali Utgard LPE Root 可行性调研报告【第二阶段更新】

## ★★ 第三阶段：L49M5-AZ EMMC dump 分析结论

用户提供 `Xiaomi L49M5-AZ TPD.T962.PC791 EMMC.rar`（2.85GB，含 4.15GB eMMC dump）。

**结论：同平台、不同构建，不能直接用于本任务。**

### 关键事实
- **L49M5-AZ 就是 matrix 平台**（`ro.product.device=matrix`，板 TPD.T962.PC791，TXL/S905L 家族）——与我们的 4A 同设备代码！
- eMMC 内含 matrix 固件演进：`7.4.24.465`（2017-04 出厂）→ `7.9.14.848`（2017-09）→ **`8.3.29.1191`**（2018-03，设备实际运行版）。
- boot 分区内核（@0x19000000/0x23e00000，ANDROID! bootimg，gzip）：`3.14.29-g879222c`，构建 **2018-03-29 11:49**，同一构建服务器（c3-mitv-bsp-build03.bj）。
- 符号数 **83951 vs 我们设备 91624** → 确认非同一构建，地址不可用。
- 镜像中无我们的构建痕迹：搜 `g4f3e19b` = 0 命中；`Mon Aug 20 16:52:56 CST 2018` = 0 命中；1275 相关固件字符串 = 0 命中（AZ 停在 1191，未收到 1275 推送）。
- CDN 探测：1191 全量包、所有指向 1275 的增量包 = 全部 404。

### 管线已打通（下次直接用）
```
UnRAR 解包 → eMMC BIN 扫 'ANDROID!' 魔数（fw\scan_bin.py）→ 解析 bootimg 头（extract_boot.py）
→ gzip 解压 → arm64 Image → vmlinux-to-elf（基址 ffffffc001080000 与目标一致）
→ kallsyms 符号数与设备 /proc/kallsyms 行数比对（91624 为匹配值）
→ 需要的指纹：b'g4f3e19b' + b'Mon Aug 20 16:52:56 CST 2018' 命中即为目标内核
```

### 下一步给用户的搜索指引
需要 **matrix 设备运行 1275（最终版）** 的 eMMC dump 或 OTA 包。搜索关键词建议：
- `L49M5-AA EMMC` / `L43M5-AA EMMC` / `L40M5-AA` / `L55M5-AA`（AA 系 = matrix 的面板变体；注意 AZ ≠ AA）
- `小米电视4A 8.8.20.1275` / `matrix 6.0.1.1275`
- 询问提供这个 AZ dump 的卖家/帖子是否有 **AA 系列** 或更高版本（≥8.8.20）的 dump——维修 dump 收藏通常是整套机型
- 判定方法（我这边 2 分钟即可验证）：镜像里搜 `g4f3e19b` 或 `Mon Aug 20 16:52:56`

### 设备本机信息补充（本次确认）
- 用户电视 cmdline: `mi.panel_size=49`、广告 API screenSize=49 → 实际为 **49 寸**（L49M5-AA 可能性大），43寸 是此前文档的说法
- 电视 adb 192.168.2.105:5555（待机时需重新 connect）

---

# 第二阶段报告（背景，已验证部分保持不变）

调研对象: https://github.com/Yeuoly/mitv4-utgard-lpe
设备: 小米电视 4A 43" / MiTV4A (matrix), 192.168.2.105, adb 已授权
**当前状态: 写原语已完全验证（含 WB0 回写到自有内存的完整管线）；读原语卡在 DDK 时代的 shader/命令流 ABI 细节上。**

## ★ 第二阶段新增成果（本阶段全部实机验证）

### 1. 写原语管线完整打通
- `gpu_arb_write` 校准通过（128 字节精确写）后，进一步验证了 **WB0 回写目标可以是我们自己的内存**：
  `pp_read clear` 模式 → 魔数 0xDDCCBBAA00998877 精确落入 OUT_VA（carveout 页 + external 绑定）。
- 调试经验（已固化到代码注释）：
  - PP job 必须 `user[328]=1`，否则驱动给 PP reg 0x0 写入 0xFFF00000 垃圾值 → UNKNOWN_ERR。
  - cmd 缓冲**绝不能全零**：必须有含 0xBC000000 终止束的 shader，否则 PP 跑飞立即报错。
  - WB0 目标页要用 CARVEOUT+external 绑定（dma_buf 绑定页做 WB 目标会失败）。
  - MEM_BIND backend: dma_buf=0x200, external=0x800（本机驱动按 flags&0x3f00 分派）。
- 结论: **对内核的最终两步写（slots→KLN、slots→memcpy）风险已降到最低**——所有机制都已单独验证。

### 2. 内核侧 job 布局完全逆向（来自 mali.ko）
- PP job (408B): [20..112]=frame_registers[23]→PP regs 0x0..0x58；[168..220]=WB0→PP 0x100..0x12C；[400..408]=通知指针。
- GP job (104B): ioctl=0xC0688500；[20..44]=frame[6]→GP regs 0x0..0x14（VSCL_START/END, PLBCL_START/END, PLBU_ALLOC_START/END）；perf=-1 跳过；**plbcl_start==end 时 GP 只跑 VS**。
- PP 寄存器: 0x0=FRAME(PLBU数组), 0x4=RSW指针, 0x30/0x34=fragment stack, 0x100/200/300=WB0/1/2（select/addr/format/downsample/layout/pitch/flags...）, 0x1084/0x10A4=perf。
- GP 寄存器: 0x0=VSCL_START, 0x4=VSCL_END, 0x8/0xC=PLBCL, 0x10/0x14=PLBU_ALLOC, 0x20=CMD（bit0=启VS, bit1=启PLBU, bit4=更新alloc）。

### 3. 崩溃现场利用（dmesg 可读）
- 故意失败的 job 会触发驱动的寄存器转储（"Executor GP: Job X Timeout"）——GP dump 显示 **VSCL_START 前进了 8 字节**，精确定位卡在哪条命令。
- 信号量命令（Lima 的 ARRAYS_SEMAPHORE BEGIN/END）在 GP 上会挂起 → 超时。
- **panic_on_oops=1**，错误写入=重启（watchdog 自动恢复，实测多次无损坏）。

### 4. DDK 用户态（libGLES_mali.so, ARM）逆向到命令序列级
- **libMali 从不构建信号量命令**（0x50000000 全库 0 处）——Lima 的 GP 命令流与本机 DDK 时代不同！
- 找到 DDK GP job 构建器（0x8b5c0），命令序列：
  1. (attr_info_va,  0x20000000|(na<<17))
  2. (out_info_va,   0x20000008|(nv<<17))
  3. (shader_va,     0x40000000|(ninstr<<16))
  4. (attr_count|(ninstr-1)<<10|(pf)<<20, 0x10000040)
  5. (0, 0x10000042)（nv=na=1 时）
  6. (0x02000000, 0)（DRAW num=2）
  7. (shader_va, 0x60000000)
  无信号量/无 UNIFORMS_ADDRESS/无 UNKNOWN1。
- 还找到 DDK 的 PP "color-fill" job 构建器（0x8b260），其 8-dword PP shader 常量与 PoC 完全一致（0xB8000000/0xBC000000）——PoC 作者即由此抄得。
- **PP shader 格式差异**：DDK fill shader 的 texld 带内联描述符地址（0xE0000000|(off>>3)），与 Lima 的"RSW 纹理数组索引"方案不同 → Lima 的 PP ISA 不能直接用于本机。

### 5. 读原语实验记录（全部安全失败，无重启）
- GP 路径: 5 种命令流变体（Lima 全套 / 无信号量 / DDK 精确 7 命令 / DDK 共享描述符+num=2）→ 全部 UNKNOWN_ERR 或超时，输出全零。
- PP 路径: RSW+Lima-ISA texld shader → UNKNOWN_ERR；RSW+常量色 Lima shader → 未执行（输出=清除色）。
- 已排除: 属性/输出描述符布局错误、WB 目标绑定类型、user[328] 标志、cmd 全零、PP/GP ioctl 号。
- 剩余未知: **本机 DDK 时代的 GP/PP shader 指令编码**（Lima ISA 不匹配），以及 SHADER_INFO 位语义、0x8b5c0 的描述符细节。

## ★ 剩余路线

### 路线 A（继续纯软件）：逆向 DDK 的 shader 编译器输出
- 思路：libMali 内含完整的 shader 编译器（__mali_link_binary_shaders 等符号）。逆向其 GP/PP 后端代码生成，得到本机 ISA 的 texld 编码，替换 Lima 编码后重试。
- 或者：hook/触发 DDK 编译一个 `gl_Position=a; v=a;` 的顶点 shader，从其产物直接抄指令字节。
- 工作量：数小时；风险：低（读内核内存安全）。

### 路线 B（最快）：拿到 boot.img → vmlinux-to-elf → 两个常量直接算出 → 用已验证的写链 root（30 分钟内）
- 固件包：小米 OTA（ota.cdn.pandora.xiaomi.com/rom/617/matrix/user/6.0.1.1275/...）已被清理；可尝试 znds 论坛/网盘找 "matrix 6.0.1.1275" 或 "8.8.20.1275" 完整包（注意校验 md5 4079c2d9ad197b044d069d5d19db30af）。
- 或拆机 UART（cmdline console=ttyS0,115200），U-Boot `mmc read` boot 分区。
- 设备上的常量验证手段已就绪：读出后用 dmesg 泄漏的 .text/.data 边界 + /proc/kallsyms 顺序交叉核对。

### 路线 C（备用）：pstore
- /sys/fs/pstore 已挂载（ramoops），但 shell 无权读。若未来拿到任意内核读/写，可顺手利用。

## 设备现场
- /data/local/tmp/: gpu_arb_write（含 0x200 修复+校准模式）、kfunc（已验证可执行）、gp_read、pp_read
- adb 192.168.2.105:5555，掉线后 `adb connect` 即可（watchdog 自动重启恢复）
- 本机 workspace: mitv4-utgard-lpe-main/（全部源码+产物）、mali.ko、libGLES_mali.so、lima_draw.c 等参考资料、dmesg 抓取

## 原第一阶段报告（背景）

调研对象: https://github.com/Yeuoly/mitv4-utgard-lpe
设备: 小米电视 4A 43" / MiTV4A (matrix), 192.168.2.105, adb 已授权
结论: **方案可行性已实机验证, 写原语已跑通, 差两个固件常量即可 root。**

## 一、设备指纹 (实测)

| 项目 | 值 | 与 PoC 验证机 (MiTV4 xmen) 对比 |
|---|---|---|
| 产品/机型 | matrix / MiTV4A | 同为 Amlogic S905L 系 (dmesg: mi_board_name=txl_matrix) |
| 系统 | Android 6.0.1, build 6.0.1.1275 (显示 8.8.20.1275), MHC19J | 同版本系 |
| 内核 | Linux 3.14.29-g4f3e19b #1 SMP PREEMPT 2018-08-20, arm64, 32位用户态 | 同系 |
| Mali 驱动 | r6p1-01rel0 (/proc/modules 显示为外置模块) | **完全一致** |
| /dev/mali | crw-rw-rw- (0666) | 一致 |
| /dev/ion | crw-rw-rw- (0666) | 一致 |
| SELinux | Enforcing, panic_on_oops=1, panic=5, 无 KASLR 痕迹 | 一致 |
| 线性映射 | dmesg: memory 0xffffffc000000000-0xffffffc080000000 → va=phys+0xffffffc000000000 | 一致 |
| 内核布局 (dmesg) | .text 0x1080000-0x1dde474, .init ..0x1f16000, .data ..0x2029610 | 获取自开机日志, 无需 root |

## 二、漏洞存在性 (实机确认)

1. 拉取了 `/system/lib/mali.ko` (8MB, 带 DWARF), 逆向确认:
   - `mali_mem_validation_check` **被编译成 `mov w0,#0; ret`** —— 物理地址校验被去掉, 与 Amlogic 上游一致。
   - `mali_mem_bind_ext_buf` 调用该校验后直接更新 GPU 页表 → 任意物理地址绑定成立。
   - ioctl 命令号与 PoC 完全一致: MEM_BIND=0xc0288302(40字节), PP_START_JOB=0xc1988400(408字节), WAIT_NOTIF=0xc0688202。
   - **驱动差异**: 本机 MEM_BIND 按 `(flags & 0x3f00)` 分派, 0x200→dma_buf, 0x800→ext_memory; PoC 对 dma_buf 用 flags=0 在本机会失败(EFAULT)。已在 `src/gpu_arb_write.c` 中加 `BIND_DMA_BUF 0x200` 修复。
2. 64 位静态程序可执行 (kfunc 前提成立), 用户态纯 32 位 (无 /system/lib64)。

## 三、写原语 (已跑通)

```
[*] calibration: request 0x800 -> changed [0x800..0x87f], 128 bytes  OK
```
zig 交叉编译 (arm-linux-musleabihf), adb push 后自校准通过: 请求偏移 0x800 精确命中 128 字节, 无越界。
(为防误操作已加 phys=0 校准-only 模式; 泄漏教训: 曾向物理地址 0 写 128 字节导致系统重启, watchdog 自动恢复, 无损坏 —— panic_on_oops=1 下错误槽写入的代价=一次重启。)

## 四、卡点: 两个固件常量

run.sh 需要:
1. `KLN_ADDR` = kallsyms_lookup_name 的虚地址
2. `SLOT_PHYS` = sys_call_table[244] 的物理地址

- `/proc/kallsyms` 可读但地址全为 0 (kptr_restrict); 符号名+顺序可见 (91624 个)。
- dmesg 泄漏了若干锚点 (pmae_reg_drv 函数指针、vdec irq handler、各 section 边界), 但都在内核后段, 无法反推早期 .text 符号地址。

### 固件镜像获取 (全部失败)
- 触发 `com.duokan.intent.action.UPGRADE` 抓到本机 OTA 响应:
  `http://ota.cdn.pandora.xiaomi.com/rom/617/matrix/user/6.0.1.1275/6.0.1.1275/package-6.0.1.1275.zip`
  md5=4079c2d9ad197b044d069d5d19db30af, size=406724176 —— **但文件已被 CDN 清理 (404)**;
  RELEASE_NOTES 小文件仍 200, 路径规则有效。
- 同规则暴力探测 25 个版本号: 全 404。web.archive.org 无任何快照。
- OTA API 有 opaque 签名, 伪装旧版本返回 status:19。
- /dev/block、/cache、/sys/fs/pstore、last_kmsg 对 shell 均不可读; /dev/ge2d 等 DMA 节点非 0666。

## 五、剩余方案 (两条路)

### 路线 A (纯软件, 推荐): 自建 GPU 任意读原语
写原语已可用, 只需补"读"。资料已齐 (Lima 开源驱动 + Amlogic r6p1 内核源码 + 本机 mali.ko 逆向):
- PP 寄存器语义已全部弄清: 0x0=FRAME(PLBU数组), 0x4=RSW指针, 0x30/0x34=fragment stack, 0x100/0x200/0x300=WB0/1/2 (各12字: select/addr/format/downsample/layout/pitch/flags...), 0x1084/0x10A4=perf counter。
- PoC 的 408 字节 job 结构 = _mali_uk_pp_start_job_s: [20..112]=frame_registers[23]→PP regs 0x0..0x58, [168..220]=WB0→PP 0x100..0x12C, [400..408]=通知指针。全零 RSW → 无 shader 的 "clear" 路径 = 现在的写原语。
- 读原语两条实现:
  - **GP 路径 (更简单)**: GP job (frame=6寄存器) 的 VS 命令栈引用 SHADER_ADDRESS/ATTRIBUTES_ADDRESS/VARYINGS_ADDRESS; 把属性数组指向内核页 (MEM_BIND ext 绑到 GPU VA), varying 输出指向自己的 ion 缓冲; 顶点 shader = "load attribute → store varying" (Lima ir/gp ISA, 1-2 条指令)。GP 把内核内存搬到我们的缓冲 = 任意读。
  - PP 路径 (备选): RSW + 纹理描述符 (Lima genxml 已有完整定义: RGBA8888, Mip0=绑定的内核页, Unnorm Coords) + texld shader + PLBU 命令流。
- 有了读之后:
  1. 扫描 .data (0x1f16000-0x2029610) 找 16 个连续相同的 .text 指针 = sys_ni_syscall run → 定位 sys_call_table → SLOT_PHYS;
  2. 扫描内存 kallsyms 表 (kallsyms_addresses/num_syms/names/token_table 特征定位) 解出 kallsyms_lookup_name 地址;
  3. 之后完全按 PoC run.sh: 槽→KLN 解析 memcpy/selinux_enforcing/core_pattern → SELinux permissive → core_pattern→roothelper → 触发 coredump → uid=0, 装 setuid su, persist adb tcp 5555。
- 工作量: 数小时级, 每次错误尝试可能触发一次重启 (可自动恢复)。

### 路线 B (依赖外部): 拿到 boot.img
- 拆机接 UART (cmdline: console=ttyS0,115200), 若 U-Boot 可中断则 `mmc read` + tftp/ymodem 导出 boot 分区; 或
- 用户在社区 (znds 论坛/网盘) 找 4A matrix 6.0.1.1275 完整包;
- 拿到后用 vmlinux-to-elf 直接重建符号表, 两个常量立刻算出, 半小时内出 root。

## 六、风险与恢复
- 所有错误写入的后果 = panic → watchdog 5 秒重启 (实测 2 次均无损恢复)。全链成功前不碰 /system。
- roothelper 落地动作可逆: `mount -o rw,remount /system && rm /system/xbin/su` + `setprop persist.adb.tcp.port ""`。
- 现场状态: gpu_arb_write(含 0x200 修复+校准模式)/kfunc 已在设备 /data/local/tmp。

## 七、本机产物
- workspace: mitv4-utgard-lpe-main/ (改好的源码+编译产物), mali.ko, MiTVUpgrade.apk, libNativeApi.so, dmesg.txt, logcat.txt
- 工具: tools/zig (交叉编译), tools/platform-tools (adb), python+pyelftools/capstone/androguard/vmlinux-to-elf
- OTA 端点 (备用): http://221.194.175.120/tvservice/getupgradeinfo?upgradetype=0&appversion=1.3.85.0&incremental=1275&systemrelease=6.0.1&buildtype=user&product=matrix&buildid=MHC19J&deviceid=...&apiver=1.8&ver=6.0.1.1275&codever=0&ptf=617&ts=...&nonce=...&opaque=... (Host: upgrade.ptmi.gitv.tv)
