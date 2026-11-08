# 操作与排障手册：无屏幕设备的远程调试

适用：设备无屏幕、只能靠网络回连、重启要人授权；内核实验要反复「改配置 → 构建 → 启动 → 看结果」。
写于 2026 年，高通 SM8250 一代 + 主线 6.16/6.17 实践。

这份文件只写"怎么做"和"怎么判断"，领域知识（显示、音频、USB 等）看对应文件。

---

## 1. 三条硬规矩

1. **重启前确认兜底**：任何自定义内核都用内存启动（`fastboot boot`），失败后由内核自己 `panic=` 超时重启回到 flash 里的正常系统。**永不**在没有验证过同一文件的情况下写分区。
2. **一次只改一个变量**：内核、设备树、initramfs、cmdline 四者中只动一个。设备树和 initramfs 尤其容易一起改、然后谁也说不清结论。
3. **结论先落盘再继续**：每个实验的记录写成「变的是什么 → 现象 → 断点 → 结论」。失败通道写成「已关闭 + 机理」，否则下次会重做一遍。

---

## 2. 标准操作流程

### 2.1 构建（CI 交叉编译）

- 构建放 CI，本机只做语法/对象级检查。产物（boot.img、`Image.gz-dtb`、`.config` 或 `config-drift.diff`、ccache 统计）一起下载归档。
- **坑【高频】：`.github/` 常在 `.gitignore` 里**，`git add` 会静默忽略，随后 `git commit` 报「无文件要提交」，`git push` 却成功——结果 CI 跑的还是旧 workflow，人却以为改动生效了。
  - 解决：`git add -f .github/workflows/<file>.yml`，提交后用 `git diff --stat <base> HEAD` 确认真的动了。
- **坑【高频】：怀疑「CI 没真编」时不要猜，直接查产物**。内核镜像通常是压缩的：
  ```sh
  zcat Image.gz | strings | grep -m5 -E 'lru_gen|MPTCP|erofs|binder'
  ```
  字符串在 = 编进去了。同时看 `config-drift.diff`（相对基线配置的差异）比看 ccache 统计可靠——ccache 的命中数是缓存生命周期累计值，不能说明这一次编了什么。

### 2.2 内存启动（不写分区）

```sh
# 1) 进 bootloader
reboot bootloader                # 或 fastboot reboot bootloader
# 2) 等设备出现
fastboot devices                 # 出现即就绪，通常 5–10 s
# 3) 只装进内存
fastboot boot /path/boot.img     # 返回 OKAY 只代表传完了，不代表启动成功
```

- **`fastboot boot` 返回 OKAY 不等于新内核在跑【致命】**。ABL 可能拒绝镜像后静默回落到 flash 里的系统。判定方法见 §2.4。
- 内存启动天然带兜底：crash / panic / 看门狗 → 重启 → 回到 flash 里的系统，不需要人碰设备。

### 2.3 写入分区（只有验证通过才做）

```sh
fastboot flash boot_b /path/boot.img   # 与刚 boot 过的文件必须是同一个（比 sha256）
fastboot reboot
# 起来后再确认一次版本字符串和服务
```
回滚镜像（原厂或上一个验证过的版本）必须常备在手边，且能在一分钟内刷回去。

### 2.4 验证清单（每次启动都过一遍）

| 查什么 | 命令 | 判读 |
|---|---|---|
| 内核版本与**构建时间戳** | `cat /proc/version` | 时间戳与你这次 CI 的构建时间对得上，才算真跑上了 |
| 启动时长 | `cut -d' ' -f1 /proc/uptime` | 和「发出 fastboot boot 之后过了多久」对得上；小于等待时间 = 中途重启过 |
| cmdline | `cat /proc/cmdline` | 前面几项是你写的，后面是 bootloader 追加的 |
| 特征文件 | `cat /sys/kernel/mm/lru_gen/enabled`、`/proc/net/tls_stat`、`/proc/sys/net/mptcp/enabled`、`grep erofs /proc/filesystems` | 文件存在且值合理；**读不到要区分「没有」和「路径不同」** |
| 子系统 | `ls /sys/class/power_supply/` | 别假设节点名是 `battery`，可能是 `<pmic>-charger`、电量计等 |
| 日志 | `dmesg \| grep -cE 'Oops\|BUG:\|Call trace'` | 0 才算干净 |
| 对外服务 | 本地 `curl` 关键端口 | 设备是服务器时，这项比任何内核指标都重要 |
| 日志断点 | 读裸分区落盘点（§3） | 阶段行到哪为止 |

**uptime 不可靠**：设备重启后若时间同步失败，墙钟会跳。判断「跑的是哪颗内核」只认 `/proc/version` 里的构建时间戳。

---

## 3. 早期日志取证（唯一可靠的办法）

无屏幕设备最痛的是「死在用户态之前，什么都拿不到」。可靠做法只有一种：**initramfs 阶段写裸分区 + 立即 sync**。

