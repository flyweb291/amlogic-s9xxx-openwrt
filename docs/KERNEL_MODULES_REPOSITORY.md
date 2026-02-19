# 关于内核模块软件源 (kmods) 的说明

## ❓ 问题：Action 是否会构建与内核匹配的 kmods 软件源？

**简短答案：❌ 目前不会**

当前的 GitHub Actions 工作流（compile-kernel.yml）只编译内核包，**不会自动创建与内核匹配的 kmods 软件源（opkg package repository）**。

## 📋 当前状况分析

### 当前 compile-kernel.yml 做什么

```yaml
- name: Compile the kernel [ ${{ inputs.kernel_version }} ]
  uses: ophub/amlogic-s9xxx-armbian@main
  with:
    build_target: kernel
    kernel_source: ${{ inputs.kernel_source }}
    kernel_version: ${{ inputs.kernel_version }}
    kernel_package: ${{ inputs.kernel_package }}
    # ... 其他参数
```

**当前输出**：
1. ✅ 编译好的内核包（boot-xxx.tar.gz, dtb-xxx.tar.gz, modules-xxx.tar.gz）
2. ✅ 上传到 Release（tag: kernel_stable）
3. ❌ **不包含** opkg 软件包索引
4. ❌ **不包含** kmod-* 软件包（.ipk 格式）
5. ❌ **不包含** Packages 索引文件

### 用户当前面临的限制

在使用 OpenWrt 系统时，用户遇到以下问题：

**场景 1：安装内核模块**
```bash
# 用户想安装某个内核模块
opkg update
opkg install kmod-usb-net-rndis

# 结果：
# ❌ 软件包不存在
# ❌ 或者版本不匹配（kernel vermagic mismatch）
```

**场景 2：内核版本不匹配**
```bash
# OpenWrt 官方源的 kmod 包与自编译内核不匹配
# 错误示例：
# kernel module is missing a version magic,
# or kernel has one and module does not
```

**原因**：
- OpenWrt 官方 opkg 源中的 kmod-* 包与官方内核匹配
- 自编译的内核有不同的版本字符串和配置
- 没有对应的 kmod 软件源

---

## 🔍 什么是 kmods 软件源？

### OpenWrt 软件包类型

OpenWrt 有两类主要软件包：

1. **普通软件包** - 与内核无关
   - 例如：luci, nano, curl, wget
   - 可以从任何 OpenWrt 版本的软件源安装

2. **内核模块包（kmod-*）** - 与内核版本强绑定
   - 例如：kmod-usb-storage, kmod-fs-ext4, kmod-wireguard
   - **必须**与运行的内核版本完全匹配
   - 文件格式：.ipk

### kmods 软件源结构

完整的 kmods 软件源包含：

```
kmods/
├── 6.12.8/                          # 内核版本
│   ├── kmod-usb-storage_6.12.8-1_aarch64.ipk
│   ├── kmod-fs-ext4_6.12.8-1_aarch64.ipk
│   ├── kmod-wireguard_6.12.8-1_aarch64.ipk
│   ├── ... (更多 kmod-* 包)
│   ├── Packages                     # 软件包索引
│   └── Packages.gz                  # 压缩的索引
└── 6.18.3/                          # 另一个内核版本
    ├── kmod-*.ipk
    ├── Packages
    └── Packages.gz
```

### Packages 索引文件示例

```
Package: kmod-usb-storage
Version: 6.12.8-1
Depends: kernel (=6.12.8-1-1234567890), kmod-scsi-core, kmod-usb-core
Section: kernel
Architecture: aarch64
Installed-Size: 45678
Filename: kmod-usb-storage_6.12.8-1_aarch64.ipk
Size: 12345
SHA256sum: abc123...
Description: Kernel module for USB storage support
```

---

## 🚧 为什么当前没有构建 kmods 源？

### 技术原因

