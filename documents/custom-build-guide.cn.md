# 自编译固件并添加第三方软件（以 OpenClash 为例）完全手册

> 适用仓库：[ophub/amlogic-s9xxx-openwrt](https://github.com/ophub/amlogic-s9xxx-openwrt)
> 适用对象：想在电视盒子（晶晨 / 瑞芯微 / 全志等 arm64 盒子）上使用自编译 OpenWrt 固件，并集成 OpenClash 等第三方插件的用户。

---

## 目录

1. [编译原理：先明白固件是怎么“变”出来的](#1-编译原理)
2. [三种编译方式对比](#2-三种编译方式对比)
3. [方式一：GitHub Actions 云编译（推荐）](#3-方式一github-actions-云编译推荐)
4. [核心：修改配置文件添加 OpenClash 等第三方软件](#4-核心修改配置添加第三方软件)
5. [方式二：本地完整编译](#5-方式二本地完整编译)
6. [方式三：本地只打包（remake）](#6-方式三本地只打包remake)
7. [刷机与安装](#7-刷机与安装)
8. [常见问题 FAQ](#8-常见问题-faq)

---

## 1. 编译原理

本仓库制作固件分为 **两个阶段**，理解这一点非常重要，因为**所有软件包（包括 OpenClash）都是在第一阶段加入的**：

```text
阶段一：编译通用系统（rootfs）
┌─────────────────────┐   源码(lede/immortalwrt/openwrt) + config 目录配置
│  编译 OpenWrt 源码    │ ────────────────────────────────►  openwrt-armsr-armv8-generic-rootfs.tar.gz
│  （软件包在这里加入）  │                                    （通用 rootfs 文件，不含盒子内核）
└─────────────────────┘

阶段二：打包成各盒子的专用固件
┌─────────────────────┐   rootfs.tar.gz + ophub/kernel 内核 + u-boot
│  打包（remake 脚本）  │ ────────────────────────────────►  openwrt_s905x3_6.1.x_日期.img.gz
│  （针对具体盒子型号）  │                                    （写入 U 盘/TF 卡刷机用）
└─────────────────────┘
```

- **阶段一**决定固件里“有什么软件”（OpenClash、主题、中文语言等都在这里集成）。
- **阶段二**决定固件“能跑在哪个盒子上、用什么内核”。同一份 rootfs 可以反复打包出 N 个型号的固件。
- 你日常改配置 99% 是在改阶段一的三个文件（见[第 4 节](#4-核心修改配置添加第三方软件)）。

---

## 2. 三种编译方式对比

| 方式 | 耗时 | 环境要求 | 适用场景 |
| ---- | ---- | ---- | ---- |
| GitHub Actions 云编译 | 首次约 2~3 小时（之后有缓存更快） | 只需一个 GitHub 账号 | **推荐**。免费、稳定、无需本地 Linux 环境，Mac/Windows 用户首选 |
| 本地完整编译 | 视机器性能 1~6 小时 | Ubuntu 22.04/24.04（物理机/虚拟机/云服务器），磁盘 ≥ 60G | 需要频繁调试 `menuconfig`、反复改源码 |
| 本地只打包（remake） | 几分钟~十几分钟 | 任意 Linux（含 Ubuntu/Debian） | rootfs 已编译好（自己或别人的 Releases 产物），只想换盒子型号/内核重新打包 |

> **macOS 用户注意**：OpenWrt 完整源码编译不支持在 macOS 上直接进行（本仓库 `.github/workflows` 中也是使用 Ubuntu 24.04 容器编译）。请使用 GitHub Actions，或在 Mac 上开一台 Linux 虚拟机（UTM/Parallels）/ Docker / 云服务器。

---

## 3. 方式一：GitHub Actions 云编译（推荐）

### 3.1 准备工作（一次性）

1. **Fork 本仓库**到自己的 GitHub 账号下。
2. **设置工作流读写权限**（否则编译好的固件无法上传到 Releases）：
   进入你 Fork 仓库的 `Settings` → `Actions` → `General` → `Workflow permissions`，选择 **`Read and write permissions`**，保存。

### 3.2 修改配置

按下文[第 4 节](#4-核心修改配置添加第三方软件)修改 `config/` 目录下的文件（例如启用 OpenClash），修改后提交到仓库。

### 3.3 触发编译

1. 进入你 Fork 仓库的 **Actions** 页面（首次需点击 `I understand my workflows, go ahead and enable them`）。
2. 左侧选择 **`Build OpenWrt system image`** → 点击 **`Run workflow`**，按需选择参数：

| 参数 | 默认值 | 说明 |
| ---- | ---- | ---- |
| source_branch | all | 选哪套源码：`lede_master`（插件最全，新手推荐）/ `immortalwrt_master` / `openwrt_main`（官方纯净源）。选 `all` = 三套全部编译（耗时长） |
| openwrt_board | all | 打包哪些盒子，如 `s905x3`、`s912`、`rockchip`（整个平台）；多个用 `_` 连接；全部型号见 [model_database.conf](../make-openwrt/openwrt-files/common-files/etc/model_database.conf) |
| openwrt_kernel | 6.12.y_6.18.y | 内核版本，多个用 `_` 连接；`6.1.y` 表示 6.1 系列最新版 |
| auto_kernel | true | 自动使用同系列最新内核 |
| kernel_repo / kernel_usage | ophub/kernel / stable | 内核仓库与标签，一般不改 |
| openwrt_ip | 192.168.1.1 | 固件默认 IP（想改管理地址时用） |
| use_ccache | true | 使用编译缓存，二次编译提速明显 |
| openwrt_storage | save | `save` 上传到 Releases（长期保存）；`temp` 只存 Actions 产物（90 天过期） |
| builder_name | ophub | 固件签名，随意 |

3. 点击绿色 `Run workflow` 按钮，等待约 2~3 小时。

### 3.4 下载固件

- `openwrt_storage=save` 时：到仓库 **Releases** 页面下载 `openwrt_xxx_盒子型号_内核_日期.img.gz`。
- `openwrt_storage=temp` 时：到 **Actions** 本次运行页面底部 `Artifacts` 下载。

同时 Releases 里还会有 `openwrt-armsr-armv8-generic-rootfs.tar.gz`（通用 rootfs）和 `config`（本次实际使用的完整配置，可回填用于精确定制）。

> **技巧**：rootfs 编译一次后，如果只想给不同盒子/内核打包（软件不变），无需重新编译 3 小时——使用 Actions 里的 **`Build OpenWrt using releases files`** 工作流，直接读取 Releases 中已有的 rootfs，几分钟即可打包出新固件。

---

## 4. 核心：修改配置添加第三方软件

### 4.1 三个定制文件（重点）

每套源码的定制文件都在 `config/<源码名>/` 目录下，共 3 个：

```text
config/lede_master/
├── config          # 种子配置：选择哪些软件包/功能编进固件（对应 make menuconfig 的 .config）
├── diy-part1.sh    # DIY 脚本一：在 feeds 更新【前】执行（用于添加第三方 feed 源）
└── diy-part2.sh    # DIY 脚本二：在 feeds 安装【后】执行（用于 clone 第三方包、改默认 IP/主题等）
```

| 源码目录 | 对应源码仓库 | 特点 | OpenClash 现状 |
| ---- | ---- | ---- | ---- |
| `config/lede_master` | [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) | 插件最多（ssr-plus、passwall 组件等），中文社区主流 | 种子 config 中已有该选项，**改一行即可启用** |
| `config/immortalwrt_master` | [immortalwrt/immortalwrt](https://github.com/immortalwrt/immortalwrt) | 更新规范，自带 luci-app-openclash | 同上，**改一行即可启用** |
| `config/openwrt_main` | [openwrt/openwrt](https://github.com/openwrt/openwrt) 官方 | 纯净，几乎无第三方插件 | **需要手动添加**（见 4.3 情况 B） |

GitHub Actions 编译时这 3 个文件的执行顺序（见 `build-openwrt-system-image.yml`）：

```text
克隆源码 → 【执行 diy-part1.sh】→ feeds update -a → feeds install -a
→ 【复制 config 为 .config】→ 【执行 diy-part2.sh】→ make defconfig → make download → make
```

本地编译时照同样顺序手工执行即可（见第 5 节）。

### 4.2 添加 OpenClash

#### 情况 A：使用 lede_master 或 immortalwrt_master（最简单，推荐）

源码自带 OpenClash 插件，只需在种子 config 中启用。编辑 `config/lede_master/config`，找到这一行（约第 5472 行；immortalwrt 在 `config/immortalwrt_master/config` 约第 5903 行）：

```shell
# CONFIG_PACKAGE_luci-app-openclash is not set
```

修改为：

```shell
CONFIG_PACKAGE_luci-app-openclash=y
```

也可以在仓库根目录用一条 sed 命令完成：

```shell
# lede 源
sed -i 's/# CONFIG_PACKAGE_luci-app-openclash is not set/CONFIG_PACKAGE_luci-app-openclash=y/' config/lede_master/config
# immortalwrt 源
sed -i 's/# CONFIG_PACKAGE_luci-app-openclash is not set/CONFIG_PACKAGE_luci-app-openclash=y/' config/immortalwrt_master/config
```

完成。提交后即可触发 Actions 编译。

> OpenClash 的运行依赖（`dnsmasq-full`、`kmod-tun`、`iptables-mod-tproxy`、`ip-full`、`curl`、`ca-bundle`、`bash`、`coreutils-nohup` 等）**当前种子 config 已全部选好**，无需额外处理。

#### 情况 B：使用 openwrt_main，或想用 vernesong 最新版 OpenClash

第一步，编辑 `config/<源码名>/diy-part2.sh`，在“Other”区域添加克隆命令（在源码根目录下执行，先删旧再克隆可覆盖源码自带版本）：

```shell
# 添加/更新 OpenClash（master 分支即 LuCI 插件包）
rm -rf package/luci-app-openclash
git clone --depth 1 -b master https://github.com/vernesong/OpenClash.git package/luci-app-openclash
```

第二步，在 `config/<源码名>/config` 文件末尾追加（若该行已存在则按情况 A 修改）：

```shell
CONFIG_PACKAGE_luci-app-openclash=y
```

> 说明：`make defconfig` 会自动补全依赖并展开为完整配置，所以直接在文件末尾追加 `CONFIG_PACKAGE_xxx=y` 也是合法做法。

#### 关于 OpenClash 内核（重要）

编译进固件的只是 **LuCI 管理界面（插件本体）**，Clash 内核（Meta / TUN 版）体积大，默认不打包：

- 常规做法：刷机后在 LuCI → `服务` → `OpenClash` → `版本更新` 中**在线下载内核**（需要路由器能访问 GitHub，可先配置好代理节点）。
- 离线做法：在 `版本更新` 页面手动上传内核文件，或把内核放到固件的 `/etc/openclash/core/` 目录（可通过第 4.6 节自定义文件方式集成）。

### 4.3 .config 文件语法速成

种子 config 每行一个配置项，规律只有三条：

```shell
CONFIG_PACKAGE_luci-app-xxx=y        # 编译并安装到固件（内置）
CONFIG_PACKAGE_luci-app-xxx=m        # 编译成 ipk 安装包（不内置，可后期 opkg 安装）
# CONFIG_PACKAGE_luci-app-xxx is not set   # 不编译（禁用）
```

- **启用**：去掉行首 `#` 和行尾 `is not set`，改为 `=y`。
- **禁用**：行首加 `#`，行尾改为 `is not set`。

常用可选项示例（均在 `config/lede_master/config` 中）：

| 需求 | 配置项 |
| ---- | ---- |
| 中文界面 | `CONFIG_PACKAGE_luci-i18n-base-zh-cn=y`（已默认启用） |
| argon 主题 | `CONFIG_PACKAGE_luci-theme-argon=y`（lede 自带；想用 jerrykuku 增强版见 4.4） |
| OpenClash 中文语言包 | `CONFIG_PACKAGE_luci-i18n-openclash-zh-cn=y`（若源码提供该包） |
| Docker（跑容器） | `CONFIG_PACKAGE_luci-app-dockerman=y` + `CONFIG_PACKAGE_dockerd=y` |
| 去掉不用的代理插件减小体积 | 将 `CONFIG_PACKAGE_luci-app-ssr-plus=y` 等改为 `is not set`（注意同时可关掉其 `INCLUDE_*` 子选项） |

> 体积提示：默认 rootfs 分区约 1GB，装 OpenClash 足够。若加了大量软件（Docker、NAS 套件等），打包时用 `-s 256/2048` 扩大分区，或刷机后在 `Amlogic 服务` 里在线扩容。

### 4.4 diy-part2.sh 常用写法（添加/替换/删除包）

`diy-part2.sh` 在源码根目录下执行，可写任何 shell 命令。常用模板：

```shell
# 1. 添加第三方软件包（源码里没有的）
git clone --depth 1 https://github.com/jerrykuku/luci-app-ttnode.git package/luci-app-ttnode

# 2. 用第三方同名包替换源码自带版本（例：jerrykuku 增强版 argon 主题）
rm -rf package/luci-theme-argon
git clone --depth 1 -b master https://github.com/jerrykuku/luci-theme-argon.git package/luci-theme-argon

# 3. 删除源码自带、自己用不到的包（减小体积、加快编译）
rm -rf package/lean/{luci-app-samba4,luci-app-ttyd}

# 4. 修改默认主题为 argon
sed -i 's/luci-theme-bootstrap/luci-theme-argon/g' feeds/luci/collections/luci/Makefile
```

> 每加一个包，记得回到 config 里加对应的 `CONFIG_PACKAGE_xxx=y`（除非它是别的包的依赖会被自动选中）。

### 4.5 diy-part1.sh：添加 feed 源

`diy-part1.sh` 在 `feeds update` 之前执行，适合引入整个第三方软件仓库（feed 方式会参与依赖管理）：

```shell
# 在源码的 feeds.conf.default 末尾追加 feed 源（取消仓库自带的注释示例即可）
sed -i '$a src-git lienol https://github.com/Lienol/openwrt-package' feeds.conf.default
```

feed 与直接 git clone 的区别：feed 源里的包在 `feeds install -a` 后同样出现在 menuconfig 里，效果一致；小型需求用 4.4 的 clone 更直观。

### 4.6 进阶：本地 menuconfig 生成精确配置回填

种子 config 有 8000+ 行，是维护者生成的“全量快照”。想精确自定义时：

```shell
# 在本地编译环境（见第 5 节）的 openwrt 源码根目录执行
make menuconfig        # 图形界面里按 / 可搜索包名，空格选择，方向键导航
# 保存退出后，导出"差异精简配置"（只含你改过的项，更好维护）
./scripts/diffconfig.sh > myconfig.diff
```

把 `myconfig.diff` 的内容作为种子 `config/lede_master/config` 上传即可（defconfig 会自动展开补全）。这个方法也是**更换源码分支时保留自定义配置**的标准做法。

---

## 5. 方式二：本地完整编译

适合需要反复调试配置的用户。以 Ubuntu 24.04 为例：

### 5.1 环境准备

```shell
# 安装编译依赖（本仓库自带依赖清单）
sudo apt-get update -y
sudo apt-get install -y $(cat make-openwrt/scripts/ubuntu2404-make-openwrt-depends)
# Ubuntu 22.04 用 ubuntu2204-make-openwrt-depends

# 建议关闭 swap 对编译无益不强制；磁盘空间确保 ≥ 60G
```

### 5.2 编译 rootfs（阶段一）

```shell
# 1. 克隆源码（三选一；lede 插件最全）
git clone --depth 1 https://github.com/coolsnowwolf/lede.git openwrt
cd openwrt

# 2. 应用本仓库的定制三件套（以 lede_master 为例，路径按实际调整）
cp ../config/lede_master/diy-part1.sh /tmp/diy-part1.sh
chmod +x /tmp/diy-part1.sh && /tmp/diy-part1.sh

# 3. 更新并安装 feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 4. 导入种子配置并执行 diy 脚本二（参数：默认IP 是否用ccache）
cp ../config/lede_master/config .config
chmod +x ../config/lede_master/diy-part2.sh
../config/lede_master/diy-part2.sh 192.168.1.1 true

# 5.（可选）图形界面微调，比如确认 luci-app-openclash 已选中
make defconfig
make menuconfig

# 6. 下载并编译（-j 后面是线程数；首次编译约 1~6 小时）
make download -j8
make -j$(nproc) || make -j1 V=s    # 失败时单线程重试可看到详细报错

# 7. 产物位置
ls bin/targets/armsr/armv8/*rootfs.tar.gz
```

### 5.3 打包成盒子固件（阶段二）

```shell
# 回到 amlogic-s9xxx-openwrt 仓库根目录，把 rootfs 放入 openwrt-armsr 目录
mkdir -p openwrt-armsr
cp <编译产物路径>/openwrt-armsr-armv8-generic-rootfs.tar.gz openwrt-armsr/

# 打包（见第 6 节参数说明）
sudo ./remake -b s905x3 -k 6.1.y
# 产物在 openwrt/out/ 目录
```

---

## 6. 方式三：本地只打包（remake）

前提：手上已有 `openwrt-armsr-armv8-generic-rootfs.tar.gz`（自己编译的，或从 [Releases](https://github.com/ophub/amlogic-s9xxx-openwrt/releases) 下载的——注意 Releases 的 rootfs 是官方默认配置，**不含你的自定义软件**，自定义必须走阶段一）。

```shell
git clone --depth 1 https://github.com/ophub/amlogic-s9xxx-openwrt.git
cd amlogic-s9xxx-openwrt

# 安装打包依赖
sudo apt-get update -y
sudo apt-get install -y $(cat make-openwrt/scripts/ubuntu2404-make-openwrt-depends)

# 放入 rootfs
mkdir -p openwrt-armsr
cp /path/to/openwrt-armsr-armv8-generic-rootfs.tar.gz openwrt-armsr/

# 打包
sudo ./remake -b s905x3 -k 6.1.y
```

常用参数：

| 参数 | 默认 | 说明 |
| ---- | ---- | ---- |
| `-b` | all | 盒子型号，多个用 `_` 连接；也可按平台批量：`amlogic` / `rockchip` / `allwinner` |
| `-k` | 最新 | 内核版本，如 `6.1.10`、`6.1.y`（系列最新），多个用 `_` 连接 |
| `-a` | true | 自动升级到同系列最新内核 |
| `-p` | 192.168.1.1 | 修改固件默认 IP |
| `-s` | 256/1024 | BOOTFS/ROOTFS 分区大小（MB），如 `-s 256/2048` |
| `-r` / `-u` | ophub/kernel / stable | 内核仓库 / 标签 |
| `-n` | 无 | 构建者签名 |

> 速度对比：remake 只做打包，几分钟即可出全部型号固件；改软件包则必须重走阶段一。

---

## 7. 刷机与安装

1. **写入 U 盘 / TF 卡**：用 balenaEtcher / Rufus 把 `openwrt_xxx.img.gz`（解压出 .img）写入。
2. **盒子从 U 盘启动**：晶晨盒子一般需先线刷/面具工具短接进入 U 盘启动；瑞芯微盒子用瑞芯微开发工具。各型号详细方法见 [README.cn.md 安装说明](../README.cn.md)。
3. **写入 eMMC（可选，性能更好）**：浏览器访问 `192.168.1.1` → `系统` → `Amlogic 服务`（luci-app-amlogic 已由 diy-part2.sh 自动集成进固件）→ `安装 OpenWrt`。

固件默认信息：

| 项目 | 值 |
| ---- | ---- |
| 管理地址 | `192.168.1.1`（可在编译参数中修改） |
| 用户名 / 密码 | `root` / `password` |

---

## 8. 常见问题 FAQ

**Q1：OpenClash 装上后启动失败 / 没有内核？**
插件本体不含 Clash 内核。进入 LuCI → OpenClash → `版本更新`，在线下载 Meta/TUN 内核；无法访问 GitHub 时先手动配置代理或在电脑上下载后上传。

**Q2：提示 dnsmasq 与 dnsmasq-full 冲突？**
OpenClash 依赖 `dnsmasq-full`，两者不能共存。本仓库种子配置已选 `dnsmasq-full`；若你自行改配置，确保 `CONFIG_PACKAGE_dnsmasq-full=y` 且 `dnsmasq` 为 not set。

**Q3：GitHub Actions 编译失败怎么办？**
点进失败的运行看日志定位：
- 下载超时（国内网络）：重跑一次通常即可（`Re-run failed jobs`）。
- 某个软件包编译错误：多为第三方包与最新源码不兼容，在 diy-part2.sh 里锁定该包的发布分支/tag 再克隆。
- 磁盘空间不足：官方工作流已自动扩容，一般是你加了太多软件，精简配置。

**Q4：二次编译如何提速？**
Actions 勾选 `use_ccache=true`（默认已开）；rootfs 不变只换盒子/内核时，用 `Build OpenWrt using releases files` 工作流跳过编译。

**Q5：想换源码分支（如 lede → immortalwrt），配置要重做吗？**
用 `./scripts/diffconfig.sh` 导出差异配置，再到新分支 `make defconfig` 展开（详见 4.6 节与 [documents/README.cn.md 4.4 节](README.cn.md)）。直接复制全量 config 在分支间不通用。

**Q6：内核版本怎么选？**
打包阶段可随时换，不影响软件。一般 5.15/6.1 系列最稳妥；新盒子（rk3588 等）需要较新内核。同一 rootfs 可打包多个内核版本分别测试。

**Q7：为什么我下载的 Releases rootfs 打包出的固件没有 OpenClash？**
Releases 里的 rootfs 是默认配置产物。含自定义软件的固件必须自己完成阶段一编译（Actions 或本地），再打包。

**Q8：想在固件里内置默认配置（如预配置 OpenClash）？**
把配置文件按 OpenWrt 根目录结构放好（如 `etc/config/network`），本地打包时用 remake 支持的自定义文件目录覆盖，或在源码根目录建 `files/` 目录（Actions 工作流会自动 `mv files openwrt/files`）。

---

## 参考链接

- 上游使用文档：[documents/README.cn.md](README.cn.md) / [README.cn.md](../README.cn.md)
- OpenClash 官方仓库：<https://github.com/vernesong/OpenClash>
- 支持的盒子型号清单：[model_database.conf](../make-openwrt/openwrt-files/common-files/etc/model_database.conf)
- 内核仓库：<https://github.com/ophub/kernel/releases>
