## petalinux 下载

本文基于 wsl2 Ubuntu-22.04.5，24.04 也许会有一点点问题（具体来说不知道啥情况，反正要试的话最好还是下载sstate cache）

https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools/archive.html

下载好后，找到.run

可以提前下载sstate cache，藏在最下面了，下面的是arm的（zynq 7000 series）

https://china.xilinx.com/member/forms/download/xef.html?filename=sstate_arm_2023.1_05010539.tar.gz

:::tip
最好还是下载sstate，配置环境速度会快很多
:::

### 安装依赖


:::warning
记得提前换源
:::

``` shell
sudo dpkg --add-architecture i386
sudo dpkg-reconfigure dash
```

```
sudo apt update 
```

```
sudo apt-get update && sudo apt-get install -y \
  iproute2 make libncurses5-dev tftpd libselinux1 wget diffstat chrpath socat tar unzip gzip tofrodos \
  debianutils iputils-ping libegl1-mesa libsdl1.2-dev pylint python3 python2 cpio gnupg zlib1g:i386 haveged perl \
  lib32stdc++6 libgtk2.0-0:i386 libfontconfig1:i386 libx11-6:i386 libxext6:i386 libxrender1:i386 libsm6:i386 \
  xinetd gawk gcc net-tools ncurses-dev openssl libssl-dev flex bison xterm autoconf libtool texinfo zlib1g-dev cpp-11 patch diffutils \
  gcc-multilib build-essential automake screen putty pax g++ python3-pip xz-utils python3-git python3-jinja2 python3-pexpect \
  liberror-perl mtd-utils xtrans-dev libxcb-randr0-dev libxcb-xtest0-dev libxcb-xinerama0-dev libxcb-shape0-dev libxcb-xkb-dev \
  openssh-server util-linux sysvinit-utils google-perftools \
  libncurses5 libncursesw5-dev libncurses5:i386 libtinfo5 \
  libstdc++6:i386 dpkg-dev:i386 \
  ocl-icd-libopencl1 opencl-headers ocl-icd-opencl-dev
```

会出现缺少 `zlib1g:i386` 的情况，

### 安装 .run

2023.1:

 ```
 sudo chown -R $USER:$USER /opt &&
 mkdir -p /opt/pkg/petalinux/2023.1 &&
 chmod +x petalinux-v2023.1-05012318-installer.run &&
 ./petalinux-v2023.1-05012318-installer.run -d /opt/pkg/petalinux/2023.1
 ```

2020.2：

 ```
 sudo chown -R $USER:$USER /opt &&
 mkdir -p /opt/pkg/petalinux/2020.2 &&
 chmod +x petalinux-v2020.2-final-installer.run &&
 ./petalinux-v2020.2-final-installer.run -d /opt/pkg/petalinux/2020.2
 ```


弹出协议后，按 q 退出查看协议，随后输入 y 同意，第二个同理

```shell
Use PgUp/PgDn to navigate the license viewer, and press 'q' to close

Press Enter to display the license agreements
Do you accept Xilinx End User License Agreement? [y/N] > y
Do you accept Third Party End User License Agreement? [y/N] > y
INFO: Installing PetaLinux...
INFO: Checking PetaLinux installer integrity...
INFO: Installing PetaLinux SDK to "/opt/pkg/petalinux/2023.1/."
INFO: Installing buildtools in /opt/pkg/petalinux/2023.1/./components/yocto/buildtools
INFO: Installing buildtools-extended in /opt/pkg/petalinux/2023.1/./components/yocto/buildtools_extended
INFO: PetaLinux SDK has been installed to /opt/pkg/petalinux/2023.1/.
```

### 配置启动环境 alias

2023.1:

```shell
cd /opt/pkg/petalinux/2023.1/ &&
source settings.sh &&
echo $PETALINUX &&#检查一下路径有没有 &&
echo "alias sptl='source $PETALINUX/settings.sh'" >> ~/.bashrc 
```

2020.2:

```shell
cd /opt/pkg/petalinux/2020.2/ &&
source settings.sh &&
echo $PETALINUX &&#检查一下路径有没有
echo "alias sptl='source $PETALINUX/settings.sh'" >> ~/.bashrc
```
### (wsl2下需要)调整 locale


```shell
environment: line 96: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
[INFO] Sourcing buildtools
[INFO] Getting hardware description...
INFO: Renaming system_wrapper.xsa to system.xsa
petalinux-yocto: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
petalinux-yocto: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
[INFO] Extracting yocto SDK to components/yocto. This may take time!
/bin/bash: warning: setlocale: LC_ALL: cannot change locale (en_US.UTF-8)
[INFO] Generating Kconfig for project
```

