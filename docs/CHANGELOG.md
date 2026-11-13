# 变更日志

只记录本仓库相对上游的改动。上游的完整开发记录见
[Spkicn/zt9612-linux 的 CHANGELOG](https://github.com/Spkicn/zt9612-linux/blob/main/CHANGELOG.md)。

## 0.3.2-hb1

基线：上游快照 `9b5dcfe`（2026-10-08），`PACKAGE_VERSION = 0.3.2`。
改动文件：`driver/zt9612.c`（+104 / −17 行）；另有 `dkms.conf` 版本号、文档与 `zt9612.conf`
的参数说明，见 [NOTICE.md](../NOTICE.md)。

### 心跳：默认 2000 ms，按固定节拍补发

- `HB_INTERVAL_MS 5000` 改为 `HB_INTERVAL_DEFAULT_MS 2000`，新增模块参数 `hb_interval_ms`
  （int，0644，夹在 200–60000 ms），运行期可在 `/sys/module/zt9612/parameters/hb_interval_ms`
  调整，不持久。
- `zt_hb_work()` 改为按固定节拍排下一次：把本次执行耗时从下个周期里扣掉，最多扣一半。既避免
  周期随执行时间累加，也避免一次超长执行把间隔压到 0。
- 依据：usbmon 抓 809 秒共 200 次心跳，中位 5.12 s、p90 5.18 s、p99 5.48 s、最大 5.49 s，
  无一次短于 5 s；上游文档给出的窗口是"超过 5 秒不发心跳即复位"。

### 心跳补发点：从 CFM 等待循环挪到命令发出之前

- `zt_cmd()`、`zt_cmd_fifo()`、`zt_cmd_fifo_resp()` 三处删掉等 CFM 循环里的补发，改为在
  `zt_send()` 之前调用 `zt_hb_if_due()`。
- 原因：固件 IPC 是一问一答模型，命令在途时再塞一条心跳会占掉命令槽，原命令的 CFM 不再到达，
  现场表现为 `scan: timeout waiting for cfm 0x0023`。

### 扫描：驻留与信道稳定等待期间也补心跳

- 新增 `zt_hb_period()`、`zt_hb_if_due()`、`zt_msleep_hb()`；`zt_scan_work()` 里的
  `msleep(20)` 与 `msleep(ZT_SCAN_DWELL_MS)` 改成 `zt_msleep_hb()`（50 ms 分片，每片前置检查）。
- 原因：扫描持锁整轮最长可达数秒，心跳工作项拿不到锁。

### 卸载收尾：mac80211 反注册之后再取消工作项

- `zt_disconnect()` 在 `zt_mac_unregister()` 之后追加 `cancel_work_sync(&z->scan_work)`、
  `cancel_work_sync(&z->tx_work)`、`skb_queue_purge(&z->txq)`。
- `zt_mac_hw_scan()` 与 `zt_mac_wake_tx_queue()` 入口增加 `alive` 判断。
- 原因：反注册之前仍可能被回调把工作项重新排上，工作项会在函数末尾 `kfree(z)` 之后才执行。

### 实测

同一台机器、同一张卡、同一个 AP（5 GHz，-57 至 -66 dBm），Ubuntu 24.04 + 7.0.0-42：

| 指标 | 上游 0.3.2（3.5 小时） | 本版本（17 小时 11 分） |
|---|---|---|
| 芯片看门狗复位（退回 `350b:f179`） | 16 次 | 0 次 |
| USB 断开 | 16 次 | 1 次（开机模式切换） |
| `bulk OUT failed` | 多次 | 0 |
| `-110` / `-71` | 有 | 0 / 0 |
| oops / panic | 0 | 0 |

17 小时收发 RX 48.2 MB / 228,158 包、TX 763 KB / 7,935 包；扫描 64 次，其中 5 次扫描前置序列
失败，均在改动上线初期。期间 AP 自行把信道从 165 换到 36，驱动随频点变化关联回同一 BSSID，
用时约 4 秒。

4 小时 252 点采样：252 点全部 `default=1`、`wifi_def=1`、`usb=1`；250 点在同一 BSSID，
另 2 点在改动之前的窗口；到网关 3 个 ping 中 228 点全通、22 点丢 1 个、1 点丢 2 个、1 点全丢；
每小时一次的公网检查 4 次全部 HTTP 200。

### 未改动但已知

- 上传吞吐上限约 15 Mbit/s，在固件 TX 侧，主机侧改不动。
- 扫描前置序列偶发 CFM 超时（本机 64 次扫描 5 次），该轮扫描中止后由上层重试。
- 驱动源码中的历史乱码注释未清理。
