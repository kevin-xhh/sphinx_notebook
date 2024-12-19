## 2. u-boot
### 2.1 u-boot编译流程分析
[u-boot官方在线文档](https://u-boot.readthedocs.io/en/latest/index.html)

在分析uboot编译过程之前，必须了解Makefile语法。由于u-boot的Makefile中存在相互调用，这里介绍以下make -f和make -c的区别

-c选项：Makefile中的-c是递归调用子目录的Makefile，-c选项后跟目录，表示到子目录下执行子目录的makefile，顶层Makefile的export的变量还有make默认的变量是可以传递给子目录中的Makefile的

-f选项：顶层Makefile使用make -f调用子目录中的文件（文件名可以随便，不必一定是Makefile）作为Makefile，顶层export的变量可以传递给底层

**注解：在顶层Makefile中使用make -f调用子目录中的文件工作目录仍是顶层目录，即CURDIR变量仍是顶层目录**

在顶层Makefile中使用make -f调用子目录中的文件工作目录仍是顶层目录，即CURDIR变量仍是顶层目录


- 2.1.1 make xxx_defconfig配置分析
  - 2.1.1.1 xxx_defconfig依赖检查更新
  - 2.1.1.2 xxx_defconfig目标命令执行
- 2.1.2 make执行流程分析
  - 2.1.2.1 目标_all和all对$(ALL-y)的依赖
  - 2.1.2.2 u-boot目标编译
    - 2.1.2.2.1 prepare编译
### 2.2 源码分析
- 2.2.1 目录结构
- 2.2.2 重要文件
- 2.2.3 生成文件
- 2.2.4 u-boot启动流程
  - 2.2.4.1 SPL启动流程分析
  - 2.2.4.2 u-boot启动流程分析
  - 2.2.4.3 移植u-boot
  - 2.2.4.4 u-boot命令总结
### 2.3 Legacy-uImage & FIT-uImage
- 2.3.1 Legacy-uImage
- 2.3.2 FIT-uImage
- 2.3.3 image sorce file语法

### 2.4 uboot解析uImage的kernel信息
- 2.4.1 kernel信息的存放位置
- 2.4.2 legacy-uImage中kernel信息的解析代码流程
- 2.4.3 FIT-uImage中kernel信息的解析

### 2.5 bootm原理
- 2.5.1 基础知识
- 2.5.2 bootm实现

### 2.6 uboot向kernel的传参机制-bootm与tags
- 2.6.1 struct tag传递参数
  - 2.6.1.1 主要数据结构介绍
  - 2.6.1.2 global data介绍及背后的思考
- 2.6.2 设备树传参

### 2.7 uboot-kernel地址空间
- 2.7.1 uboot地址空间
  - 2.7.1.1 uboot在DDR中的地址空间
  - 2.7.1.2 system boot sequence
  - 2.7.1.3 emmc flash env
- 2.7.2 kernel地址空间
  - 2.7.2.1 arm64 物理地址与虚拟地址空间

### 2.8 多核异构平台系统启动分析
- 2.8.1 IPL启动系统流程
  - 2.8.1.1 load image
  - 2.8.1.2 完整性检查
  - 2.8.1.3 IPL flow
- 2.8.2 memory layout