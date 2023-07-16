# 编译的内容

- 系统引导程序bootloader
- 内核kernel
- 文件系统rootfs
- 驱动程序

# 步骤

电脑使用的系统是Arch Linux 64位

- 创建工作目录，用于存放将要编译的文件以及编译好的文件
- 安装交叉编译工具链
- 安装qemu
- 编译u-boot
- 编译内核
- 制作根文件系统
- 利用编译好的内核和根文件系统启动系统

## 创建工作目录

- 创建工作目录，用于存放将要编译的文件以及编译好的文件，该工作目录在当前用户主目录下，路径为`develop/arm/`，执行命令：`mkdir -p ~/develop/arm`

- 创建一个存放编译好的文件的目录，路径为`develop/arm/qemu_linux`，执行命令：`mkdir -p ~/develop/arm/qemu_linux`
  
  - 创建存放编译好的uboot的目录：`mkdir -p ~/develop/arm/qemu_linux/uboot`
  
  - 创建存放编译好的kernel的目录：`mkdir -p ~/develop/arm/qemu_linux/kernel`
  
  - 创建存放编译好的busybox的目录：`mkdir -p ~/develop/arm/qemu_linux/busybox`

- 创建存放uboot源码的目录：`mkdir -p ~/develop/arm/uboot`

- 创建存放kernel源码的目录：`mkdir -p ~/develop/arm/kernel`

- 创建存放busbox源码的目录：`mkdir -p ~/develop/arm/busybox`

## 安装交叉编译工具链

- 交叉编译工具链下载地址：https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads
- 交叉编译工具链版本：`12.2.Rel1`
- 交叉编译工具链下载文件：`arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf.tar.xz`
- 解压到安装目录：`sudo tar xJf arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf.tar.xz -C /opt`
- 配置环境变量，添加到`.zshrc`文件中：`export PATH=/opt/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf/bin:$PATH`
- 重新打开终端后，执行命令验证：`arm-none-linux-gnueabihf-gcc -v`

## 安装qemu

- 执行命令安装：`yay -S qemu-full`
- 可选安装GUI图形界面：`yay -S virt-manager`

## 编译u-boot

- 进入存放uboot源码的目录：`cd ~/develop/arm/uboot`
- u-boot下载地址：https://source.denx.de/u-boot/u-boot/-/tags 或者 https://github.com/u-boot/u-boot/tags
- 下载版本：`v2023.07.02`
- 解压下载的文件：`u-boot-2023.07.02.tar.gz`，
- 进入解压后的目录：`cd ~/develop/arm/uboot/u-boot-2023.07.02`
- 配置，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm vexpress_ca9x4_defconfig`，默认会在当前目录下生成一个`.config`配置文件，下面编译时会根据该配置文件进行编译
- 编译，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm`
- 在qemu中启动u-boot，执行命令：`qemu-system-arm -M vexpress-a9 -kernel u-boot -nographic -m 512M`
- 启动后，在输出的信息中最后看到`ERROR: can't get kernel image!`即可
- 退出qemu：`Ctrl+a`，松开后迅速按x键即可

## 编译内核

- 进入存放kernel源码的目录：`cd ~/develop/arm/kernel`

- linux内核下载地址：https://www.kernel.org/

- 下载版本：`6.1.38`

- 解压下载的文件：`linux-6.1.38.tar.xz`

- 进入解压后的目录：`cd ~/develop/arm/kernel/linux-6.1.38`

- 生成vexpress a9的配置文件，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm vexpress_defconfig`，默认会在当前目录下生成一个`.config`配置文件，下面编译时会根据该配置文件进行编译

- 可选的内核配置，如果不进行配置就跳过该步骤，内核配置的命令如下：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm menuconfig`

