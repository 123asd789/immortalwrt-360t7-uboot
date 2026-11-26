# ImmortalWrt 360T7 U-Boot 大分区固件编译仓库

基于 [chasey-dev/immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase)
（ImmortalWrt 25.12 + MTK OpenWrt Feeds 补丁）的 **360T7（MT7981B，128M SPI-NAND）**
云编译仓库，默认产物适配 **hanwckf 360T7 U-Boot 的 108M 大分区布局**。

> 一句话：Fork 本仓库 → 在 Actions 里点一次「Run workflow」→ 到 Releases 下载
> `immortalwrt-mediatek-filogic-qihoo_360t7-squashfs-sysupgrade.itb`，进 U-Boot Web 恢复页刷入即可。

---

## 一、适配的分区布局

### 默认：`uboot-108m`（hanwckf U-Boot 108M 大分区）

与 `hanwckf/bl-mt798x` 中 `mt7981_360t7_defconfig` 的 `MTDPARTS_DEFAULT` 完全一致
（`1024k(bl2),512k(u-boot-env),2048k(factory),2048k(fip),108M(ubi),1M(stock-config),512k(stock-factory),4M(stock-log)`），
也与上游 `mt7981b-qihoo-360t7.dts` 的 `qihoo_360t7` 设备定义一一对应：

| 分区 | 起始偏移 | 大小 | 说明 |
| --- | --- | --- | --- |
| `bl2` | `0x000000` | 1 MiB | preloader（只读） |
| `u-boot-env` | `0x100000` | 512 KiB | U-Boot 环境变量 |
| `factory` | `0x180000` | 2 MiB | 出厂校准数据（只读） |
| `fip` | `0x380000` | 2 MiB | U-Boot FIP 镜像 |
| `ubi` | `0x580000` | **108 MiB** | 固件所在 UBI，`fit` 卷存放内核 + 根文件系统 |
| `stock-config` | `0x7180000` | 1 MiB | 原厂残留（只读） |
| `stock-factory` | `0x7280000` | 512 KiB | 原厂残留（只读） |
| `stock-log` | `0x7300000` | 4 MiB 起 | 原厂日志区（只读） |

固件只关心 `ubi` 分区：它从 `0x580000` 开始、大小 108 MiB，内核 FIT 镜像写入
`fit` UBI 卷，由 U-Boot 引导。**只要你的 360T7 刷的是这个 108M 布局的 U-Boot，本仓库产物即可直接使用。**

### 可选：`ubi-127m`（OpenWrt 新版全 UBI 布局）

上游 PR #22797 之后的新布局：`bl2` 占 `0x0-0x100000`，其余 127 MiB 整体为 UBI，
内含 `factory` / `fip` / `fit` / `ubootenv` / `ubootenv2` 卷。该布局需要先刷入配套的
`preloader.bin` + `bl31-uboot.fip`，**不能与 108M 布局混刷**。只有明确知道自己用的是新布局时才选它。

两套布局的配置分别位于 [`configs/360t7-uboot-108m.config`](configs/360t7-uboot-108m.config)
和 [`configs/360t7-ubi-127m.config`](configs/360t7-ubi-127m.config)，对应上游设备
`qihoo_360t7` 与 `qihoo_360t7-ubi`。

---

## 二、仓库做了什么

上游仓库本身已经包含 360T7 的设备定义与 108M 分区 DTS，这个仓库把「能编译、能刷机、
能长期跟进上游」固化下来：

- **GitHub Actions 云编译**：Ubuntu 22.04 runner 上完整编译，产物含 sysupgrade 固件、
  U-Boot FIP、preloader、manifest、`SHA256SUMS`；
- **按布局出包**：`workflow_dispatch` 可选 `uboot-108m` / `ubi-127m`，配置与设备自动对应；
- **自动跟进上游**：每天北京时间 02:00 检查上游提交，有更新才编译并发布 Release；
- **可扩展**：`patches/*.patch` 在编译前自动应用，`extra_packages` 输入可临时加包；
- **可复现**：Release 附带 `config.buildinfo` / `feeds.buildinfo` 与校验和。

