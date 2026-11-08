# 无屏幕设备上的内核配置二分（方法论定稿，2026-11-08）

> 场景：手机/板子只有网络能连，内核改动后「起不来」，且没有任何串口/UART 可看。
> 目标不是「猜哪个 CONFIG 有问题」，而是**用 4 分钟一轮、可自恢复的实验，把死因定位到一个符号**。
> 本文是 2026-11-08 在 oneplus8（SM8250 / 主线 6.16.7）上真打出来的流程，含三次误判的教训。

---

## 0. 先决条件（缺一条就别开始）

1. `fastboot` 可用，且**只在内存里启动**（`fastboot boot`），永不 flash 未验证镜像；
2. 设备有可写的裸分区（本例 `logfs`，4 MiB，`PARTNAME=logfs`）用来落盘取证；
3. 会维护一份「检查点 initramfs」——把探针脚本塞进 Debian initramfs 的 `init-top`；
4. 有一个**硬件看门狗**兜底（本例 bootloader 参数 `phx_rus_conf.kernel_time=240`，实测 2–3 分钟自动复位）；
5. 构建在 CI（GitHub Actions）做，本地只打包 + 启动，避免本地交叉编译环境差异。

---

## 1. 判据：唯一可信的是「cmdline 里的 nonce」

**错的判据（都骗过我）**：`fastboot boot` 返回 OKAY / SSH 能连上 / uptime 很小 / 服务都正常。
这些在「内存内核挂了、回落到 flash 内核」时**全都成立**。

**对的判据**：打包时在 cmdline 里塞一个内核不解析的惰性参数，启动后回读：

```sh
# 打包
echo "root=/dev/sda24 rw console=tty0 panic=15 zzram=$(date +%s)" > cmd.txt
python3 pack_bootimg.py --kernel Image.gz-dtb --ramdisk rd.gz --cmdline "$(cat cmd.txt)" --output boot.img
# 启动后验证
rsh server 'cat /proc/cmdline' | tr ' ' '\n' | grep ^zzram    # 对得上才是你那个内核在跑
```

要点：
- **每次启动换新 nonce**。固定标记被多个镜像共用时，事后无法分辨现场属于哪一次。
- 不要用 `netconsole=…` 当指纹：它可能根本没编进内核（`# CONFIG_NETCONSOLE is not set` 时那串是死参数），而且会让人误以为问题跟 netconsole 有关。
- 不要用调试开关（`softlockup_panic=1 hung_task_panic=1 hardlockup_panic=1`）做 A/B——它们把警告升级成 panic，会污染对照组。

---

## 2. 取证装置：检查点 initramfs（跨 `switch_root` 存活）

### 2.1 结构

```
scripts/init-top/00-zz-arm     # 最先执行：准备 /run/zz 并拉起探针、看门狗
scripts/zz-probe               # 每秒写检查点（pre-* / chrooted / post-N）
scripts/zz-wd                  # 200 s 后 systemd 没起来就硬复位
scripts/zz-ckpt                # 真正写盘：<stage> <offsetKiB>
scripts/init-bottom/save-dmesg # 把整份 dmesg 落到真实根 /var/log/early-dmesg.txt
```

### 2.2 四个静默失败点（全部实际踩过）

| 坑 | 现象 | 正解 |
|---|---|---|
| `init` 把 `/run` 挂成 `nodev,noexec,nosuid` | `cp busybox /run/zz/` 后执行 EACCES，探针**一条都不写**且无报错 | 在 `/run/zz` **再挂一个允许 exec 的 tmpfs**（会随 `mount -o move /run` 进真实根） |
| `switch_root` 把 `/run`、`/dev` 整棵搬走 | 交接后老根视图里路径全空，探针/看门狗静默死亡 | 入口改用搬过去的位置 `/root/run/zz/busybox`，并 `chroot /root` 后再写（真实根的 `/dev` 才有块设备节点） |
| initramfs 里没有 `setsid` | 后台任务压根没起来 | 用 shell 的 `&`，别依赖 coreutils |
| busybox 是动态链接 | 跨根后 loader 随老根被删 | 用静态 busybox，或把 `ld-linux-aarch64.so.1` 一起放进 exec tmpfs |