### 3.1 落盘点

- 找一个**无文件系统**的小分区（几百 KB 到几 MB 就够），或任何不会被挂载的裸分区。
- 定位不要写死设备名：遍历 `/sys/class/block/*/uevent` 匹配 `PARTNAME=<label>`，拿 `DEVNAME` 的 `major:minor` 自己 `mknod`（早期 initramfs 里 `/dev` 可能还没这个节点）。
- **写完必须 `sync`**：硬复位/panic 时页缓存里的数据会丢，这正是「诊断脚本跑了但日志不存在」的常见原因。

```sh
dd if="$tmp" of="$dev" bs=1024 seek="$off" conv=notrunc 2>/dev/null
sync
```

### 3.2 阶段探针

在 initramfs 的每个阶段钩子里写一个槽位（`init-top`、`init-premount`、`local-top`、`local-premount`、`local-bottom`、`init-bottom`、`switch_root` 前），每个槽位写：阶段名、`/proc/uptime`、`/proc/cmdline`、`dmesg` 尾部、块设备列表。

读回：

```sh
dd if=<part> bs=65536 count=64 | ...          # 整块取出
for off in 0 64 128 192 256 320 384 448 512; do
  dd if=logfs.bin bs=1024 skip=$off count=1 2>/dev/null | head -c 200 | tr -d '\0' | head -1
done
```

槽位首行告诉你走到哪一步；**最后一个有内容的槽位就是死点**。若 `init-bottom` 之后的阶段都写了，说明 initramfs 走完了，问题在用户态。

### 3.3 活着也要能回话（看门狗）

`init-top` 里 fork 一个后台：

```sh
(
  sleep 120
  write_checkpoint watchdog 384      # 超时还没起来 → 留下最后一段 dmesg
  sync; sleep 1
  reboot -f                          # 内存启动时自动回到原系统
) >/dev/null 2>&1 &
```

配套要点：

- **fork 出来的进程要能活过 `switch_root`**：把 busybox 和探针脚本拷进 `/run`（`/run` 会被搬到新根，initramfs 其余内容会被删掉），用 `/run/.../busybox sh /run/.../<probe>` 调，别用 `/bin/sh`。
- 设备节点可以不重找：initramfs 的 `/dev` 是 devtmpfs，`init-bottom` 会把它 `mount -o move` 到新根，**同一个节点路径在切换后依然有效**。
- 探针别只在超时写一次。**每秒写一次**「dmesg 尾部」到固定槽位，才能拿到死前最后几条（panic 发生时用户态已停止调度，panic 正文抓不到，但触发前 1 秒的日志通常足够定位）。

### 3.4 已关闭的通道（别再试）

| 通道 | 为什么不行 |
|---|---|
| netconsole over USB gadget 以太网 | gadget 侧 MAC 全零、主机侧 `cdc_ether` 收到畸形帧，实测 0 包 |
| pstore / ramoops | 设备树里可能有 4 MiB 保留区，`ramoops` 后端注册成功、console 也注册了，但重启后 `/sys/fs/pstore` 恒空——这块内存在重启过程中存不住。`/dev/pmsg0` 存在也救不了 |
| `console=ttyGS0`（USB ACM 当控制台） | ACM 的 tty 由 configfs 在 initramfs 里创建（约 5 s），而 console 只在 `console_init`（<1 s）注册；这个内核没有「tty 后到再绑定 console」的路径。`/proc/consoles` 里根本没有它，`/dev/kmsg` 写进去主机收不到 |
| `init=<诊断脚本>` 里 `cat /proc/kmsg` 落盘 | 能跑、能写出文件，但 panic 发生在用户态之后，未 `sync` 的行全丢，且进程在 panic 期不再被调度 |

---

## 4. 排障条目

### 4.1 `fastboot boot` 返回 OKAY，但跑的还是 flash 里的内核【致命】
- 现象：新内核的版本/特征查不到；`uptime` 比等待时间还短；日志断点里没有新一次启动的阶段行。
- 原因：bootloader 拒绝了这次内存启动（镜像过大/格式/设备树不匹配），静默回落；或根本没有真正重启进 bootloader。
- 怎么确认：看 `/proc/version` 的构建时间戳；看裸分区里有没有本次 `fastboot boot` 之后写的阶段行。
- 怎么解决：换回**上一次能起**的镜像做对照，确认流程本身没坏；再逐步改回本次的差异（内核 / ramdisk / cmdline 三者分开）。
- 提前预防：每个镜像都带检查点 ramdisk，用「有没有阶段行」而不是「我按下去了」判断。