- 编译内核，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm`，编译完成后内核文件位于：`~/develop/arm/kernel/linux-6.1.38/arch/arm/boot/zImage`

- 将编译好的内核文件拷贝到工作目录中存放编译好的文件的目录：`cp ~/develop/arm/kernel/linux-6.1.38/arch/arm/boot/zImage ~/develop/arm/qemu_linux/kernel/`

- 将dtb文件拷贝到工作目录中存放编译好的文件的目录：`cp ~/develop/arm/kernel/linux-6.1.38/arch/arm/boot/dts/vexpress-v2p-ca9.dtb ~/develop/arm/qemu_linux/kernel/`

- 验证该内核是否能正常启动，执行命令：`qemu-system-arm -M vexpress-a9 -m 512M -kernel ~/develop/arm/qemu_linux/kernel/zImage -nographic -dtb ~/develop/arm/qemu_linux/kernel/vexpress-v2p-ca9.dtb -append "console=ttyAMA0"`

- 启动后，在输出的信息最后看到`Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)`即可

## 制作根文件系统

- 进入存放busybox源码的目录：`cd ~/develop/arm/busybox`

- busybox下载地址：https://busybox.net/

- 下载版本：`1.36.1`

- 解压下载文件：`busybox-1.36.1.tar.bz2`

- 进入解压后的目录：`cd ~/develop/arm/busybox/busybox-1.36.1`

- 生成配置文件，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm defconfig`，默认会在当前目录下生成一个`.config`配置文件，下面编译时会根据该配置文件进行编译

- 配置，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm menuconfig`
  
  - 打开静态编译选项：`Busybox Settings -> Build Options -> Build BusyBox as a static binary (no shared libs)`

- 编译，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm`

- 生成install，执行命令：`make CROSS_COMPILE=arm-none-linux-gnueabihf- ARCH=arm install`，会在当前目录下生成一个`_install`目录，该目录下的程序就是系统运行所需要的命令

- 进入工作目录中存放编译好的文件的目录：`cd ~/develop/arm/qemu_linux/busybox`

- 创建一个rootfs目录：`mkdir -p ~/develop/arm/qemu_linux/busybox/rootfs/{dev,etc/init.d,lib}`

- 将刚生成的`_install`目录下文件拷贝到rootfs目录下：`cp ~/develop/arm/busybox/busybox-1.36.1/_install/* -r ~/develop/arm/qemu_linux/busybox/rootfs/`

- （上面配置了静态编译，这一步复制就不需要了）（不使用静态编译，使用这一步复制，启动后会有报错导致无法进入系统，报错：`not syncing: No working init found`，还不知道是什么原因）从交叉编译工具链中拷贝库文件到rootfs目录下的lib目录中：`cp -P /opt/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf/arm-none-linux-gnueabihf/lib/* ~/develop/arm/qemu_linux/busybox/rootfs/lib/`

- 创建4个tty终端设备：
  
  - `sudo mknod rootfs/dev/tty1 c 4 1`
  
  - `sudo mknod rootfs/dev/tty2 c 4 2`
  
  - `sudo mknod rootfs/dev/tty3 c 4 3`
  
  - `sudo mknod rootfs/dev/tty4 c 4 4`

- 制作根文件系统镜像：
  
  - 生成一个512M大小的磁盘镜像：`dd if=/dev/zero of=disk.img bs=1M count=512`
  
  - 格式化成ext4文件系统：`mkfs.ext4 disk.img`
  
  - 将rootfs目录中的所有文件复制到磁盘镜像中：
    
    - 创建临时挂载点：`mkdir tmpfs`
    
    - 挂载：`sudo mount -o loop disk.img tmpfs/`
    
    - 复制文件：`sudo cp -r rootfs/* tmpfs/`
    
    - 卸载临时挂载点：`sudo umount tmpfs`
  
  - 检查根文件系统镜像：`file disk.img`，输出如下即可：`disk.img: Linux rev 1.0 ext4 filesystem data, UUID=c9d3c5c8-3d55-4d93-8af4-ff1b00a56026 (extents) (64bit) (lar  
    ge files) (huge files)`

## 利用编译好的内核和根文件系统启动系统



执行命令：`qemu-system-arm -M vexpress-a9 -m 512M -kernel ~/develop/arm/qemu_linux/kernel/zImage -nographic -dtb ~/develop/arm/qemu_linux/kernel/vexpress-v2p-ca9.dtb -append "root=/dev/mmcblk0 rw console=ttyAMA0" -sd ~/develop/arm/qemu_linux/busybox/disk.img`

## 利用编译好的uboot加载内核并启动系统