编辑 locale 配置文件：

```
sudo nano /etc/locale.gen
```

**取消注释** 包含 `en_US.UTF-8` 的行（删除行首的 `#`）。

```
en_US.UTF-8 UTF-8
```

**生成语言环境：**

```
sudo locale-gen
```

## sstate cache配置

```bash
petalinux-config
```

找到 Yocto Settings -> Local sstate feeds settings，设置为你的sstate解压之后的文件夹

比如我的: `(/home/pansy/cache/sstate-cache) local sstate feeds url`


## 优化启动流程编译（2023.1）

下面介绍正点原子中介绍的优化zynq的启动流程方法，该方法将比特流（vivado可生成）和内核设备树（petalinux生成）从BOOT.BIN中独立出来。

方便修改：bitstream 设备树 就这两。

:::warning

我好像很久没有再看这套流程了，正点原子的方法内核都是单独下载的，现在我已经默认使用petalinux2023里自带的6.1内核

请辩证看待每一步是否正确

个人认为正点原子的方案脱离了petalinux的yocto大锅炖的理念了

最新的启动方案是这个：[基于网络的启动方式](https://pansyhou.top/docs/zynq/基于网络的启动方式)

:::

如果还想修改内核，可以参考更轮椅的流程，将 `kernel` ` rootfs`  `system.bit` `system.dtb` 改为全从电脑的服务器上读取。

[基于网络的启动方式](https://pansyhou.top/docs/zynq/基于网络的启动方式)

方便修改：bitstream 设备树 内核 rootfs 一整套。

:::note
1. 将 `system.bit` 从 BOOT.BIN 独立出

	1. BOOT.BIN 包含：fsbl、uboot、**system.dtb**（缺 dtb 会使得 uboot 不能输出串口）

2. 将 image.ub 分开为 `内核 zImage ` 和 `设备树 dtb`

:::

1. 新建工程
```shell
petalinux-create -t project --template zynq -n 工程名
```

2. 将.xsa 丢进去
```shell
petalinux-config --get-hw-description
```


:::warning
注意

- 记得调整启动方式为 EXT4
- 启动镜像为 zImage
- yocto 的镜像源 http 改成 https（两个源）

你在启动 kernel 的时候发现用户是 `oe-user@oe-host` 那就是用的 petalinux 自带的 kernel，你自己编译的是你的 linux 用户名和机器

:::


:::note
如果要重新配置

```shell
petalinux-config
```
:::

3. 编译 BootLoader

```shell 
petalinux-build -c bootloader
```

4. 编译 u-boot

```shell 
petalinux-build -c u-boot
```

5. 打包

~~注意，petalinux 工具在打包的时候会自动把 dtb 文件加进入，所以这里加入“--dtb no”参数把 dtb 文件除去。~~

不加入设备树，uboot 就没办法输出了

`BOOT.BIN` 中包含的 DTB 是用来**支持 U-Boot 自身的启动**的，它确保 U-Boot 能够正确地初始化硬件，从而进入到可以执行命令的阶段。

而 `boot.scr` 中的 DTB 是用来**支持 Linux 内核的启动**的，它告诉 Linux 内核如何使用 DTB 来识别和配置硬件。

所以，即使你的 `boot.scr` 脚本写得再完美，如果 `BOOT.BIN` 本身存在问题，系统就会在更早的阶段卡住，你自然就看不到任何串口输出了。

```shell
petalinux-package --boot --fsbl --u-boot --force
```


7. 生成 boot.scr
```shell
petalinux-build
```

然后删除 boot.scr 的第一行，之后将第 6 行的 uImage 改成 zImage

在下面图示地方加上 fpga 加载部分

```
if test -e ${devtype} ${devnum}:${distro_bootpart} /system.bit; then

fatload ${devtype} ${devnum}:${distro_bootpart} 0x00800000 system.bit;

fpga loadb 0 ${fileaddr} ${filesize}

fi
```

:::note

注意，petalinux2023.1 的 boot.scr 修改与 2020 有点不一样

修改的地方：

1. kernel name 的定义改成 zImage

2. 不是全部的 bootm 都改成 bootz，只有 mmc0 那一个启动项的最后一个 bootm 改成 bootz

![](https://pic1.imgdb.cn/item/698885ba956625ad535775a7.png)

:::

```shell
mkimage -c none -A arm -T script -d boot.cmd.default boot.scr
```

在 Petalinux 工程中执行编译 uboot 后，会在工程的“components/plnx_workspace/device-tree/device-tree/”目录下生成设备树文件

![](https://pic1.imgdb.cn/item/698885d0956625ad535775a9.png)

8. 编译内核

下载对应版本的内核

[Xilinx/linux-xlnx at xlnx_rebase_v6.1_LTS_2023.1](https://github.com/Xilinx/linux-xlnx/tree/xlnx_rebase_v6.1_LTS_2023.1)


复制设备树到内核源码“arch/arm/boot/dts”目录下

```shell
pansy@pansy11:~/peta/scanner_7020/components/plnx_workspace/device-tree/device-tree
$ cp pcw.dtsi pl.dtsi system-top.dts zynq-7000.dtsi system-conf.dtsi ~/peta/linux-xlnx/arch/arm/boot/dts/
```


同时将自己的 system-user.dtsi 复制进去

```shell
pansy@pansy11:~/peta/scanner_7020/project-spec/meta-user/recipes-bsp/device-tree/files$ 
cp system-user.dtsi ~/peta/linux-xlnx/arch/arm/boot/dts/
```

修改个人 dts 部分跳过

修改 makefile 加入刚刚的 dts （图好像有错）

![](https://pic1.imgdb.cn/item/698885e8956625ad535775ab.png)

:::note
需要按照第十章的 10.4 安装 petalinux 的 sdk 才进行 make xilinx_zynq_defconfig
:::

### 编译 sdk

回到工程文件夹

``` shell
petalinux-build --sdk
```

成功构建的 SDK 文件在工程的 images/linux 目录下，名为 sdk.sh

``` shell
cd images/linux/
./sdk.sh
```

保持默认，yes 即可

``` shell
cd /opt/petalinux/2023.1/
source environment-setup-cortexa9t2hf-neon-xilinx-linux-gnueabi
#下面这个会把交叉编译链直接写到bashrc里，如果不想每次编译内核时找gcc时可以直接用这个
echo '. /opt/petalinux/2023.1/environment-setup-cortexa9t2hf-neon-xilinx-linux-gnueabi' | tee -a ~/.bashrc
```

### 编译内核

回到内核目录

给予脚本执行权限（如果是直接 git clone 应该不用）

```shell
cd ~/peta/linux-xlnx-xlnx_rebase_v5.4_2020.1 
chmod +x scripts/*.sh
```


```shell
make xilinx_zynq_defconfig
```

开始编译

```shell
make -j8
```

编译好后 zImage 在 `arch/arm/boot`，设备树镜像文件 system-top.dtb 在 `arch/arm/boot/dts`


9. 编译根文件系统 rootfs

```shell
petalinux-config -c rootfs
```


```shell
petalinux-build -c rootfs
```


生成之后我们只用rootfs.tar.gz 文件，解压到 SD 卡 ext4 上

最后将 5 个文件：BOOT.BIN、system.bit、system.dtb、boot.scr、zImage 丢入 FAT32 分区启动即可

from peta: BOOT.BIN、boot.scr、system.bit
from kernel: system.dtb(\peta\linux-xlnx\arch\arm\boot\dts)、zImage(\peta\linux-xlnx\arch\arm\boot)

:::warning 注意

记得别把 image.ub 放进去，FIT Image 在 boot.scr 的启动顺序是很靠前的~

:::

:::note 登录问题

2023.1 的 User 是 petalinux，首次登入会让你修改密码
不过也可以在config里设置成root免密码登录

:::

### 重新编译设备树

先删除

```shell
petalinux-build -c device-tree -x cleansstate
```


```shell
petalinux-build -c device-tree -x cleansstate
```


```shell
petalinux-package --boot --fsbl --u-boot --force
```


## 会碰到的一些问题


### cracklib fetch fail

因为上游的 branch 名因为 zz 原因从 master 改成 main ，不能找到正确的分支名

我们需要手动修改 components/yocto/layers/poky/meta/recipes-extended/cracklib/cracklib_2.9.8.bb 这里的

把

SRC_URI = “git:// [github.com/cracklib/cracklib;protocol=https;branch=master](https://github.com/cracklib/cracklib;protocol=https;branch=master) \

改成

SRC_URI = “git:// [github.com/cracklib/cracklib;protocol=https;branch=main](https://github.com/cracklib/cracklib;protocol=https;branch=main) \

[尊敬的先生：我们在最后一步 petalinux-build 使用 petalinux v 2023.1 配置 petalinux 环境时遇到错误，错误如下：](https://adaptivesupport.amd.com/s/question/0D54U00008myMvoSAE/dear-sir-we-got-an-error-while-configuring-petalinux-environment-with-petalinux-v-20231-at-the-last-step-petalinuxbuild-the-error-is-as-follows?language=en_US)