1. **不同的构建流程**
   - **内核编译**：只编译内核本身和基本模块
   - **kmod 包构建**：需要完整的 OpenWrt 构建系统

2. **OpenWrt SDK 要求**
   - 构建 kmod-* 包需要 OpenWrt SDK
   - SDK 包含：
     - 内核头文件
     - 编译工具链
     - 包构建脚本（Makefiles）
     - 依赖关系定义

3. **存储和带宽成本**
   - 每个内核版本的 kmods 源约 100-500MB
   - 支持多个内核版本会快速增加存储需求
   - GitHub Release 有 2GB 单文件限制

4. **维护复杂度**
   - 需要为每个内核版本构建完整的 kmod 集合
   - 需要处理依赖关系
   - 需要更新和维护软件包索引

---

## 💡 解决方案和替代方法

### 方案 1：使用官方 OpenWrt 软件源（限制）

**适用情况**：使用官方内核或版本匹配

```bash
# 在 /etc/opkg/distfeeds.conf 中配置
src/gz openwrt_kmods https://downloads.openwrt.org/releases/24.10/targets/armsr/armv8/kmods/6.1.xxx/
```

**限制**：
- ❌ 只适用于官方内核版本
- ❌ 不适用于自编译内核（版本字符串不匹配）
- ❌ 不适用于带自定义补丁的内核

### 方案 2：手动编译和安装内核模块

**步骤**：

1. **获取内核源码和配置**
```bash
# 下载与运行内核匹配的源码
wget https://github.com/ophub/kernel/releases/...
```

2. **编译特定模块**
```bash
cd linux-6.12.8
make oldconfig
make modules_prepare
make M=drivers/usb/storage
```

3. **手动安装模块**
```bash
cp drivers/usb/storage/usb-storage.ko /lib/modules/6.12.8/
depmod -a
modprobe usb-storage
```

**限制**：
- ⚠️ 需要技术知识
- ⚠️ 每次需要模块都要重复
- ⚠️ 无法使用 opkg 管理

### 方案 3：构建包含所需模块的完整固件

**推荐方法**：在编译 OpenWrt 时包含所有需要的模块

在 `.config` 文件中：
```makefile
# 网络模块
CONFIG_PACKAGE_kmod-usb-net-rndis=y
CONFIG_PACKAGE_kmod-usb-net-asix=y

# 文件系统模块
CONFIG_PACKAGE_kmod-fs-ext4=y
CONFIG_PACKAGE_kmod-fs-ntfs3=y

# VPN 模块
CONFIG_PACKAGE_kmod-wireguard=y
```

**优点**：
- ✅ 所有模块都内置在固件中
- ✅ 版本完全匹配
- ✅ 不需要额外软件源

**缺点**：
- ⚠️ 固件体积增大
- ⚠️ 添加新模块需要重新编译固件

### 方案 4：创建自定义 kmods 软件源（高级）

**如果确实需要 kmods 源，需要以下步骤**：

#### 步骤 1：修改 compile-kernel.yml

需要添加构建 kmod 包的步骤：

```yaml
- name: Build kernel modules packages
  run: |
    # 使用 OpenWrt SDK 构建 kmod 包
    wget https://downloads.openwrt.org/releases/.../openwrt-sdk-...
    tar xf openwrt-sdk-*
    cd openwrt-sdk-*

    # 配置使用自编译的内核
    cp /path/to/compiled/kernel/.config .config

    # 构建所有 kmod 包
    make package/kernel/linux/compile

    # 生成 Packages 索引
    make package/index
```

#### 步骤 2：创建软件源结构

```yaml
- name: Create kmods repository
  run: |
    mkdir -p kmods/$KERNEL_VERSION
    cp bin/targets/*/packages/kmod-*.ipk kmods/$KERNEL_VERSION/
    cd kmods/$KERNEL_VERSION

    # 生成 Packages 索引
    $STAGING_DIR_HOST/bin/ipkg-make-index . > Packages
    gzip -k Packages
```

