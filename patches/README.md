# 本地补丁

编译流程会在克隆上游源码、执行 `make defconfig` 之前，按文件名字典序把本目录下所有
`*.patch` 依次以 `git apply --3way` 应用。

- 补丁必须是标准 `git diff` / `git format-patch` 格式，相对上游仓库根目录；
- 文件名建议带序号：`0001-xxx.patch`、`0002-yyy.patch`；
- 修改设备 DTS、内核配置、`target/linux/mediatek/image/filogic.mk` 等都可以放在这里；
- 本仓库默认不需要任何补丁：上游已内置 360T7 108M U-Boot 布局支持。

生成补丁示例（在 openwrt 源码目录内修改后）：

```bash
git diff > ../patches/0001-my-change.patch
```
