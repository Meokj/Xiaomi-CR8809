# Xiaomi-CR8809 A版 M79 USB转TTL救砖

小米路由器有4个未焊接孔位，当它处于右上角时，从左到右分别是`VCC`，`RXD`，`GND`，`TXD`，RXD接USB上的TXD，TXD接USB上的RXD，使用3.3v电压

## 1. 网络与环境变量设置（路由器断电重启，必须重新执行以下命令）
本步骤用于配置开发板与 TFTP 服务器（通常是你的 PC）之间的网络通信，确保可以正常下载固件。

```bash
# 1.1 查看当前设备所有的环境变量设置，确认当前网络参数
printenv

# 1.2 设置开发板（客户端）的 IP 地址
setenv ipaddr 192.168.10.1

# 1.3 设置 TFTP 服务器（PC端）的 IP 地址
setenv serverip 192.168.10.101

# 1.4 设置子网掩码
setenv netmask 255.255.255.0

# 1.5 设置网关 IP 地址
setenv gatewayip 192.168.10.1

# 1.6 指定当前处于激活状态的网卡接口（此处指定为 eth1）
setenv ethact eth1

# 1.7 再次查看环境变量，检查刚刚设置的参数是否生效
printenv

# 1.8 测试网络连通性，尝试 ping 服务器 IP（必须返回 host is alive 才能继续后续步骤）
ping 192.168.10.101
```

## 2. U-Boot 合并分区与 Web U-Boot 刷入
本步骤用于更新分区表（MIBIB）并刷入带有 Web 恢复界面的新型 U-Boot（APPSBL）。请严格执行断电重启步骤。

```bash
# ==================== 阶段一：重构分区表 ====================

# 2.1 从 TFTP 服务器下载分区表文件 MIBIB.bin，并烧录到 0:MIBIB 分区
# （注：此步仅仅改变了 Flash 的分区规划，此时设备内还没有新的 U-Boot）
tftpboot MIBIB.bin && flash 0:MIBIB

# 【⚠️核心安全操作】：
# 刷完 MIBIB 之后，必须立即断电重启！
# 目的是让芯片硬件重新加载刚刚写入的新分区表。
# 重新通电时，在串口终端连按任意键，再次拦截进入旧的 U-Boot 命令行，继续执行阶段二。


# ==================== 阶段二：刷入全新引导程序 ====================

# 2.2 下载并刷入全新的主引导程序（APPSBL.bin）到 0:APPSBL 分区
tftpboot APPSBL.bin && flash 0:APPSBL

# 2.3 下载并刷入全新的备用引导程序（APPSBL1.bin）到 0:APPSBL1 分区
# （注：到这一步执行完毕并再次重启后，你才算真正拥有了“带 Web 的 U-Boot”）
tftpboot APPSBL1.bin && flash 0:APPSBL1
```

## 3. RootFS（系统固件）分区刻录 (无法Web访问时，可执行1和3步骤)
本步骤用于清理原有的系统分区，并通过内存将全新的系统固件写入 NAND Flash 中。

```bash
# 3.1 查看当前系统的内存与分区映射表（确认 rootfs 分区所处的目标地址与大小）
smem

# 3.2 擦除原有的 rootfs 数据（从偏移量 0xa80000 开始，擦除大小为 0x7480000 的整块区域）
nand erase 0xa80000 0x7480000

# 3.3 从 TFTP 服务器将系统固件 factory.ubi 下载到内存的 0x44000000 地址处
tftpboot 0x44000000 factory.ubi

# 3.4 将内存中的固件写入 NAND Flash（从内存 0x44000000 读取，写入到 Flash 的 0xa80000 位置，写入长度为实际文件大小也就是具体的下载固件大小这里例如0xb60000）
nand write 0x44000000 0xa80000 0xb60000

# 3.5 重启开发板，系统将引导进入刚刚刷入的新固件中
reset
```
