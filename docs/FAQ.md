# FAQ

只列装机和日常使用会撞上的问题。装完能不能用、支持哪些卡见
[README.md](../README.md)，排障细节见 [TROUBLESHOOTING.md](TROUBLESHOOTING.md)。

## 插上只看到一个光驱（`350b:f179`），没有网卡？

出厂默认就是驱动光盘模式，需要一次标准 SCSI 弹出才切到网卡模式（与 Windows 上 `auto_load.exe`
做的事等价，不是厂商私有命令）：

```bash
sudo eject /dev/sr0
sudo usb_modeswitch -K -W -v 350b -p f179
```

一次性配置让插入时自动切换：`sudo ./scripts/setup-mode-switch.sh`。切换后芯片 ROM 只在数秒内
接受固件下载，所以切完要尽快加载驱动 —— udev 自动切换就是为此存在。

## 仓库里为什么没有 `zt9612_fw.bin`？

固件版权属厂商，不再分发。从自己网卡的配套驱动盘里取（Windows 下运行 `auto_load.exe` 后，
安装目录里通常有这两个文件），或向模组厂商索取 Linux/Android 驱动包。取得后用
`sudo ./scripts/install-firmware.sh <目录>` 安装，脚本校验 SHA-256。

## 用着用着又变回 `350b:f179` 光驱了？

固件看门狗复位：主机超过约 5 秒没发心跳（`0x05c2`）。本驱动默认 2 秒一次并按固定节拍补发；
自己写用户态脚本直连 `/dev/zt9612` 做实验时不经过驱动，需要自己维持心跳。

## 加载后 `dmesg` 出现 `-110`、`can't set config #1`、`Entity not found`？

芯片已挂死，只能冷启动复位：`systemctl poweroff` 断电后重新上电（热重启不断 VBUS，ROM 的
固件接收窗口不会重开），不必物理拔插。常见诱因是同一上电周期内重复灌固件、切模式后拖延太久、
不等 CFM 就连发 IPC。

## 可以 `rmmod` 之后换个模块再 `insmod` 吗？

不要。卸载时设备可能不应答在途 bulk-IN，驱动按设计"宁可泄漏实例也不 UAF"。换模块请重启机器；
`uninstall-driver.sh` 也只删文件，不执行 `rmmod`。

## 装好了下次开机怎么没自动加载？

`install-driver.sh` 默认写 `blacklist zt9612`，只挡自动加载，手动 `modprobe zt9612` 可用。
要自动加载加 `--enable-autoload` —— 但"插卡 + 开机自动加载"这条完整路径上游尚未复验。

## 支持 DKMS 吗？内核升级后怎么办？

支持。`sudo ./install-driver.sh --dkms`，或 `sudo dkms install .`。`dkms.conf` 在仓库根目录，
版本 `0.3.2-hb1`，装到 `/lib/modules/<内核>/updates/dkms/`，内核升级后自动重建。Secure Boot
机器还要处理签名：dkms 不会给压缩后的 `.ko.zst` 签名，需就地补签，命令见
[INSTALL.md](INSTALL.md)「Secure Boot 签名」。