### 2.3 写盘的两个数据完整性问题

- **`dd conv=notrunc` 不清零**：新记录比旧记录短时，尾部残留上一次的 dmesg，读的时候两次启动会混在一起。**写前先把整个槽位（64 KiB）清零**。
- **只采信槽位头行**（`### STAGE=… UPTIME=… DEV=…`），正文要与头行的 uptime 交叉验证；对不上就是旧数据。

### 2.4 槽位约定（本例）

`0..320` 各 initramfs 阶段 / `384` 看门狗 / `448` 臂自检 / `512` 每秒探针 / `576` 臂完成 / `640..1088` 历史遗留（勿用）。

读回：`dd if=/dev/sde56 bs=1024 skip=512 count=8 | head -1`。

---

## 3. 基线纪律：先证明「只有一处不同」

> 2026-11-08 我拿「特性内核」和「能启动的内核」对比，结论写了两轮，最后发现两者还差 3 个提交、驱动文件差 145 行——全部作废。

开跑前必须做：

```sh
git log --oneline  A..B          # 对照分支比候选多/少了什么
git diff --stat    A B           # 到底差哪些文件
```

- 对照内核所在分支 = **唯一基线**，只在其上增删 CONFIG；
- 驱动/设备树/CI 脚本的差异会伪装成「配置的锅」，必须先消掉或明确排除；
- 产物里常常没有 `.config`（`actions/upload-artifact` v4 默认不传隐藏文件），用二进制自证：

```sh
zcat Image.gz-dtb | strings | grep -ci mptcp     # 0 = 没编进去；非 0 = 编进去了
```

---

## 4. 执行方式：CI 并行矩阵 + 一次只信一个变量

1. 每个假设一个分支（`feat-<标签>`），分支名即假设名；
2. `gh workflow run build-bootimg.yml --ref feat-X`，8 个并行也无妨（构建在云端，互不干扰）；
3. 每个产物用**同一个 ramdisk、同一条干净 cmdline（换 nonce）** 打包 → `fastboot boot` → 读 512 槽位头行；
4. 先按**组**收敛（内存/网络/文件系统），再对可疑组做**单特性**隔离；
5. 每轮 4–5 分钟（启动 + 回落 + 取 logfs），失败会自动被看门狗/硬件狗复位，不需要人守。

**结果解读**：

| 头行 | 含义 |
|---|---|
| `pre-1` ≈ 2.7 s | 死在 `switch_root` 交接瞬间（构建/驱动/核心配置类） |
| `post-3` ≈ 7 s | 交接成功，用户态早期死（systemd 起来后触发） |
| `post-N` 持续增长 | 这次配置是安全的 |
| 完全没有新写入 + nonce 不在 cmdline | **回落到了 flash 内核，实验无效** |

---

## 5. 本次实证（2026-11-08，主线 6.16.7 / oneplus8）

基线 `c74ddab8b`（含浮充修正的充电驱动），候选 13 个 CONFIG：

| 内核 | 配置 | 结果 |
|---|---|---|
| `feat-all2` | 全部 13 个 | 死在交接 `pre-1 2.78` |
| `feat-h1` | USERFAULTFD + LRU_GEN(+ENABLED) | 通过 `post-14 19.23` |
| `feat-h2` | TLS + TCP_CONG_ADVANCED + BBR + NET_SCH_FQ + MPTCP(+IPV6) | 死在交接 `pre-1 2.79` |
| `feat-h3` | ANDROID_BINDERFS + EROFS(+ZIP+ZSTD) | 通过 `post-14 19.73` |
| `feat-s-mptcp` | 仅 MPTCP + MPTCP_IPV6 | 过交接后 ~7 s 死 `post-3 7.18` |