#### 步骤 3：上传到 Release 或托管服务

```yaml
- name: Upload kmods repository
  run: |
    # 上传到 GitHub Release
    gh release upload kernel_stable kmods-$KERNEL_VERSION.tar.gz

    # 或上传到其他服务器
    rsync -av kmods/ user@server:/path/to/repo/
```

#### 步骤 4：配置 OpenWrt 使用自定义源

在 OpenWrt 系统中：

```bash
# 编辑 /etc/opkg/customfeeds.conf
echo "src/gz custom_kmods https://github.com/USER/REPO/releases/download/kernel_stable/kmods/6.12.8" >> /etc/opkg/customfeeds.conf

# 更新并安装
opkg update
opkg install kmod-usb-storage
```

---

## 📊 各方案对比

| 方案 | 难度 | 灵活性 | 存储需求 | 用户便利性 |
|------|------|--------|----------|------------|
| **使用官方源** | 低 | 低 | 无 | 高（如果版本匹配） |
| **手动编译模块** | 高 | 高 | 低 | 低 |
| **固件内置模块** | 中 | 中 | 中 | 高 |
| **自建 kmods 源** | 高 | 高 | 高 | 高 |

---

## 🎯 推荐做法

### 对于大多数用户

**✅ 推荐：在编译固件时包含所需模块**

修改 `config/xxx/config` 文件，添加：

```makefile
# USB 网络
CONFIG_PACKAGE_kmod-usb-net=y
CONFIG_PACKAGE_kmod-usb-net-rndis=y
CONFIG_PACKAGE_kmod-usb-net-cdc-ether=y

# 文件系统
CONFIG_PACKAGE_kmod-fs-ext4=y
CONFIG_PACKAGE_kmod-fs-ntfs3=y
CONFIG_PACKAGE_kmod-fs-exfat=y

# 无线
CONFIG_PACKAGE_kmod-cfg80211=y
CONFIG_PACKAGE_kmod-mac80211=y

# VPN
CONFIG_PACKAGE_kmod-wireguard=y
CONFIG_PACKAGE_kmod-tun=y
```

### 对于高级用户/项目维护者

如果确实需要 kmods 软件源，可以：

1. **创建独立的工作流** `build-kmods-repository.yml`
2. **使用 OpenWrt SDK** 构建 kmod 包
3. **托管软件源** 在 GitHub Pages 或其他服务
4. **文档化配置** 让用户知道如何添加自定义源

---

## 🔮 未来改进建议

### 对项目的建议

如果要添加 kmods 软件源功能，建议：

1. **新建专门的工作流**
   - `build-kmods-repository.yml`
   - 与 `compile-kernel.yml` 分离，按需运行

2. **使用 OpenWrt SDK**
   - 不需要完整编译 OpenWrt
   - 只构建内核模块包
   - 速度更快，资源消耗更少

3. **选择性构建**
   - 只构建常用的 kmod 包
   - 减少存储需求
   - 示例：网络、文件系统、VPN 相关模块

4. **提供配置选项**
   ```yaml
   workflow_dispatch:
     inputs:
       build_kmods:
         description: "Build kmods repository"
         type: boolean
         default: false
       kmod_selection:
         description: "Kernel modules to build"
         type: choice
         options:
           - essential  # 仅核心模块
           - network    # 网络相关
           - all        # 所有模块
   ```

5. **托管方案**
   - GitHub Pages（免费，2GB 限制）
   - GitHub Releases（按版本分发）
   - 第三方对象存储（S3, OSS 等）

---

## 📚 参考资源

### OpenWrt 文档

