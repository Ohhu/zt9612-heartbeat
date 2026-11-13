# zt9612-heartbeat

ZT9612U（ZTOP / 兆通微 ACEV100，USB ID `350b:9612`）的 Linux 驱动，在上游
[Spkicn/zt9612-linux](https://github.com/Spkicn/zt9612-linux) v0.3.2 基础上修掉**固件看门狗
导致的随机复位**：网卡用着用着变回光驱、整机断网几十秒。GPL-2.0，与芯片厂商无隶属关系。

## 能不能用

| 项目 | 结论 |
|---|---|
| 网卡 | USB ID `350b:9612`。实测机型 COMFAST CF-945F（AX1800 USB，模组 ZT9612U V1.0） |
| 出厂状态 | 默认枚举为光驱 `350b:f179`，需切一次模式才能当网卡用（脚本已带自动切换） |
| 内核 | 6.12 及以上可编译；实机验证过 7.0.0-31 / 7.0.0-34 / 7.0.0-42（Ubuntu 24.04 x86_64） |
| 工作模式 | 仅 managed（STA）：双频扫描、WPA2-PSK、DHCP、IPv4/IPv6 数据面 |
| 未实现 | AP 模式、监听与注入、蓝牙、80 MHz、WPA2-Enterprise（代码路径未验证） |
| 固件 | 需自备 `zt9612_fw.bin` + `zt9612_settings.bin`，仓库不分发（厂商版权） |
| Secure Boot | 支持，需自行生成 MOK 密钥并注册 |
| 不适用 | 同厂 `350b:9101`、`350b:9611` 等其它型号 |

实测吞吐（端到端，对端为另一台设备上的 TCP sink，接收侧逐字节核对，数据来自上游）：5 GHz
上传 12.7–14.9 Mbit/s、下载 18.5–20.4 Mbit/s；2.4 GHz 上传 6.5–7.5 Mbit/s。上传上限在固件
TX 侧，主机侧改不动。`iw link` 里的速率数字是主机侧记账，不是真实发射速率。

## 安装

前置：`build-essential` 与对应内核头文件；装好前别把现有网卡拔了。

```bash
sudo ./scripts/install-firmware.sh <固件目录>   # 校验 SHA-256 后装到 /lib/firmware
sudo ./scripts/setup-mode-switch.sh            # 插卡时自动从光盘模式切到网卡模式
sudo ./install-driver.sh                       # 编译、按需签名、安装，并加载一次
sudo ./install-driver.sh --dkms                # 同上，但走 DKMS，内核升级自动重建
```

默认会写 `blacklist zt9612`，只挡插卡/开机自动加载，手动 `modprobe zt9612` 可用；要开机自动
加载加 `--enable-autoload`。验收命令与期望输出、Secure Boot 签名、DKMS 补签见
[docs/INSTALL.md](docs/INSTALL.md)。

插上只看到光驱、或者装不上，见 [docs/FAQ.md](docs/FAQ.md) 与
[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)。

## 为什么需要这个补丁

固件要求主机周期性发心跳（`0x05c2`），超时即复位，设备退回 `350b:f179` 光盘模式，主机侧没有
补救手段，只能断电重来。上游文档写的窗口是"超过 5 秒不发心跳"。

上游把心跳周期设成 5000 ms，而实际周期是"设定值 + 本次执行耗时"。在本机用 usbmon 抓了
809 秒共 200 次心跳：

| 统计量 | 值 |
|---|---|
| 中位周期 | 5.12 s |
| p90 / p99 | 5.18 s / 5.48 s |
| 最大周期 | 5.49 s |
| 短于 5 s 的次数 | 0 |

按文档口径这等于零余量。窗口真值没有测出来（只能由复位样本反推），所以不是每次心跳都复位，
而是间隔被拉长时随时可能复位 —— 扫描持锁、命令在途、bulk 超时都会把它拉长。实测后果是
2026-11-12 11:31–15:07 的 3.5 小时里 16 次 USB 断开，每次断 46–90 秒。

## 改了什么

只动 `driver/zt9612.c`（+104 / −17 行），其余代码与脚本和上游一致：

1. 心跳默认周期 5000 ms 改为 2000 ms，新增模块参数 `hb_interval_ms`（200–60000，运行期可改）。
2. 心跳改固定节拍：把本次执行耗时从下个周期里扣掉，最多扣一半，不再让周期随执行时间累加。
3. 心跳补发点从等 CFM 的循环前移到命令发出之前，扫描期间的长睡眠改成分片睡眠。原因是固件
   IPC 一问一答，命令在途时插一条心跳会占掉命令槽，原命令的 CFM 不再到达。
4. 卸载收尾加固：mac80211 反注册之后再取消工作项、清空发送队列，两个回调入口加存活判断。

代码注释里写了每处改动的依据，逐条明细与实测数据见 [docs/CHANGELOG.md](docs/CHANGELOG.md)。
仓库第二个提交（`driver: use a 2 s fixed-cadence firmware heartbeat`）就是完整补丁，可直接
用 `git show` 审。

## 验证到什么程度

同一台机器、同一张卡、同一个 AP：

| 指标 | 上游 0.3.2（3.5 小时） | 本版本（17 小时 11 分） |
|---|---|---|
| 芯片看门狗复位（退回 `350b:f179`） | 16 次 | 0 次 |
| USB 断开 | 16 次 | 1 次（开机时从出厂光盘模式切到网卡模式，非复位） |
| `bulk OUT failed` | 多次 | 0 |
| `-110` / `-71` | 有 | 0 / 0 |
| oops / panic | 0 | 0 |

17 小时里实际收发 RX 48.2 MB / 228,158 包、TX 763 KB / 7,935 包；扫描 64 次，其中 5 次扫描
前置序列失败（都在改动上线初期，之后为 0，失败自动重试，不影响连接）。期间 AP 自己把信道
从 165 换到 36，驱动随频点变化关联回同一 BSSID，用时约 4 秒。

另有一段 4 小时、252 个采样点的观测（每点 1 分钟，记录默认路由、链路状态与三组 ping）：252 点
全部满足"默认路由恰好一条且走无线"；250 点在同一 BSSID，另 2 点在改动之前的窗口；到网关的
3 个 ping 中，228 点全通、22 点丢 1 个、1 点丢 2 个、1 点全丢；每小时一次的公网检查 4 次全部
返回 HTTP 200。

证据边界：单机、单卡、单个 AP（5 GHz，-57 至 -66 dBm）、Ubuntu 24.04 + 7.0.0-42。这些数字说明
看门狗复位这条路径被堵住了，不等于所有环境都不再掉线 —— 供电异常、固件自身挂死仍会导致复位。

复现命令：

```bash
journalctl -k -b 0 | grep -E 'zt9612|350b' | tail -50
```

## 仓库结构

```text
driver/               zt9612.c（单文件驱动）、Kbuild 兼用户入口的 Makefile
scripts/              固件安装、模式切换、udev 规则、排障信息收集、IPC 自测
install-driver.sh     一键安装：编译 / 签名 / 安装 / 加载 / blacklist
uninstall-driver.sh   卸载文件（不执行 rmmod）
dkms.conf             DKMS 定义，版本 0.3.2-hb1
zt9612.conf           /etc/modprobe.d 参数模板，逐参数说明
supported-device-IDs  USB ID 判定与不支持的同厂型号
docs/                 INSTALL / TROUBLESHOOTING / PARAMETERS / FAQ / CHANGELOG
firmware/             只有占位说明，二进制需自备
```

参数表见 [docs/PARAMETERS.md](docs/PARAMETERS.md)。日常只需要 `hb_interval_ms`，其余是上游的
实验开关，默认值即正常行为。

版本号 `0.3.2-hb1` = 上游 v0.3.2 + 上文改动；本仓库长跑用的 `.ko` 由这份源码编译（唯一差别是
`MODULE_VERSION` 字符串）。

## 已知限制

- 默认不自动加载：`install-driver.sh` 写 `blacklist zt9612`。加 `--enable-autoload` 可开机自动
  加载，但这条完整路径上游尚未复验。
- 不要反复 `rmmod` / `insmod`：卸载时设备可能不应答在途 bulk-IN，驱动按设计宁可泄漏实例也不
  UAF。换模块请重启机器。
- 芯片挂死后只能冷启动复位：`systemctl poweroff` 断电再上电。热重启不断 VBUS，ROM 的固件
  接收窗口不会重开。
- 同一 SSID 有多个 BSS 且信号接近时，NetworkManager 会按单帧 RSSI 来回漫游（本机实测 20 分钟
  4 次）。用 `nmcli con mod <连接名> 802-11-wireless.bssid <BSSID>` 固定到一个 BSS 可消除，代价
  是该 BSS 变成单点。这是主机侧策略，与驱动无关。
- 扫描前置序列偶发 CFM 超时（本机 64 次扫描 5 次），该轮扫描中止后由上层重试。
- 驱动源码里有若干早期编辑损坏的中文注释（双重编码乱码），不影响编译，尚未清理。

## 来源与许可

上游 [Spkicn/zt9612-linux](https://github.com/Spkicn/zt9612-linux)，快照 `9b5dcfe`
（2026-10-08，`PACKAGE_VERSION` 0.3.2），GPL-2.0-only。上游驱动是独立实现：只依据设备对外
可观测的行为与公开的 CEVA RivieraWaves rwnx 消息框架编写，不含厂商源码、反编译产物或厂商
二进制。本仓库是那份代码的再分发加心跳修复，保留原许可与原版权声明，修改说明见
[NOTICE.md](NOTICE.md) 与 [LICENSE](LICENSE)。