---

## 三、快速开始

1. 右上角 **Fork** 本仓库到你的账号（公开仓库的 Actions 额度免费，推荐公开）。
2. 进入你的 Fork → **Actions** → 左侧 `Build ImmortalWrt for 360T7 (U-Boot 108M layout)` → 右侧 **Run workflow**。
   - `layout` 保持默认 `uboot-108m`；其它参数可留空；
   - 点击 **Run workflow** 开始编译（首次约 2–4 小时）。
3. 编译完成后：
   - Actions 页面可下载 **Artifacts**（保留 90 天）；
   - 或到 **Releases** 下载正式产物（长期保留）。

> 私有仓库会消耗 Actions 时长（免费额度 2000 分钟/月，一次编译约 2–4 小时），
> 且单次任务上限 6 小时。公开仓库无此顾虑，因此建议公开。

---

## 四、产物说明

以 `qihoo_360t7`（108M 布局）为例，`bin/targets/mediatek/filogic/` 下会有：

| 文件 | 用途 |
| --- | --- |
| `...-squashfs-sysupgrade.itb` | **刷机用固件**：U-Boot Web 恢复页刷写，或系统内升级 |
| `...-initramfs-recovery.itb` | 内存运行恢复系统（救砖/排障，重启不保留） |
| `...-bl31-uboot.fip` | U-Boot FIP 镜像（更新 Bootloader 用，谨慎） |
| `...-preloader.bin` | BL2 preloader（更新 Bootloader 用，谨慎） |
| `...manifest` | 本次固件包含的软件包清单 |
| `profiles.json` | 设备/镜像描述 |
| `config.buildinfo` / `feeds.buildinfo` | 编译配置与 feed 版本 |
| `SHA256SUMS` | 全部产物校验和 |

---

## 五、刷机指南（108M 布局）

### 0. 确认设备与布局

- 适用于 **360T7（MT7981B，128M NAND）**，且已刷入 **108M 大分区 U-Boot**（hanwckf bl-mt798x 系列）。
- 如果设备还在原厂固件 / 36M 分区，需要先通过 TTL（或免拆方式）刷入 108M 布局的 U-Boot。
  **本仓库不包含、也不负责 Bootloader 刷写步骤**，相关操作有变砖风险，请自行评估。

### 1. U-Boot Web 恢复页刷写

1. 网线连接 360T7 的 **LAN 口**，电脑网卡设为静态 IP `192.168.1.2`，掩码 `255.255.255.0`。
2. 设备断电，按住 **reset** 键再上电，约 5 秒后松开（启动失败时 U-Boot 也会自动进入 Web 恢复）。
3. 浏览器打开 `http://192.168.1.1`，选择固件文件
   `immortalwrt-mediatek-filogic-qihoo_360t7-squashfs-sysupgrade.itb`，上传并确认刷写。
4. 等待写入完成，设备自动重启；首次启动约 1–3 分钟，完成后 LuCI 默认地址
   `http://192.168.1.1`（或 DHCP 下发的网关地址）。

### 2. 系统内升级

设备已运行 ImmortalWrt 时：

- **LuCI**：系统 → 备份/升级 → 选择 `*-squashfs-sysupgrade.itb` → 按需勾选「保留配置」；
- **SSH**：把固件传到 `/tmp` 后执行 `sysupgrade -v /tmp/xxx-sysupgrade.itb`
  （保留配置；加 `-n` 表示不保留）。

### 3. 验证布局（刷机后）

```bash
cat /proc/mtd
# mtd5: 0006c00000 00020000 "ubi"    # 108M 布局应为 0x6c00000
```

### 4. 救砖

- 能进 U-Boot Web 恢复页：直接重新上传固件；
- 不能进：TTL 串口（115200 8N1）接 GND/TX/RX，配合 `mtk_uartboot` 引导 BL2，再按上面步骤刷写。

