# GitHub Actions 对比说明 / GitHub Actions Comparison

本文档详细说明了此仓库中各个 GitHub Actions 工作流的用途和区别。

This document explains the purpose and differences between the various GitHub Actions workflows in this repository.

---

## 📋 工作流概览 / Workflow Overview

本仓库包含 7 个 GitHub Actions 工作流：

This repository contains 7 GitHub Actions workflows:

### 1. **compile-kernel.yml** - 编译内核 / Compile the Kernel

**主要用途 / Primary Purpose:**
- 编译 Linux 内核
- Compiles Linux kernels

**特点 / Features:**
- 运行环境：`ubuntu-24.04-arm`
- 支持多种内核源：`unifreq`、`ophub`
- 支持多个内核版本：5.10.y、5.15.y、6.1.y、6.6.y、6.12.y、6.18.y
- 支持多种编译工具链：gcc、clang、gcc-15.2、gcc-14.3、gcc-14.2
- 支持 ccache 缓存加速编译
- 支持选择编译包类型：all、dtbs
- 可自定义内核签名

**输出 / Output:**
- 编译好的内核包上传到 Release (`kernel_stable` tag)
- Compiled kernel packages uploaded to Release

**适用场景 / Use Cases:**
- 需要自定义编译内核
- 需要最新的内核版本
- 需要使用特定的编译工具链

---

### 2. **build-openwrt-system-image.yml** - 构建 OpenWrt 系统镜像

**主要用途 / Primary Purpose:**
- 从源代码完整编译 OpenWrt 系统
- Compiles OpenWrt system from source code

**特点 / Features:**
- 运行环境：`ubuntu-24.04`
- 支持三个源代码分支：
  - `openwrt_main` - 官方 OpenWrt 主分支
  - `immortalwrt_master` - ImmortalWrt 主分支
  - `lede_master` - Lean's LEDE 源码
- 完整的编译流程：
  - 克隆源代码
  - 加载自定义 feeds
  - 下载和安装软件包
  - 完整编译 OpenWrt
  - 打包为设备镜像
- 支持 ccache 缓存加速后续编译
- 支持超过 200 种设备板型

**编译流程 / Build Process:**
1. 初始化环境
2. 克隆 OpenWrt 源代码
3. 加载自定义 feeds 和配置
4. 下载依赖包
5. 完整编译系统（最耗时）
6. 打包为 rootfs.tar.gz
7. 使用 `ophub/amlogic-s9xxx-openwrt` 打包为设备镜像

**输出 / Output:**
- OpenWrt rootfs 和设备镜像
- Tag: `OpenWrt_{source_branch}_{storage}_YYYY.MM`

**适用场景 / Use Cases:**
- 需要自定义 OpenWrt 配置
- 需要添加自定义软件包
- 需要最新的源代码编译
- 第一次构建或需要完整控制编译过程

**编译时间 / Build Time:**
- 最长（2-4 小时或更久）

---

### 3. **build-openwrt-using-imagebuilder.yml** - 使用 ImageBuilder 构建

**主要用途 / Primary Purpose:**
- 使用 OpenWrt 官方 ImageBuilder 快速构建镜像
- Uses OpenWrt's official ImageBuilder to quickly build images

**特点 / Features:**
- 运行环境：`ubuntu-24.04`
- 基于 OpenWrt/ImmortalWrt 官方发布的预编译版本
- 支持版本：
  - openwrt:24.10.5
  - openwrt:24.10.4
  - immortalwrt:24.10.4
  - immortalwrt:24.10.3
- 无需完整编译，只是重新打包
- 构建速度快（相比完整编译）
- 运行脚本 `config/imagebuilder/imagebuilder.sh`

**工作流程 / Workflow:**
1. 下载官方 ImageBuilder
2. 配置软件包和设置
3. 生成 rootfs.tar.gz
4. 打包为设备镜像

**输出 / Output:**
- OpenWrt 镜像基于官方发布版本
- Tag: `OpenWrt_imagebuilder_{releases_branch}_YYYY.MM`

**适用场景 / Use Cases:**
- 基于官方稳定版本
- 不需要自定义编译
- 需要快速构建
- 只需要添加/删除软件包

**编译时间 / Build Time:**
- 中等（30分钟-1小时）

---

### 4. **build-openwrt-using-unifreq-scripts.yml** - 使用 Unifreq 脚本构建

**主要用途 / Primary Purpose:**
- 使用 Unifreq 的打包脚本从已有的 rootfs 制作设备镜像
- Uses Unifreq's packaging scripts to create device images from existing rootfs

**特点 / Features:**
- 运行环境：`ubuntu-24.04`
- 从 Release 下载已有的 OpenWrt rootfs.tar.gz
- 使用 `ophub/flippy-openwrt-actions` 打包
- 支持自定义设备选择（包括 Amlogic、Rockchip、Allwinner 设备）
- 可自定义 IP 地址、内核版本
- 支持自定义脚本（`script_diy_path`）
- 支持自定义 rk3399 设备

