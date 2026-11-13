# 已知问题与故障处置

> 本文件是 [README.md](../README.md) 的故障分册。日常安装见 [INSTALL.md](INSTALL.md)，
> 模块参数见 [PARAMETERS.md](PARAMETERS.md)。

## 先看这张表

用 `sudo ./scripts/collect-debug-info.sh` 收一份排障包，按现象对号入座：

| 你看到什么 | 说明什么 | 下一步 |
|---|---|---|
| `lsusb -d 350b:` 只显示 `350b:f179` | 设备在**光盘模式** | 见「设备自行回到光盘模式」 |
| 什么 350b 设备都没有 | 没插好，或插入后未枚举 | 换口重插；仍无则查供电与 `dmesg` |
| `固件校验` 段出现 `No such file` | 固件没装或路径不对，这是装载失败最常见的原因 | `scripts/install-firmware.sh` 装两个文件（自带 SHA-256 校验） |
| dmesg 有 `hello ack timeout` | 设备没进入固件下载态 | S5 冷启动（见下） |
| dmesg 有 `-110` / `Entity not found` | USB 传输失败，芯片已挂死 | S5 冷启动；**不要**反复 unbind-bind 或 `USBDEVFS_RESET`（实测无效） |
| dmesg 有 `WARNING:` / `BUG:` / `Oops` | 内核侧异常 | 附该行前后各 20 行与复现步骤 |
| dmesg 有 `assert`（`macif.c`） | 固件断言 | 把实验开关恢复默认（见 [PARAMETERS.md](PARAMETERS.md)）后复现；仍复现请附完整 dmesg |
| dmesg 有 `M1+M2 done` 且接口已出现 | 驱动侧正常 | 问题在关联、密钥或 NetworkManager 侧 |

## 设备自行回到光盘模式

固件看门狗复位：主机超过约 5 秒没发心跳（`0x05c2`）。本驱动默认 2 秒一次，并按固定节拍补发，
扫描期间也补。若仍发生，多半是驱动没在跑（例如被 blacklist、模块加载失败），或自己写的用户态
脚本直连 `/dev/zt9612` 时没维持心跳。

## 芯片挂死：只能冷启动复位

出现 `-110`、`can't set config #1`、`Entity not found`，或反复 `hello ack timeout` /
`firmware boot failed: -110`，表示芯片已挂死。热 `systemctl reboot` 不断 USB VBUS，芯片 ROM 的
秒级固件接收窗口不会重开，之后任何 `insmod` 都会失败。已实测无效的手段：端口 disable 往返、
设备级/驱动级 unbind-bind、`USBDEVFS_RESET`、`authorized` 往返、`xhci_hcd` unbind→bind、
`usb_modeswitch -R`、D3cold。

```bash
sudo rtcwake -m no -s 180      # 可选：设 RTC 唤醒闹钟
sudo systemctl poweroff        # 等 ping 100% 丢包 = 真的断电，再上电
```

改代码后必须把新 `.ko` 装进 `/lib/modules/$(uname -r)/extra/`，并移走 DKMS 的
`updates/dkms/zt9612.ko.zst`（`updates/` 优先级更高，否则加载的还是旧代码）。

## 反复 `rmmod` / `insmod` 会泄漏实例

卸载时设备不应答在途 bulk-IN，驱动按设计"宁可泄漏也不 UAF"，实测 12 次卸载里 11 次如此。
**换模块优先重启机器。**

## 默认不自动加载

`install-driver.sh` 默认写 `blacklist zt9612`。原因是历史上"开机自动加载"曾导致机器失联（根因
是漏设 wiphy 父设备，已在 v0.2.0 修复），但"插卡 + 开机自动加载"这条完整路径还没有重新复验。
确认可用后加 `--enable-autoload`。

## `modprobe` 静默返回 0 却没加载

守卫文件里除了 `blacklist` 还可能有 `install zt9612 /bin/true`（手工加过守卫的机器）。用：

```bash
sudo modprobe --ignore-install zt9612
```

这类机器上 `--enable-autoload` 会整个删除该文件（连同守卫）。

## 运行期改 MTU 会短暂失去关联

丢 1–2 个 ping，NetworkManager 会重配。别在运行期反复改。

## 吞吐：上传上限约 15 Mbit/s

端到端实测见 [README.md](../README.md)：UDP 容量上限 15.0 Mbit/s，TCP 已达其 94%，而同一张卡
下载 18.5–20.4 Mbit/s ⇒ 上限在固件 TX 侧，不是链路容量。主机侧可试的几个杠杆（描述符速率字、
速率消息、`MM_STA_ADD_REQ` 的 RC 块、A-MPDU 打包、HT40、A-MSDU）上游都试过，均不提速。
A-MSDU 需要对端支持，两个测试 AP 都报 `amsdu=0`，无法协商。

聚合（`ampdu_en`）与站点会话（`sta_add_en`）默认关闭。它们能长跑但**不提速**，且 UDP 无流控
满压会打死设备；要动这些开关，先读 [PARAMETERS.md](PARAMETERS.md) 里对应条目。

做 TX 实验必须带功能判据：`usb_bulk_msg` 完成得快**不等于**帧发出去了。用 `ping -M do 1472`、
DHCP，或"在开放网络上发 ARP 看网关是否应答"。注意未加密裸帧判据在 WPA2 链路上无效（会被 AP
丢弃），同一链路普通 `ping` 与 ARP 却正常。

## 其它

- WPA2-Enterprise（802.1X）未验证：实验环境没有 802.1X AP。
- 驱动源码中部分中文注释在早期编辑中损坏成乱码，不影响编译。
- 0.x 阶段的模块参数、`/dev/zt9612` 协议可能变化。

## 稳定性怎么测

日常使用更关心丢包与延迟，这两个数不需要厂商资料就能测，也能区分"慢"和"不稳"：

```bash
IFACE=$(ls /sys/class/net | grep -E '^(wlx|wlan)' | head -1)   # 接口名按 MAC 生成
ping -I "$IFACE" -c 100 "$(ip route show default | awk '{print $3; exit}')"   # 到网关
ping -I "$IFACE" -c 100 223.5.5.5                                            # 到公网
```

两者差别大说明问题在无线段而不是出口。看丢包率、平均延迟、mdev/jitter 三个数；报问题时附
`uname -r`、`iw dev "$IFACE" link` 与距离环境。

## 遇到问题去哪里问

先查 [FAQ.md](FAQ.md)；仍未解决就到
[Issues](https://github.com/Ohhu/zt9612-heartbeat/issues) 反馈，请附 `lsusb -d 350b:`、
`uname -r` 与 dmesg 全段。无线问题几乎都能从这三样看出来。

```bash
sudo ./scripts/collect-debug-info.sh   # 生成 zt9612-debug-<时间戳>.txt，直接附到 issue
```

它只读、不改配置，收集系统/设备/模块/固件校验/参数/接口/dmesg 十来项。贴之前自己过一遍：
dmesg、ip、iw 的输出里可能有真实 MAC、SSID 与主机名。