- [Opkg 包管理](https://openwrt.org/docs/guide-user/additional-software/opkg)
- [使用 SDK](https://openwrt.org/docs/guide-developer/toolchain/using_the_sdk)
- [软件包索引](https://openwrt.org/docs/guide-developer/packages)
- [内核模块](https://openwrt.org/docs/guide-developer/kernel)

### 相关项目

- [ophub/kernel](https://github.com/ophub/kernel) - 预编译内核
- [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) - OpenWrt 源码
- [openwrt/openwrt](https://github.com/openwrt/openwrt) - 官方 OpenWrt

---

## ❓ 常见问题

### Q1: 为什么安装 kmod-* 包时提示版本不匹配？

**A**: 因为自编译内核的版本字符串与官方源的 kmod 包不匹配。解决方法：
- 在编译固件时内置所需模块
- 或使用方案 4 构建匹配的 kmods 源

### Q2: 可以混用不同版本的内核和 kmod 包吗？

**A**: ❌ 不可以。内核模块必须与内核版本完全匹配，包括：
- 主版本号（如 6.12.8）
- 配置选项
- 编译器版本
- 版本魔数（vermagic）

### Q3: 哪些模块最常用，应该内置？

**A**: 推荐内置以下模块：

**网络**：
- kmod-usb-net, kmod-usb-net-rndis
- kmod-pppoe, kmod-ppp
- kmod-tun（VPN 必需）

**文件系统**：
- kmod-fs-ext4（外置硬盘）
- kmod-fs-ntfs3（Windows 分区）
- kmod-fs-exfat（大U盘）

**无线**（如果硬件支持）：
- kmod-cfg80211, kmod-mac80211
- 特定芯片驱动

**存储**：
- kmod-usb-storage
- kmod-scsi-core

### Q4: 如何查看当前内核版本和已安装模块？

**A**:
```bash
# 查看内核版本
uname -r

# 查看已安装的 kmod 包
opkg list-installed | grep kmod-

# 查看已加载的模块
lsmod

# 查看模块详情
modinfo <module_name>
```

### Q5: 能否提供一个完整的 kmods 构建示例？

**A**: 以下是一个完整的工作流示例（需要根据实际情况调整）：

```yaml
name: Build kmods repository

on:
  workflow_dispatch:
    inputs:
      kernel_version:
        description: "Kernel version"
        required: true
        default: "6.12.8"

jobs:
  build-kmods:
    runs-on: ubuntu-24.04
    steps:
      - name: Download OpenWrt SDK
        run: |
          wget https://downloads.openwrt.org/releases/24.10/targets/armsr/armv8/openwrt-sdk-24.10-armsr-armv8_gcc-13.2.0_musl.Linux-x86_64.tar.xz
          tar xf openwrt-sdk-*.tar.xz

      - name: Configure SDK with custom kernel
        run: |
          cd openwrt-sdk-*
          # 配置使用自定义内核

      - name: Build kernel modules
        run: |
          cd openwrt-sdk-*
          make package/kernel/linux/compile -j$(nproc)

      - name: Generate package index
        run: |
          cd openwrt-sdk-*
          make package/index

      - name: Create repository structure
        run: |
          mkdir -p kmods/${{ inputs.kernel_version }}
          cp openwrt-sdk-*/bin/targets/*/packages/kmod-*.ipk kmods/${{ inputs.kernel_version }}/
          cd kmods/${{ inputs.kernel_version }}
          # 生成 Packages 文件

      - name: Upload to Release
        uses: ncipollo/release-action@main
        with:
          tag: kmods_${{ inputs.kernel_version }}
          artifacts: kmods-${{ inputs.kernel_version }}.tar.gz
```

---

## 📝 总结

**当前状态**：
- ❌ 项目不自动构建 kmods 软件源
- ✅ 只编译内核包
- ✅ 内核包可用于更新内核

**推荐做法**：
- 📦 在编译固件时包含所需的内核模块
- 🎯 这是最简单、最可靠的方法
- ✅ 避免版本不匹配问题

**高级需求**：
- 如需独立的 kmods 源，需要额外开发
- 可以参考本文档的方案 4
- 需要权衡开发成本和维护成本

---

**最后更新**: 2026-02-19
**适用版本**: 当前仓库所有工作流
