# ImmortalWrt 360T7 U-Boot 大分区固件编译仓库

基于 [chasey-dev/immortalwrt-mt798x-rebase](https://github.com/chasey-dev/immortalwrt-mt798x-rebase)
（ImmortalWrt 25.12 + MTK OpenWrt Feeds 补丁）的 **360T7（MT7981B，128M SPI-NAND）**
云编译仓库，默认产物适配 **hanwckf/bl-mt798x 的 108M 自定义 U-Boot（大分区）布局**，
输出 **`*-squashfs-factory.bin`** 和 **`*-squashfs-sysupgrade.bin`** 两类固件。

> 上游仓库自带的 `qihoo_360t7` 是 OpenWrt U-Boot 布局（FIT / `sysupgrade.itb`），
> 与 hanwckf 108M 自定义 U-Boot **不兼容**。本仓库通过补丁新增了
> `qihoo_360t7-uboot` 设备：UBI 内使用 `kernel` / `rootfs` 卷 + ubiblock 启动，
> 对应 MTK U-Boot 的 tar / UBI 固件格式。

---

## 一、三种布局与产物

| layout | 适用 U-Boot | 产物 | 说明 |
| --- | --- | --- | --- |
| **`uboot-108m`（默认）** | hanwckf/bl-mt798x「108M 大分区」自定义 U-Boot（2022/2023 版均可） | `*-squashfs-factory.bin`、`*-squashfs-sysupgrade.bin` | 本仓库补丁新增的 `qihoo_360t7-uboot` 设备，kernel/rootfs UBI 卷 |
| `fit-108m` | OpenWrt/ImmortalWrt 官方 U-Boot（本仓库 `fit-108m` 构建出的 `-bl31-uboot.fip`） | `*-squashfs-sysupgrade.itb`、`-bl31-uboot.fip`、`-preloader.bin` | 上游原生 `qihoo_360t7` 设备，FIT + fitblk |
| `ubi-127m` | 新版 OpenWrt 全 UBI 布局 U-Boot（127M） | `*-squashfs-sysupgrade.itb` | 上游 `qihoo_360t7-ubi` 设备 |

> ⚠️ 三种布局互不兼容，**不要跨布局强刷**（`sysupgrade -F` 也不行）。

### 为什么 108M U-Boot 要用 `factory.bin` / `sysupgrade.bin`？

- `*-sysupgrade.bin`：本质是 tar 包，内含 `kernel` 与 `rootfs` 镜像；
  2022 年之后的 MTK U-Boot 可直接用 `mtkupgrade fw` 或 Web 页面刷入，系统内升级也用它。
- `*-factory.bin`：直接把 `kernel` + `rootfs` 和 UBI 头打包成原始 UBI 镜像，
  按原样写入闪存，适合首次刷写（会重建 UBI，清掉原厂残留数据）。
- `*-sysupgrade.itb`：FIT 格式。**旧版 MTK U-Boot（如 bl-mt798x 20220606）没有 FIT 支持，会刷入失败**；
  只有 2025 年后的 MTK U-Boot 或 OpenWrt 官方 U-Boot 才支持。

参考：[MediaTek Filogic 常见固件格式说明](https://blog.imouto.in/post/openwrt/2025/fw-layouts-on-mtk-filogic/)。

---

## 二、108M 布局分区表

与 `hanwckf/bl-mt798x` 的 `mt7981_360t7_defconfig` 中
`MTDPARTS_DEFAULT`（`1024k(bl2),512k(u-boot-env),2048k(factory),2048k(fip),108M(ubi),1M(stock-config),512k(stock-factory),4M(stock-log)`）
逐项一致：

| 分区 | 起始偏移 | 大小 | 说明 |
| --- | --- | --- | --- |
| `bl2` | `0x000000` | 1 MiB | preloader（只读） |
| `u-boot-env` | `0x100000` | 512 KiB | U-Boot 环境变量（只读） |
| `Factory` | `0x180000` | 2 MiB | 出厂校准 / MAC（只读） |
| `fip` | `0x380000` | 2 MiB | U-Boot FIP（只读） |
| `ubi` | `0x580000` | **108 MiB** | 固件区，内含 `kernel`、`rootfs`、`rootfs_data` 卷 |
| `stock-config` | `0x7180000` | 1 MiB | 原厂残留（只读） |
| `stock-factory` | `0x7280000` | 512 KiB | 原厂残留（只读） |
| `stock-log` | `0x7300000` | 4 MiB | 原厂残留（只读） |

启动流程：U-Boot 从 `kernel` 卷读取 FIT 内核引导，内核根文件系统为 `rootfs` 卷的
squashfs（ubiblock），`rootfs_data` 卷作 overlay。

---

## 三、仓库实现了什么

- [`patches/0001-mediatek-add-360t7-uboot-layout.patch`](patches/0001-mediatek-add-360t7-uboot-layout.patch)：
  新增 `mt7981b-qihoo-360t7-uboot.dts`（108M 分区、`kernel`/`rootfs` 卷、ubiblock 启动、MAC/EEPROM 取自 Factory），
  并在 `filogic.mk` 注册设备 `qihoo_360t7-uboot`：
  `IMAGES := sysupgrade.bin factory.bin`、`append-ubi` 生成 factory、`sysupgrade-tar` 生成 sysupgrade、
  `IMAGE_SIZE := 110592k`（108M）。
- 云编译流程：克隆上游 → 应用补丁 → 选配设备 → feeds → 下载 → 编译 → 发布；
  每天检查上游更新，有新提交才编译。
- 布局可切换：`uboot-108m`（默认）/ `fit-108m` / `ubi-127m`。

---

## 四、快速开始

1. 右上角 **Fork** 本仓库（公开仓库 Actions 免费，推荐公开）。
2. 进入 Fork → **Actions** → `Build ImmortalWrt for 360T7 (U-Boot layout)` → **Run workflow**：
   - `layout` 保持默认 **`uboot-108m`**；
   - `source_ref`、`extra_packages` 可留空；
   - 点击 **Run workflow** 开始编译（首次约 2–4 小时）。
3. 编译完成后：Actions 页面下载 **Artifacts**，或到 **Releases** 下载正式产物。

---

## 五、产物说明（uboot-108m）

`bin/targets/mediatek/filogic/` 下：

| 文件 | 用途 |
| --- | --- |
| `...-qihoo_360t7-uboot-squashfs-factory.bin` | **首次刷写**：U-Boot Web 页面上传，重建 UBI |
| `...-qihoo_360t7-uboot-squashfs-sysupgrade.bin` | **系统内升级**：U-Boot Web 页面或 LuCI/SSH 升级 |
| `...-manifest` | 固件包清单 |
| `config.buildinfo` / `feeds.buildinfo` | 编译配置与 feed 版本 |
| `SHA256SUMS` | 校验和 |

---

## 六、刷机步骤（108M 自定义 U-Boot）

### 0. 前提

- 360T7 已刷入 **hanwckf/bl-mt798x 108M 大分区 U-Boot**（原厂 / 36M 分区设备需先用
  TTL 或免拆方式刷入对应 U-Boot，本仓库不提供该步骤，风险自负）。

### 1. 首次刷入（U-Boot Web 恢复）

1. 网线连接 360T7 的 **LAN 口**，电脑网卡设为静态 IP `192.168.1.2/24`；
2. 设备断电，按住 **reset** 键上电约 5 秒后松开（启动失败时 U-Boot 也会自动进入）；
3. 浏览器打开 `http://192.168.1.1`，上传 `*-squashfs-factory.bin`，确认刷写；
4. 等待写入并自动重启，首次启动约 1–3 分钟；默认 LuCI `http://192.168.1.1`。

### 2. 系统内升级

- **LuCI**：系统 → 备份/升级 → 上传 `*-squashfs-sysupgrade.bin`（按需保留配置）；
- **SSH**：`sysupgrade -v /tmp/*-sysupgrade.bin`（`-n` 不保留配置）；
- **U-Boot Web**：也可以直接上传 `*-squashfs-sysupgrade.bin` 刷写。

### 3. 验证

```bash
cat /proc/mtd
# mtd4: 0006c00000 00020000 "ubi"     # 108M = 0x6c00000
ubinfo -a                              # 可见 kernel / rootfs / rootfs_data 卷
```

### 4. 救砖

- 能进 U-Boot Web：重新上传 `factory.bin` / `sysupgrade.bin`；
- 不能进：TTL（115200 8N1）配合 `mtk_uartboot` 引导 BL2 后再刷。

---

## 七、自定义编译

| 需求 | 做法 |
| --- | --- |
| 加装软件包 | Run workflow 时填 `extra_packages`，如 `luci-app-ttyd` |
| 默认集成软件包 | 在 `configs/360t7-uboot-108m.config` 追加 `CONFIG_PACKAGE_xxx=y` |
| 修改内核/设备树 | 在 `patches/` 放 `0001-xxx.patch`（按文件名顺序 `git apply --3way`） |
| 切换布局 | workflow 的 `layout` 选 `fit-108m` 或 `ubi-127m` |
| 定时编译 | 已内置每天北京时间 02:00 检查上游 |

> configs/*.config 可以直接用 make menuconfig 导出的完整 .config。
> workflow 在 make defconfig 之前会删除其中的 CONFIG_TARGET_* 目标/设备行，
> 并按所选 layout 强制设备（uboot-108m 固定为 qihoo_360t7-uboot），
> 因此配置里只需要保留软件包选项；从 stock/FIT 设备导出的配置也能直接复用。

> 当前默认构建**不生成 U-Boot 文件**（`-bl31-uboot.fip` / `-preloader.bin`），
> 因为你已经刷好 108M 自定义 U-Boot，不需要再用 OpenWrt U-Boot 的 FIP 覆盖它。

---

## 八、本地编译

建议 Ubuntu 22.04/24.04 或 Debian 12，磁盘 ≥ 30 GB，内存 ≥ 4 GB。

```bash
sudo apt update
sudo apt install -y build-essential clang flex bison g++ gawk gcc-multilib g++-multilib \
  gettext git libncurses-dev libssl-dev python3-setuptools rsync swig unzip zlib1g-dev file wget

git clone https://github.com/<你的用户名>/immortalwrt-360t7-uboot.git
cd immortalwrt-360t7-uboot
git clone -b 25.12 --depth 1 https://github.com/chasey-dev/immortalwrt-mt798x-rebase.git openwrt

for p in patches/*.patch; do [ -e "$p" ] && git -C openwrt apply --3way "$(realpath "$p")"; done

cd openwrt
./scripts/feeds update -a
./scripts/feeds install -a
cp ../configs/360t7-uboot-108m.config .config
make defconfig
make -j"$(nproc)" download
make -j"$(nproc)" V=s
```

产物在 `openwrt/bin/targets/mediatek/filogic/`。

---

## 九、上游与版本

- 上游：`chasey-dev/immortalwrt-mt798x-rebase`，分支 `25.12`（ImmortalWrt 25.12 + MTK feeds 补丁）；
- 本仓库只做配置、补丁、编译与发布，不改动上游原有设备逻辑；
- Release 会记录上游提交号，`.last_sync_commit` 用于定时任务去重。

---

## 十、FAQ

**Q：为什么默认不是 `sysupgrade.itb`？**
A：108M 自定义 U-Boot（尤其是 bl-mt798x 20220606）不支持 FIT 固件，刷 `itb` 会失败；
`factory.bin` / `sysupgrade.bin` 才是它认识的格式。如果你的 U-Boot 是 2025 年后支持 FIT 的版本，
可以选 `fit-108m`，但那属于 OpenWrt U-Boot 布局，不能和 108M 自定义布局混用。

**Q：能刷回 360 官方固件吗？**
A：可以，需官方固件镜像并通过 U-Boot/TTL 刷写；本仓库不提供官方固件。

**Q：编译失败怎么办？**
A：Actions 会附带 `build-logs-*`（`build.log`、`logs/`）。上游下载偶发失败时重跑即可。

---

## 十一、免责声明

- 固件版权归 ImmortalWrt / OpenWrt 及各上游项目（GPL-2.0）；
- 刷机有风险（变砖、丢数据、失保），请自行承担；刷机前务必备份原厂 factory 分区；
- 仓库脚本与文档以 MIT 发布，不提供任何担保。