**工作流程 / Workflow:**
1. 从 Release 下载最新的 rootfs.tar.gz
2. 使用打包脚本为指定设备制作镜像
3. 上传到 Release

**输出 / Output:**
- 各种设备的 OpenWrt 镜像
- Tag: `OpenWrt_armv8_{storage}_YYYY.MM`

**适用场景 / Use Cases:**
- 已有 rootfs，需要为不同设备打包
- 需要快速生成多设备镜像
- 测试不同内核版本
- 基于已编译的 rootfs 制作镜像

**编译时间 / Build Time:**
- 最短（10-30分钟）

---

### 5. **build-openwrt-using-releases-files.yml** - 使用 Release 文件构建

**主要用途 / Primary Purpose:**
- 从 Release 下载已构建的 rootfs 并重新打包
- Downloads pre-built rootfs from Release and repackages

**特点 / Features:**
- 运行环境：`ubuntu-24.04`
- 与 `build-openwrt-using-unifreq-scripts.yml` 类似
- 从仓库的 Release 下载指定分支的 rootfs
- 支持三个源代码分支的 rootfs：
  - openwrt_main
  - immortalwrt_master
  - lede_master
- 使用 `ophub/amlogic-s9xxx-openwrt` 打包

**工作流程 / Workflow:**
1. 从 Release 搜索并下载指定分支的 rootfs
2. 打包为设备镜像
3. 上传到 Release

**输出 / Output:**
- 设备镜像
- Tag: `OpenWrt_{source_branch}_{storage}_YYYY.MM`

**适用场景 / Use Cases:**
- 复用之前编译好的系统
- 为不同设备快速打包
- 测试不同内核搭配

**编译时间 / Build Time:**
- 很短（10-30分钟）

---

### 6. **build-openwer-docker-image.yml** - 构建 Docker 镜像

**主要用途 / Primary Purpose:**
- 构建 OpenWrt 的 Docker 镜像并推送到 Docker Hub
- Builds OpenWrt Docker image and pushes to Docker Hub

**特点 / Features:**
- 运行环境：`ubuntu-24.04-arm`
- 从 Release 下载 rootfs
- 运行 `config/docker/make_docker_image.sh` 制作 Docker 镜像
- 推送到 Docker Hub
- 支持的镜像：
  - ophub/openwrt-armv8:latest
  - ophub/openwrt-aarch64:latest

**工作流程 / Workflow:**
1. 下载指定分支的 rootfs.tar.gz
2. 制作 Docker 镜像
3. 使用 Docker Buildx 构建
4. 推送到 Docker Hub

**输出 / Output:**
- Docker 镜像推送到 Docker Hub
- 平台：linux/arm64

**适用场景 / Use Cases:**
- 需要在容器中运行 OpenWrt
- 需要部署到 Docker 环境
- 测试和开发用途

**前置要求 / Requirements:**
- 需要配置 Docker Hub 凭证：
  - `DOCKERHUB_USERNAME`
  - `DOCKERHUB_PASSWORD`

---

### 7. **delete-older-releases-workflows.yml** - 清理旧版本和工作流

**主要用途 / Primary Purpose:**
- 清理旧的 Release 文件和工作流运行记录
- Cleans up old releases and workflow run records

**特点 / Features:**
- 运行环境：`ubuntu-24.04-arm`
- 使用 `ophub/delete-releases-workflows` action
- 可配置：
  - 是否删除 Releases
  - 是否删除相关 Tags
  - 保留最新的 N 个 Release
  - 保留包含特定关键词的 Release
  - 是否删除工作流记录
  - 保留最近 N 天的工作流

**默认配置 / Default Settings:**
- 保留最新 2 个 Release
- 保留关键词：`v0/_save_/kernel_`
- 保留最近 1 天的工作流记录

**适用场景 / Use Cases:**
- 清理仓库空间
- 删除过时的 Release
- 管理工作流运行历史

---

## 🔄 工作流关系图 / Workflow Relationships

```
┌─────────────────────────────────────────────────────────────┐
│                    compile-kernel.yml                        │
│                     编译 Linux 内核                           │
│                    Compile Kernel                            │
└──────────────────────┬──────────────────────────────────────┘
                       │ outputs: kernel packages
                       ↓
┌─────────────────────────────────────────────────────────────┐
│              build-openwrt-system-image.yml                  │
│           从源代码完整编译 OpenWrt                             │
│           Full compilation from source                       │
└──────────────────────┬──────────────────────────────────────┘
                       │ outputs: rootfs.tar.gz
                       ↓
┌─────────────────────────────────────────────────────────────┐
│         build-openwrt-using-imagebuilder.yml                 │
│          基于官方 ImageBuilder 构建                           │
│          Build using official ImageBuilder                   │
└──────────────────────┬──────────────────────────────────────┘
                       │ outputs: rootfs.tar.gz
                       ↓
        ┌──────────────┴──────────────┐
        ↓                              ↓
┌───────────────────────┐   ┌──────────────────────────┐
│  build-openwrt-using- │   │  build-openwrt-using-    │
│  unifreq-scripts.yml  │   │  releases-files.yml      │
│  使用已有 rootfs 打包  │   │  从 Release 下载打包      │
│  Package from rootfs  │   │  Download and package    │
└───────────┬───────────┘   └────────────┬─────────────┘
            │ outputs: device images      │
            └──────────────┬──────────────┘
                           ↓
              ┌─────────────────────────┐
              │  build-openwer-docker-  │
              │  image.yml              │
              │  制作 Docker 镜像         │
              │  Build Docker image     │
              └─────────────────────────┘

                           ↓
        ┌──────────────────────────────────────┐
        │  delete-older-releases-workflows.yml  │
        │       清理旧版本和工作流记录              │
        │       Cleanup old releases            │
        └──────────────────────────────────────┘
```