结论：内存组、文件系统组无罪；交接即死来自 TLS/`TCP_CONG_ADVANCED`/BBR/`NET_SCH_FQ` 之一；MPTCP 另有一个独立的 ~7 s 死法（即使单独开也会死）。

---

## 6. 三次误判的根因（给未来的自己）

1. **把回落当成功**：用 SSH 是否回来判断，结果 flash 内核一直在替我「成功」。→ 判据必须来自 cmdline nonce。
2. **装置静默失效**：`/run` 是 noexec，探针一条没写，却因为「槽位有旧内容」看着像有记录。→ 装置要先自证（臂自检槽位 448 + 每次清零）。
3. **基线不同血缘**：对照内核带着 3 个额外提交。→ 先 `git log A..B`，再谈「只改了一个变量」。

**通用原则**：任何「结论」都必须能指出一条**来自产物本身**的证据（cmdline / 二进制 strings / 槽位头行），推断不算。

---

## 7. 探针自己的失败路径会伪装成「内核死了」（2026-11-08 当天最后发现的坑）

写探针时我用了 `chroot … || exit 0`。结果：当 `chroot` 因为真实根还没就绪而失败时，探针**在 2.8 s 静默退出**，槽位最后一条永远停在 `pre-1 UPTIME=2.78`——看起来和「内核在交接瞬间硬死」一模一样。

判别与修法：

- **任何失败分支都必须落盘留痕**：`wait-move-N`、`chrootfail-N`、`CHROOT-GAVE-UP`、`NO-MOVED-ENTRY` 各写一条，绝不允许静默 `exit`。
- **等待要留痕**：等 `/run` 搬移、等真实根就绪的每一秒都写一条 `wait-*`，这样「最后一条是什么」就能区分「在等」和「已死」。
- **判据链**：`pre-*` 只说明 initramfs 阶段活着；`chrooted` 说明进了真实根；`post-N` 递增才说明用户态在跑。**没看到 `chrooted` 就不能断言「死在交接」**。
- 对照组交叉验证：同一个内核隔一段时间再跑一次，死点应当稳定；死点漂移说明测的是探针/时序，不是内核。

**教训**：测量装置的每一种失败都必须有可见输出，否则「没测到」会被读成「死了」，而结论会一路传下去污染整条二分链。

---

## 8. 最终定位结果（2026-11-08 收尾）

同基线（`c74ddab8b`）、同 ramdisk（`rd-live7`）、唯一 nonce 下：

| 配置 | 结果 | 判定 |
|---|---|---|
| USERFAULTFD + LRU_GEN(+ENABLED) | `post-14` 持续 | 安全 |
| ANDROID_BINDERFS + EROFS(+ZIP+ZSTD) | `post-14` 持续 | 安全 |
| TCP_CONG_ADVANCED + BBR + NET_SCH_FQ | `post-14` 持续 | 安全 |
| **仅 TLS** | `post-3 7.27` | **致命** |
| **仅 MPTCP + IPV6** | `post-3 7.18` | **致命** |
| TLS+BBR+FQ | `post-2 6.25` | 致命（TLS 导致） |
| TLS+BBR+FQ+MPTCP | `pre-1 2.76/2.79` | 致命且更早（组合放大） |
| 13 项全开 | `pre-1 2.78` | 致命 |

方法上的收获（比结论更重要）：

1. **组→单特性的收敛顺序**能一步把 13 个符号砍到 2 个：先 3 组，再对致命组做单特性；
2. **一个符号单独致命、两个一起更早死**这种组合放大很常见，不要因为「单开能过」就认定组合安全；
3. 判据必须来自产物（nonce / 槽位头行 / `strings`）——三次误判全部发生在用「间接现象」下结论的时候；
4. 探针的失败路径必须留痕（§7），否则「没测到」会被当成「死了」。
