# 说明

下面的操作都是按照 `OpenJDK`官方的文档来的，使用的两个文档都可以在 `OpenJDK`源码中找到：

- `doc/building.md`，这个是编译源码的文档
- `doc/ide.md`，这个是导入到各种IDE中的文档

由于主要的目的是使用 `IntelliJ IDEA`查看 `JDK`中的 `Java`源码，不涉及到运行和调试，也不会看虚拟机部分的实现，故下面的操作使用官方文档中给出的最快速的编译步骤，并且会忽略部分报错。

主要步骤：

1. 编译源码
2. 导入 `IDEA`并进行配置

# 1. 编译源码

1. 克隆 `OpenJDK`源码: `git clone https://github.com/openjdk/jdk.git`
2. 进入 `jdk`目录: `cd jdk`
3. 执行命令: `bash configure`。如果执行下一步的命令报错，可以重新来，使用这个命令: `bash configure --disable-warnings-as-errors`
4. 执行命令: `make images`
5. 测试编译好的 `JDK`: `build/linux-x86_64-server-release/images/jdk/bin/java -version`
6. 跑一下最基本的测试: `make test-tier1`

# 2. 导入到IDEA并进行配置

1. 执行命令: `bash bin/idea.sh`
	- 如果这一步提示未安装 `ANT`的错误，则安装一下 `ANT`: `yay -S ant`，安装后再重新执行命令: `bash bin/idea.sh`
2. 导入 `IDEA`: `File -> Open...`，选择 `jdk`目录后打开
3. 在 `IDEA`中配置项目的 `SDK`: `File -> Project Structure -> Project`，在 `SDK`中选择上面编译好的 `OpenJDK`: `build/linux-x86_64-server-release/images/jdk`

# 编译OpenJDK 8

1. 切换到分支 `jdk8-b120`: `git checkout jdk8-b120`
2. 安装 `OpenJDK 8`作为编译时使用的 `JDK`: `yay -S jdk8-openjdk`
3. 安装 `gcc 4.9`: `yay -S gcc49`
4. 执行命令: `bash ./configure CC=gcc-4.9 CXX=g++-4.9`
5. 执行命令: `make CC=gcc-4.9 CXX=g++-4.9 all`
	- 如果有提示操作系统不支持的错误: `This OS is not supported: Linux fs 7.0.11-arch1-1`，需要在 `hotspot/make/linux/Makefile`文件中这一行: `SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3%`添加当前操作系统版本: `SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 3% 7%`
	- 如果有提示make语法不兼容，需要在 `hotspot/make/linux/makefiles/adjust-mflags.sh`文件中将这一行: `s/ -\([^ 	][^ 	]*\)j/ -\1 -j/`修改为: `s/ -\([^ 	I][^ 	]*\)j/ -\1 -j/`
	- 如果遇到警告信息导致报错: `all warnings being treated as errors`，需要将 `hotspot/make/linux/makefiles/gcc.make`文件中的这一行 `WARNINGS_ARE_ERRORS = -Werror`注释掉
	- 如果遇到时间错误: `time is more than 10 years from present`，将 `jdk/src/share/classes/java/util/CurrencyData.properties`这个文件中的所有年份修改为距离当前时间10年以内的年份即可
	- 如果遇到错误: `fatal error: sys/sysctl.h: No such file or directory`，需要将 `jdk/src/solaris/native/java/net/`目录下的 `PlainSocketImpl.c`和 `PlainDatagramSocketImpl.c`文件中的 `#include <sys/sysctl.h>`修改为 `#include <linux/sysctl.h>`
	- 如果遇到报错 `Running nasgen Exception in thread "main" java.lang.VerifyError: class jdk.nashorn.internal.objects.ScriptFunctionImpl overrides final method setPrototype.(Ljava/lang/Object;)V`，将 `nashorn/make/BuildNashorn.gmk`文件中的这一行 `-cp "$(NASHORN_OUTPUTDIR)/nasgen_classes$(PATH_SEP)$(NASHORN_OUTPUTDIR)/nashorn_classes" \`修改为 `-Xbootclasspath/p:"$(NASHORN_OUTPUTDIR)/nasgen_classes$(PATH_SEP)$(NASHORN_OUTPUTDIR)/nashorn_classes" \`
6. 编译好的 `JDK`在 `build/linux-x86_64-normal-server-release/images/j2sdk-image`目录下

# 编译OpenJDK 9

1. 切换到分支 `jdk-9+181`: `git checkout jdk-9+181`
2. 安装 `OpenJDK 8`作为编译时使用的 `JDK`: `yay -S jdk8-openjdk`
3. 安装 `gcc 4.9`: `yay -S gcc49`
4. 安装 `make 3.81`: `yay -S make-3.81-static`
5. 执行命令: `bash ./configure CC=gcc-4.9 CXX=g++-4.9 MAKE=make-3.81 --disable-warnings-as-errors`
6. 执行命令: `make-3.81 CC=gcc-4.9 CXX=g++-4.9 images`
	1. 如果遇到时间错误: `time is more than 10 years from present`，将 `jdk/make/data/currency/CurrencyData.properties`这个文件中的所有年份修改为距离当前时间10年以内的年份即可
7. 编译好的 `JDK`在 `build/linux-x86_64-normal-server-release/images/jdk`目录下