> ⚠️ **禁止跨布局强刷**。108M 与 127M 布局的 UBI 起点/大小不同，混刷会无法启动；
> 不要使用 `sysupgrade -F` 跳过平台校验。

---

## 六、自定义编译

| 需求 | 做法 |
| --- | --- |
| 加装软件包 | Run workflow 时填 `extra_packages`，如 `luci-app-ttyd luci-app-upnp` |
| 默认集成软件包 | 在 `configs/360t7-uboot-108m.config` 追加 `CONFIG_PACKAGE_xxx=y` |
| 修改内核/设备树 | 在 `patches/` 放 `0001-xxx.patch`，编译前会自动应用（见 [patches/README.md](patches/README.md)） |
| 切换上游分支 | `source_ref` 填其它分支（如 `master`，需自行确认可用性） |
| 定时自动编译 | 已内置每天 02:00（北京时间）检查上游，有新提交才编译 |

修改 workflow 或配置后，可直接推送触发（push 到 `main` 不会自动编译，需要手动 Run workflow，
或在仓库 Settings 里按需调整触发器）。

---

## 七、本地编译

建议 Ubuntu 22.04/24.04 或 Debian 12，磁盘 ≥ 30 GB，内存 ≥ 4 GB（推荐 8 GB）。

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib g++-multilib \
  gettext git libncurses-dev libssl-dev python3-setuptools rsync swig unzip zlib1g-dev file wget

git clone https://github.com/<你的用户名>/immortalwrt-360t7-uboot.git
cd immortalwrt-360t7-uboot
git clone -b 25.12 --depth 1 https://github.com/chasey-dev/immortalwrt-mt798x-rebase.git openwrt

# 应用本地补丁（没有可跳过）
for p in patches/*.patch; do [ -e "$p" ] && git -C openwrt apply "$(realpath "$p")"; done

cp configs/360t7-uboot-108m.config openwrt/.config
cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
make defconfig
make -j"$(nproc)" download
make -j"$(nproc)" V=s
```

产物在 `openwrt/bin/targets/mediatek/filogic/`。

---

## 八、上游与版本

- 上游：`chasey-dev/immortalwrt-mt798x-rebase`，分支 `25.12`（ImmortalWrt 25.12 + MTK feeds 补丁）；
- 本仓库不修改上游源码逻辑，只负责配置、补丁挂载、编译与发布；
- 编译通过后会在 Release 中记录上游提交号，`.last_sync_commit` 用于定时任务去重。

---

## 九、FAQ

**Q：为什么默认是 108M 而不是 127M？**
A：中文社区 360T7 最常见的 U-Boot 是 hanwckf 的 108M 大分区版本；两者 DTS 分区表逐字节一致，
直接选择上游 `qihoo_360t7` 设备即可，无需额外补丁。新布局用户改用 `ubi-127m` 即可。

**Q：能刷回 360 官方固件吗？**
A：可以，但需要 360 官方固件镜像并通过 U-Boot Web 恢复/TTL 刷写；本仓库不提供官方固件，也不对回刷后果负责。

**Q：编译失败怎么办？**
A：Actions 会附带 `build-logs-*` 产物（`build.log`、`logs/`）。另外上游 `dl/` 下载偶发失败时，
重跑 workflow 即可；持续失败请提 Issue 并附日志片段。

**Q：想加 OpenClash/PassWall 等插件？**
A：在 `extra_packages` 里加对应包名（需上游 feed 中存在）；如果来自第三方 feed，
请自行在编译流程中增加 feed 源（修改 `feeds.conf.default` 需放进 `patches/` 或改 workflow）。

---

## 十、免责声明

- 本项目仅提供编译流程与配置文件，固件版权归 ImmortalWrt / OpenWrt 及各上游项目（GPL-2.0）；
- 刷机有风险（变砖、丢数据、失去保修），请自行承担；刷机前务必备份原厂分区（factory 校准数据等）；
- 仓库脚本与文档以 MIT 许可发布，不提供任何明示或暗示担保。