---

## 📊 对比表格 / Comparison Table

| 工作流 / Workflow | 构建时间 / Time | 灵活性 / Flexibility | 适用场景 / Use Case |
|------------------|----------------|---------------------|-------------------|
| **compile-kernel** | 长 / Long | 高 / High | 自定义内核编译 / Custom kernel |
| **system-image** | 最长 / Longest | 最高 / Highest | 完整自定义编译 / Full custom build |
| **imagebuilder** | 中等 / Medium | 中等 / Medium | 基于官方稳定版 / Official stable base |
| **unifreq-scripts** | 短 / Short | 中等 / Medium | 多设备快速打包 / Quick multi-device |
| **releases-files** | 短 / Short | 低 / Low | 复用已编译系统 / Reuse built system |
| **docker-image** | 短 / Short | 低 / Low | Docker 容器部署 / Docker deployment |
| **delete-older** | 很短 / Very Short | - | 清理维护 / Cleanup maintenance |

---

## 🎯 推荐使用流程 / Recommended Workflow

### 场景 1：首次构建 / First Build
```
1. build-openwrt-system-image.yml (编译系统 / Build system)
   ↓
2. build-openwrt-using-unifreq-scripts.yml (打包设备镜像 / Package for devices)
```

### 场景 2：更新内核 / Update Kernel
```
1. compile-kernel.yml (编译新内核 / Compile new kernel)
   ↓
2. build-openwrt-using-releases-files.yml (使用新内核打包 / Package with new kernel)
```

### 场景 3：快速构建稳定版 / Quick Stable Build
```
1. build-openwrt-using-imagebuilder.yml (基于官方版本 / Based on official release)
   ↓
2. build-openwrt-using-unifreq-scripts.yml (可选，为更多设备打包 / Optional, for more devices)
```

### 场景 4：添加 Docker 支持 / Add Docker Support
```
使用已有的 rootfs / Use existing rootfs
   ↓
build-openwer-docker-image.yml (制作 Docker 镜像 / Build Docker image)
```

### 场景 5：定期维护 / Regular Maintenance
```
delete-older-releases-workflows.yml (清理旧文件 / Clean up old files)
```

---

## 💡 选择建议 / Selection Guide

### 如果你需要... / If you need...

**最快速度 / Fastest Build:**
→ `build-openwrt-using-releases-files.yml` 或 `build-openwrt-using-unifreq-scripts.yml`

**最大自定义 / Maximum Customization:**
→ `build-openwrt-system-image.yml`

**官方稳定版 / Official Stable:**
→ `build-openwrt-using-imagebuilder.yml`

**自定义内核 / Custom Kernel:**
→ `compile-kernel.yml`

**Docker 部署 / Docker Deployment:**
→ `build-openwer-docker-image.yml`

**清理空间 / Cleanup Space:**
→ `delete-older-releases-workflows.yml`

---

## 📝 注意事项 / Notes

1. **存储空间 / Storage:**
   - `system-image` 需要大量磁盘空间（创建模拟物理磁盘）
   - Other workflows require less storage

2. **构建时间 / Build Time:**
   - 完整编译可能需要 2-4 小时或更久
   - 使用 ccache 可以加速后续编译

3. **运行环境 / Runner:**
   - 部分使用 ARM runner (`ubuntu-24.04-arm`)
   - 部分使用标准 runner (`ubuntu-24.04`)

4. **依赖关系 / Dependencies:**
   - 某些工作流依赖于之前构建的产物（Release 中的文件）
   - 确保按正确顺序运行

5. **内核版本 / Kernel Versions:**
   - 所有工作流支持多个内核版本
   - 可以自动使用最新内核（`auto_kernel: true`）

---

## 🔗 相关链接 / Related Links

- OpenWrt 官方: https://openwrt.org/
- ImmortalWrt: https://github.com/immortalwrt/immortalwrt
- Lean's LEDE: https://github.com/coolsnowwolf/lede
- Ophub's Actions: https://github.com/ophub/amlogic-s9xxx-openwrt
- Unifreq's Scripts: https://github.com/unifreq/openwrt_packit

---

**最后更新 / Last Updated:** 2026-02-19