### 4.2 同一颗内核，换个 initramfs 结论就反了【耗时】
- 现象：内核 + 自建 ramdisk 能进系统；内核 + 官方/CI 的 ramdisk 起不来。
- 原因：变量没固定——ramdisk 里的模块集、init 脚本、cmdline 追加项都会改变结论。
- 怎么确认：用同一颗内核分别配两种 ramdisk 各启动一次，只比这一项。
- 怎么解决：先把「内核是否可用」和「用户态是否可用」分成两个实验，别混在一次里。
- 提前预防：实验记录里写清 kernel / dtb / ramdisk / cmdline 四项的来源和哈希。

### 4.3 调试 cmdline 把警告升级成 panic【高频】
- 现象：某组配置「一定起不来」，换个人/换个分支同样的配置又能起。
- 原因：cmdline 里有 `softlockup_panic=1`、`hung_task_panic=1`、`hardlockup_panic=1`、`nmi_watchdog=1`、`panic_on_warn` 这类开关。它们把本来只是警告的情况变成 panic；而这类警告与本次改的配置无关。
- 怎么确认：把 cmdline 换成 `root=<root> rw console=tty0 panic=15`，其它一切不动，再启动一次。
- 怎么解决：**判定「某配置能不能起」的对照实验必须用干净 cmdline**，调试开关只在已经确认能起之后再逐个加。
- 提前预防：把调试 cmdline 和正式 cmdline 分成两个构建参数，别写死在工作流里。

### 4.4 CI 看起来没编（ccache 误判）
- 现象：构建几分钟就结束，怀疑没编。
- 怎么确认：产物里 `zcat Image.gz | strings | grep <符号>`；看 `config-drift.diff`。
- 提前预防：产物里固定带上 config 差异和版本字符串，别靠 ccache 统计判断。

### 4.5 远端脚本里的本地变量不展开 / 写 `/root` 权限不够
- 现象：管道给远端 `bash` 的脚本里 `$VAR` 是空的；`cat > /root/x.sh` 报权限不够。
- 原因：变量在远端展开，而远端没有这个环境变量；`/root` 对普通用户不可写。
- 怎么解决：先落脚本到 `$HOME`，再用 `sudo -S bash $HOME/x.sh` 执行；需要密码时把密码从**本地**管道喂给 `sudo -S`。
- 提前预防：任何要 sudo 的远端操作都做成「本地写脚本 → 远端 `$HOME` → sudo 执行」三步。

### 4.6 `pkill -f` 杀掉自己【高频】
- 现象：后台任务莫名其妙 SIGTERM 退出，或者一条命令执行完自己的影子进程都没了。
- 原因：`pkill -f "cat /dev/ttyACM0"` 会匹配到发起这条命令的进程自己的命令行。
- 怎么解决：用字符类打断自匹配：`pkill -f 'cat /dev/ttyACM[0]'`；或用 `pkill -x <name>`；或记录 PID 精确 `kill`。

### 4.7 sysfs 属性读不到 ≠ 驱动没起来
- 现象：`cat /sys/class/power_supply/battery/capacity` 报没有这个文件，但设备其实工作正常。
- 原因：节点名随驱动实现不同（`<pmic>-charger`、`bq27xxx-0`、`tcpm-source-psy-*` 等）。
- 怎么解决：先 `ls /sys/class/power_supply/`（或对应 class 目录）看实际名字，再读属性；不要硬编码。

### 4.8 「两组配置符号零重叠，却都起不来」
- 现象：拆分二分后每一半都失败，于是怀疑「不是单个符号的问题」。
- 原因：多半不在符号上，而在**共同的实验条件**上——cmdline 的调试开关、ramdisk、构建缓存、回落到 flash 内核（§4.1）。
- 怎么解决：在做第二轮二分之前，先把「基线配置 + 完全相同条件」跑通一次，确认基线本身能起；再确认每个变体真的在跑（§2.4）。

---

## 5. 判定表

| 症状 | 先查 |
|---|---|
| 完全没反应、USB 也不枚举 | bootloader 是否在（`fastboot devices`）；镜像是否被拒 |
| 网络回来得比预期快（uptime 很小） | 是否回落到 flash 内核（§4.1） |
| 日志断点停在 initramfs 某阶段 | 该阶段的钩子/模块/驱动；对照上一阶段 dmesg 尾部 |
| 日志断点走到最后但服务没起 | 用户态（systemd 单元、模块 ABI、挂载） |
| 特征文件不存在 | 先确认跑的是不是新内核（§2.4），再查配置是否真的编进去（§2.2） |
| 一切正常但偶尔硬复位 | 电源路径、温控、看门狗；换干净 cmdline 复核（§4.3） |

---

## 6. 记录纪律

- 每个实验写四行：**改了什么 / 现象 / 断点在哪 / 结论**。没有断点的结论一律标「未定」。
- 关掉的通道写「已关闭 + 机理」，不要只写「试过不行」——后者下次还会被重新捡起来。
- 判断「某件事做过没有」时读记录文件，不凭印象；做之前先搜一遍经验库，避免重造轮子。
