# ARM平台的编译器

ARM平台的编译器主要有以下几种：

- ARMCC
- IAR
- GCC  ARM
- LLVM(clang)

对比如下：

| 编译器         | 开发商                                      | 是否免费 | 支持的平台                  |
| ----------- | ---------------------------------------- | ---- | ---------------------- |
| ARMCC       | ARM                                      | 收费   | Windows、Linux          |
| IAR         | IAR                                      | 收费   | Windows                |
| GCC  ARM    | ARM、Linaro、Mentor(Codesourcery(Siemens)) | 免费   | Windows、Linux、Mac OS X |
| LLVM(clang) | LLVM                                     | 免费   | Windows、Linux、Mac OS X |

# GCC ARM

GCC ARM交叉编译器主要有三大工具商提供：

- ARM

- Mentor（Codesourcery(Siemens)）

- Linora

ARM交叉编译工具链的命名规则：`arch [-vendor] [-os] [-(gnu)eabi] [-gcc]`，各部分解释如下：

- arch：体系架构，比如ARM、MIPS等

- vendor：工具链提供商，如果没有vendor则使用none代替

- os：目标操作系统，没有os支持时则使用none代替

- eabi：Embedded Application Binary Interface嵌入式应用二进制接口

如果vendor和os都没有，则只使用一个none来代替。

## Arm GNU Toolchain

### 2022年之前的Arm GNU Toolchain

在2022年之前Arm GNU Toolchain被分为两类：

- GNU Toolchain for A-profile processors

- GNU Arm Embedded Toolchain

区别如下：

| 分类                                     | 适用处理器                                         | 支持的平台                  | 下载链接                                                                                                             |
| -------------------------------------- | --------------------------------------------- | ---------------------- | ---------------------------------------------------------------------------------------------------------------- |
| GNU Toolchain for A-profile processors | Cortex-A处理器家族                                 | Windows、Linux          | https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads |
| GNU Arm Embedded Toolchain             | 32位的Cortex-A处理器家族、Cortex-M处理器家族、Cortex-R处理器家族 | Windows、Linux、Mac OS X | https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-rm/downloads |

#### GNU Toolchain for A-profile processors

版本：`10.3-2021.07`

##### Windows平台

Windows (mingw-w64-i686) hosted cross compilers

| 编译器名称                    | 编译的目标平台                        |              |
| ------------------------ | ------------------------------ | ------------ |
| arm-none-eabi            | AArch32 bare-metal target      | 32位纯裸机平台     |
| arm-none-linux-gnueabihf | AArch32 target with hard float | 32位带硬件模式浮点运算 |
| aarch64-none-elf         | AArch64 bare-metal target      | 64位纯裸机平台     |
| aarch64-none-linux-gnu   | AArch64 GNU/Linux target       | 64位Linux平台   |

##### Linux平台

x86_64 Linux hosted cross compilers

| 编译器名称                     | 编译的目标平台                        |                  |
| ------------------------- | ------------------------------ | ---------------- |
| arm-none-eabi             | AArch32 bare-metal target      | 32位纯裸机平台         |
| arm-none-linux-gnueabihf  | AArch32 target with hard float | 32位带硬件模式浮点运算     |
| aarch64-none-elf          | AArch64 ELF bare-metal target  | 64位纯裸机平台         |
| aarch64-none-linux-gnu    | AArch64 GNU/Linux target       | 64位Linux平台       |
| aarch64_be-none-linux-gnu | AArch64 GNU/Linux target       | 64位Linux平台（大端模式） |

AArch64 Linux hosted cross compilers

| 编译器名称                    | 编译的目标平台                        |              |
| ------------------------ | ------------------------------ | ------------ |
| arm-none-eabi            | AArch32 bare-metal target      | 32位纯裸机平台     |
| arm-none-linux-gnueabihf | AArch32 target with hard float | 32位带硬件模式浮点运算 |
| aarch64-none-elf         | AArch64 ELF bare-metal target  | 64位纯裸机平台     |

#### GNU Arm Embedded Toolchain

