---
id: vivado_bitstream_encryption
slug: /vivado_bitstream_encryption
title: Vivado bitstream 加密
authors: pansyhou
description: 详细介绍了在 Vivado 和 Petalinux 中为 Zynq SoC 的 bitstream 进行 AES 加密的步骤，包括使用 BBRAM 和 eFUSE 两种密钥存储方式。
keywords:
  - Zynq
  - Vivado
  - Petalinux
  - bitstream
  - AES 加密
  - BBRAM
  - eFUSE
  - FPGA 安全
---

对于高版本的 vivado，在中文论坛中的加密方式也许不再适用，xilinx 将这部分转移到了 bootgen 中

## vivado 中执行

:::warning 前置条件
需要安装 bootgen（在安装 vivado 时在 installer 中选择 bootgen）
 
一般默认安装，可以在 tcl console 中输入 `bootgen -help` 看看有没有反应
:::

1. 编辑 `secure.bif` 文件到 vivado 工程目录下
```
// arch = zynq; split = false; format = BIN
the_ROM_image:
{
    [aeskeyfile] secret_key.nky
    [bootloader, encryption=aes] zynq_fsbl.elf
    [encryption=aes] system.bit
    [encryption=aes] u-boot.elf
    [load=0x00100000] system.dtb
}
```

可以看到需要 `zynq_fsbl.elf`、`system.bit` 、 `u-boot.elf`、`system.dtb`

如果不需要系统的可以按需调整.bif 文件

**记得从 petalinux 的工程中复制上述文件到 vivado 工程目录下（.bit 可以 vivado 中导出）**

对于key，bootgen 会自己生成

2. 第一次生成，使用：`bootgen -image secure.bif -w -o BOOT.BIN -encrypt bbram -p xc7z010`（bbram 方式、器件选择为xc7z010 ，可以按照你的器件进行修改），这一步会生成 key
3. 之后的更改就不需要 `-p` 指定芯片型号了

### bbram 方式

第一次使用就用这个来调试

1. 将启动引脚调整至 `jtag` 启动，连接 jtag 、串口、电源
2. 复制 vivado 工程中生成的 `BOOT.bin` 到 sd 卡中
3. 在 Hardware Manager 里连接设备
![](https://pic1.imgdb.cn/item/692eccb77313ea6c250df78c.png)
1. 对器件右键选择 `program BBR Key`，会自动选择 nky 的
2. 不要断电，直接调整启动引脚到 SD 卡，按复位键
3. 看到 `DONE 灯亮起` + `串口有 uboot 输出`，即成功
4. 成功后可以按需选择 `电池供电` 或者用 `efuse`

### efuse

与上述同理

生成 efuse 版本的 BOOT.BIN 后，连接 jtag，上图的最下面有一个 `Program eFUSE Registers`

选择第一个的加密秘钥，下一步

Controll 部分寄存器可以跳过

## 在 petalinux 中（未测试）

:::warning 前置条件
 需要安装 bootgen
 
 ``` shell
 sudo apt install xilinx-bootgen
 ```
:::

cd 到 `project/images/linux`

1. 编辑 `secure.bif` 
```
// arch = zynq; split = false; format = BIN
the_ROM_image:
{
    [aeskeyfile] secret_key.nky
    [bootloader, encryption=aes] zynq_fsbl.elf
    [encryption=aes] system.bit
    [encryption=aes] u-boot.elf
    [load=0x00100000] system.dtb
}
```

可以看到需要 `zynq_fsbl.elf`、`system.bit` 、 `u-boot.elf`、`system.dtb`

如果不需要系统的可以按需调整.bif 文件

对于 key，bootgen 会自己生成

2. 第一次生成，使用：`bootgen -image secure.bif -w -o BOOT.BIN -encrypt bbram -p xc7z010`（bbram 方式、器件选择为 xc7z010 ，可以按照你的器件进行修改），这一步会生成 key
3. 之后的更改就不需要 `-p` 指定芯片型号了

后续烧录 key 方式和 vivado 中的一样[bbram 方式](# bbram 方式)





