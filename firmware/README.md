# 固件目录（不含固件二进制）

`zt9612_fw.bin` 与 `zt9612_settings.bin` 的版权属于芯片厂商（ZTOP / 山东兆通微电子），本仓库
没有再分发许可，因此不提供下载，只说明需要哪些文件、怎么校验、装到哪里。做法参照
[linux-firmware 的 WHENCE 约定](https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git)。

## 需要的文件

| 文件名 | 大小 | SHA-256 |
|---|---|---|
| `zt9612_fw.bin` | 219,076 字节 | `8562003c3558c7f608d5ff926d368d6de021c1ae12ac833978ea7c4e5a9ebe2d` |
| `zt9612_settings.bin` | 212 字节 | `58c2d73a4d2a1c1eb10a606a6e0bf6a201fb8f288fce1cbe0ce5364f7271fb46` |

两个文件都要放在 `/lib/firmware/` 下（不要放子目录）。`zt9612_fw.bin` 的容器头是 `"ZT"` +
`u16 0x9612`；如果拿到的是别的型号（例如 100 多 KB 的 `zt9611_fw.bin`），大小与哈希都对不上，
不能用。

## 从哪里取得

1. 网卡自带的驱动光盘。设备出厂默认就是这张光盘（`350b:f179`，卷标 "Wi-Fi6 Adapter"）。在
   Windows 上运行里面的 `auto_load.exe` 装好驱动后，到安装目录（通常 `C:\Program Files (x86)\`
   下）找这两个文件，也可以直接从光盘的 CAB 里解出来。
2. 向模组厂商索取：厂商有 Linux/Android 驱动包 `ZTOP_ACEV100_Android_wifi_bt_*.tar.gz`，内含
   `fw/zt9612_fw.bin`。模组型号 ZT9612U V1.0，平台 ACEV100。
3. 找卖你网卡的商家转原厂技术支持。这条看运气，无法保证。

## 安装

```bash
sudo ./scripts/install-firmware.sh /path/to/firmware-dir
# 或指定文件
sudo ./scripts/install-firmware.sh ~/zt9612_fw.bin ~/zt9612_settings.bin
```

脚本按 SHA-256 校验，只有哈希匹配才会安装到 `/lib/firmware/` 并以 0644 落地；不匹配会打印期望值
与实际值并返回非零。若 `/lib/firmware/` 下已有同名文件但哈希不符，脚本不会静默覆盖成功，需要
先把旧文件删掉。

## 许可

在取得明确的再分发许可之前，请不要把 `.bin` 提交到本仓库 —— `.gitignore` 已排除 `*.bin`。
将来若获得许可，请在此处补充文件名、来源产品、许可方与条款（单独的 `LICENSE.<vendor>`）以及
是否允许再分发。