此类编译工具链名字只有`arm-none-eabi`，只能编译裸机平台

版本：`10.3-2021.10`

##### Windows平台

| 编译器名称         | 编译的目标平台 |
| ------------- | ------- |
| arm-none-eabi |         |

##### Linux平台

x86_64 Linux hosted cross compilers

| 编译器名称         | 编译的目标平台 |
| ------------- | ------- |
| arm-none-eabi |         |

AArch64 Linux hosted cross compilers

| 编译器名称         | 编译的目标平台 |
| ------------- | ------- |
| arm-none-eabi |         |

##### Mac OS X平台

| 编译器名称         | 编译的目标平台 |
| ------------- | ------- |
| arm-none-eabi |         |

#### arm-none-eabi说明

arm-none-eabi用于编译ARM架构的裸机系统，包括ARM Linux的boot、kernel、不适用于编译Linux应用，使用的是专用于嵌入式系统的newlibc

### 2022年之后的Arm GNU Toolchain

2022年之后的Arm GNU Toochain进行了统一：

| 分类                | 适用处理器 | 支持的平台                  | 下载链接                                                              |
| ----------------- | ----- | ---------------------- | ----------------------------------------------------------------- |
| Arm GNU Toolchain | Arm系列 | Windows、Linux、Mac OS X | https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads |

版本：`12.2.Rel1`

#### Windows平台

| 编译器名称                    | 编译的目标平台                                  |              |
| ------------------------ | ---------------------------------------- | ------------ |
| arm-none-eabi            | AArch32 bare-metal target                | 32位纯裸机平台     |
| arm-none-linux-gnueabihf | AArch32 GNU/Linux target with hard float | 32位带硬件模式浮点运算 |
| aarch64-none-elf         | AArch64 bare-metal target                | 64位纯裸机平台     |
| aarch64-none-linux-gnu   | AArch64 GNU/Linux target                 | 64位Linux平台   |

#### Linux平台

x86_64 Linux hosted cross compilers

| 编译器名称                     | 编译的目标平台                                  |                  |
| ------------------------- | ---------------------------------------- | ---------------- |
| arm-none-eabi             | AArch32 bare-metal target                | 32位纯裸机平台         |
| arm-none-linux-gnueabihf  | AArch32 GNU/Linux target with hard float | 32位带硬件模式浮点运算     |
| aarch64-none-elf          | AArch64 bare-metal target                | 64位纯裸机平台         |
| aarch64-none-linux-gnu    | AArch64 GNU/Linux target                 | 64位Linux平台       |
| aarch64_be-none-linux-gnu | AArch64 GNU/Linux big-endian target      | 64位Linux平台（大端模式） |

AArch64 Linux hosted cross compilers

| 编译器名称                    | 编译的目标平台                                  |              |
| ------------------------ | ---------------------------------------- | ------------ |
| arm-none-eabi            | AArch32 bare-metal target                | 32位纯裸机平台     |
| arm-none-linux-gnueabihf | AArch32 GNU/Linux target with hard float | 32位带硬件模式浮点运算 |
| aarch64-none-elf         | AArch64 ELF bare-metal target            | 64位纯裸机平台     |

#### Mac OS X平台

macOS (x86_64) hosted cross toolchains

| 编译器名称            | 编译的目标平台                   |          |
| ---------------- | ------------------------- | -------- |
| arm-none-eabi    | AArch32 bare-metal target | 32位纯裸机平台 |
| aarch64-none-elf | AArch64 bare-metal target | 64位纯裸机平台 |

macOS (Apple silicon) hosted cross toolchains

| 编译器名称            | 编译的目标平台                   |          |
| ---------------- | ------------------------- | -------- |
| arm-none-eabi    | AArch32 bare-metal target | 32位纯裸机平台 |
| aarch64-none-elf | AArch64 bare-metal target | 64位纯裸机平台 |

## Codesourcery Toolchain

Codesourcery被Mentor收购，Mentor被Siemens收购，找不到下载地址。

## Linaro Toolchain

下载地址：https://www.linaro.org/downloads/#gnu_and_llvm
