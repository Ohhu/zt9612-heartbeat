# 来源、修改与许可

## 上游

- 仓库：[Spkicn/zt9612-linux](https://github.com/Spkicn/zt9612-linux)
- 快照：`main` 分支归档，提交 `9b5dcfe1c0855b920c51feb143e0391e10ca7792`（2026-10-08），
  `dkms.conf` 中 `PACKAGE_VERSION` 为 0.3.2。归档于 2026-11-12 取得，该提交当时是 `main` 的
  最新提交。
- 许可：GPL-2.0-only，全文见 [LICENSE](LICENSE)

上游驱动是针对 `350b:9612`（ZTOP / 兆通微 ACEV100）的独立实现：仅依据设备对外可观测的行为
（USB 描述符、总线上的请求与响应序列）以及公开的 CEVA RivieraWaves rwnx 消息框架编写，仓库内
不含厂商源码、反编译产物或厂商二进制。版权归上游作者所有。

## 本仓库的修改（2026-11-13）

GPL-2.0 要求修改过的文件带显著修改声明与日期，逐文件对应关系如下：

| 文件 | 状态 |
|---|---|
| `driver/zt9612.c` | **已修改**（+104 / −17 行）。文件头注释里带修改声明；改动内容是心跳周期与固定节拍、心跳补发点前移、扫描分片睡眠、卸载收尾加固，明细见 [docs/CHANGELOG.md](docs/CHANGELOG.md) |
| `driver/Makefile` | 未改内容，仅把 Secure Boot 文档指向从 `README.md` 改为 `docs/INSTALL.md` |
| `dkms.conf` | 已修改：`PACKAGE_VERSION` 改为 `0.3.2-hb1`，内核验证清单补入 7.0.0-42 |
| `zt9612.conf` | 已修改：新增 `hb_interval_ms` 说明，并把指向 `README.md` 的章节名改成实际名称 |
| `docs/INSTALL.md` `docs/TROUBLESHOOTING.md` `docs/PARAMETERS.md` | 上游文档，已修改：路径改为 `docs/` 布局、心跳周期与窗口的表述更新、参数表新增 `hb_interval_ms`、删掉指向未公开内部资料的引用 |
| `firmware/README.md` `supported-device-IDs` | 上游文档，已修改：事实更新与删减 |
| `README.md` `NOTICE.md` `docs/FAQ.md` `docs/CHANGELOG.md` `.gitignore` | 本仓库重写 |
| 其余（`scripts/`、`install-driver.sh`、`uninstall-driver.sh`、`dkms-make.sh`、`LICENSE`、`.gitattributes`） | 与上游一致 |

原上游树中的 `ci/`、`.github/`、`AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md`、`SECURITY.md`、
`MAINTAINERS`、上游 `CHANGELOG.md`、`FAQ.md`、`driver/zt9612_fw.c`、`.checkpatch.conf`、
`.editorconfig` 没有进入本仓库：它们分别是上游 CI 配置、开发约定（引用上游未公开的 `docs/`、
`tools/`）、贡献流程、仅作对照的源码与上游本地工具链配置。
需要这些内容请取上游仓库。

源码注释中出现的 `re/`、`docs/`、`tools/` 路径指向上游未公开的内部资料，本仓库不包含。

## 厂商材料

厂商固件 `zt9612_fw.bin`、`zt9612_settings.bin` 与 Windows 驱动均不入库，也不随发行版分发；
取得方式见 [firmware/README.md](firmware/README.md)。

与芯片厂商（ZTOP / 山东兆通微电子）无隶属关系，也未获其背书；厂商名与型号仅用于说明兼容性。
