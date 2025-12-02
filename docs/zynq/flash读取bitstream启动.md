---
id: flash_read_bitstream
slug: /flash_read_bitstream
title: flash读取bitstream启动
authors: pansyhou
---

# 法一：在 uboot 里将 bitstream 写到 flash 后修改 boot.scr 启动


在 Uboot 里：

```shell
# 1. 从 SD 卡读取 bit 文件到内存
fatload mmc 0 0x100000 system.bit

# 2. 探测 Flash
sf probe 0 0 0

# 3. 擦除指定区域 (假设 bit 文件小于 4MB，我们放在 0x900000 这个位置)
# 请确保这个地址没有和 U-Boot/Kernel 冲突
sf erase 0x900000 0x400000 

# 4. 写入 Flash
sf write 0x100000 0x900000 ${filesize}
```


修改 boot.cmd.default，将 fpga load 部分修改如下

```shell
#add fpga part 
# 修改逻辑：不再检测文件，直接从 Flash 读取 
# 注意：你需要硬编码 bit 文件的大小，或者预估一个足够大的值 
# 假设 bit 文件大概 4MB (0x400000)，存放在 0x900000 
echo "Loading FPGA from QSPI Flash 0x900000..." 
sf probe 0 0 0 
# 读取 Flash 到 RAM 0x00800000，长度 0x400000 (4MB) 
sf read 0x00800000 0x900000 0x400000 
# 加载 FPGA (注意：loadb 支持带有 bitstream header 的数据) 
# 如果你知道确切大小最好，不知道的话，loadb 通常能识别 header 里的长度 
fpga loadb 0 0x00800000 0x400000
```

