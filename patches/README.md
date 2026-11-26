# 本地补丁

编译流程会在克隆上游源码、执行 `make defconfig` 之前，按文件名字典序把本目录下所有
`*.patch` 依次以 `git apply --3way` 应用。

## 当前补丁

`0001-mediatek-add-360t7-uboot-layout.patch`：

1. 新增 `target/linux/mediatek/dts/mt7981b-qihoo-360t7-uboot.dts`
   —— 108M 自定义 U-Boot 布局：`bl2(1M) / u-boot-env(512K) / Factory(2M) / fip(2M) /
   ubi(108M @0x580000) / stock-config(1M) / stock-factory(512K) / stock-log(4M)`，
   UBI 内为 `kernel` + `rootfs` 卷，内核经 ubiblock 挂载 squashfs。
2. 在 `target/linux/mediatek/image/filogic.mk` 末尾注册设备 `qihoo_360t7-uboot`：
   `IMAGES := sysupgrade.bin factory.bin`，
   `IMAGE/factory.bin := append-ubi`，`IMAGE/sysupgrade.bin := sysupgrade-tar`，
   `IMAGE_SIZE := 110592k`（108M）。

## 新增补丁

- 补丁必须是标准 `git diff` / `git format-patch` 格式，相对上游仓库根目录；
- 文件名建议带序号：`0001-xxx.patch`、`0002-yyy.patch`；
- 修改设备 DTS、内核配置、`filogic.mk` 等都可以放在这里。

生成补丁示例（在 openwrt 源码目录内修改后）：

```bash
git diff > ../patches/0002-my-change.patch
```
