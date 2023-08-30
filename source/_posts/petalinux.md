---
title: petalinux编译环境替换设计
date: 2020-05-20
tags:
- Linux
---
<div align="center">
<font size=7 > petalinux编译环境替换设计

</font>
</div>
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

<div align="center">

|版本|简介|编辑日期|作者|
|---|----|----|----|
|V1.0|初稿|2020/5/20|January|


</div>


#  1. 概述 

## 设计背景

1. petalinux环境占用内存过大，每个项目产品，都需要针对其创建一个petalinux的平台环境实测20G，每个项目实测26G，服务器上2022平台就占用了168G。
2. petalinux编译过慢，快则40分钟，慢则两个小时，甚至更久。
3. petalinux黑盒子较多，许多框架较为神秘，不知怎么编译，怎么实现，怎么链接的，全包含在他的编译工具里面。
4. 编译过程中，出现各种错误，很难进行排查，常见错误python xxx错误，验证影响底层开发。
5. 调试内核时，往往编个版本进行验证，需要花费许久时间，一点小改动，就得重新进行以小时为单位的编译时间花费。极其不人性化。
6. busybox中的命令被xilinx内嵌进环境中，无法获取环境之外的命令支持。
7. 要么得联网编译，网速感人，要么需要下载75G的环境安装包，占用内存太大。

## 设计目的

- 提高内核编译效率，有助于内核bug解决。
- 减少搭建环境造成的资源浪费。（内存资源和搭建成本）
- 底层开发项目复用。（petalinux环境有权限要求，需创建人权限才能编译）
- 开源项目。

# 2. petalinux环境介绍

1. petalinux环境搭建
    - xilinx官网下载petalinux-v2019.2-final-installer.run（以19版为例）
    - 服务器/虚拟机下载好一系列环境库（python zlib1g-dev gcc-multilib zlib1g:i386 screen pax等）
    - 运行run文件开始搭建petalinux环境（耗时很久）

2. 创建项目及编译前准备
    - petalinux-create --type project --template zynqMP --name test(创建项目)
    - 生成源码（每次创建项目都需要重新生成源码，2015以后均不可复用）
    - petalinux-config（配置各种环境，UBOOT、kernel、rootfs、config，uboot和kernel的config文件不知逻辑保存）

3. 各个模块进行编译
    - petalinux-build -c u-boot(编译UBOOT)
    - petalinux-build -c kernel(编译kernel)
    - petalinux-build -c rootfs(编译rootfs)
    - petalinux-build -c device-tree(编译device-tree)

4. 每次编译都需要40+min,第一次编译甚至更久。

# 3. 设计思路

## 底层内核image.ub组成

- Linux内核编译出来的Image.gz（选择内核压缩方式为gzip）
- rootfs.cpio.gz（包含busybox及一些底层命令和脚本文件）
- system.dtb（设备树）

## Linux内核编译

- 官网下载Linux内核源码，目前下载的是和xilinx同版本内核linux-5.15.19
- 下载arm官网编译器aarch64-none-linux-gnu-gcc，不采用xilinx的aarch64-linux-gnu-gcc
- 移植arm芯片的soc及platform信息，以及一些xilinx驱动（dma、clk等xilinx打过补丁）
- 配置linux内核config。（由于是标准内核，config文件都可找到）
- 直接make进行编译内核，生成目录就在arch/arm64/boot/

## rootfs编译

- rootfs只是命令集合体和脚本文件
- petalinux增减命令功能，可下载最新的busybox命令库，在busybox中选定需要的命令进行编译替换。
- 脚本启动文件等，我直接借用A3的rootfs文件，暂未深入研究，不影响编译环境。
- 用户名密码/shell命令行提示符等，均可通过修改passwd/host等实现（已完成测试）

## 设备树编译

- 目前我是将解析出来的FPGA的xsa文件移植到arch/arm64/boot/dts中
- 通过设备树中的include链接成一个system.dts
- 和linux内核一起编译成system.dtb，生成目录就在arch/arm64/boot/dts

## 生成image.ub文件

- 通过mkimage命令将Image.gz/rootfs.cpio.gz/system.dtb文件进行链接生成image.ub（xilinx同款链接方式）
- 该种编译内核方式是linux官方给的一种参考，故与xilinx无关，暂不修改

# 4. 个人测试结果

1. 内核可以稳定启动，内核相关启动日志及驱动打印和petalinux环境生成的一模一样
2. 网卡、串口、N850驱动、GPIO驱动、硬核驱动、UBI文件系统挂载正常、FPGA加载OK、WEB端显示正常（查询、操作等）、ARM运行未见异常。
3. 底层自测表格测试OK，未深入测试。
4. 内核编译花费3分钟，且使用几核编译均可参数控制。
5. 内核占用内存1.5G,压缩包800MB。进行备份，可做到如ARM代码压缩备份。不依赖权限问题,指定编译器即可编译。

# 5. 风险评估

1. 稳定性风险，设备没有经过长时间稳定性测试，无法保证稳定性。
2. 在线升级、网卡驱动等一些列，均只做过一次性测试，未做压力测试，故无法保证功能完整性。
3. 设备性能可能存在风险，未做任何性能测试，无法保证性能。

# 6. 工作量评估

## 初次编译

- 根据xilinx SDK源码移植xilinx相关驱动及控制器、平台信息等
- 配置内核的config文件
- 根据xsa，使用petalinux生成设备树（目前未研究设备树该如何生成）
- 移植设备树到内核，进行编译
- 按项目需求确定busybox的命令库及启动顺序

## 二次编译/其他平台

- 内核源码直接解压缩复用
- 若是其他平台，生成设备树后替换设备树即可
- 按实际项目需求进行处理

# 7. 设计总结

此次设计属于解决底层开发人员的去繁就简，提高开发效率，减少内存占用和提高可复用性，避免严重依赖xilinx而做的技术研究。单就设备底层而言，未有任何性能优化和提高的地方，只是给公司提供了另一种底层开发的思路。

