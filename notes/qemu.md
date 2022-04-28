## qemu 开发内核环境搭建

```
qemu-system-arm -M ?
```

```
KERNEL=kernel7
make bcm2709_defconfig
```

### 编译

```
make -j4 zImage modules dtbs
```

### 使用该内核

```
cp arch/arm/boot/zImage ~/kernel7.img
git clone https://github.com/dhruvvyas90/qemu-rpi-kernel.git
cp qemu-rpi-kernel/linux/arch/arm/boot/dts/bcm2709-rpi-2-b.dtb ~/bcm2709-rpi-2-b.dtb
```

### 下载并解压镜像

```
wget https://downloads.raspberrypi.org/raspios_armhf/images/raspios_armhf-2022-04-07/2022-04-04-raspios-bullseye-armhf.img.xz
unxz 2022-04-04-raspios-bullseye-armhf.img.xz
```

### 启动 qemu

```
qemu-system-arm -M raspi2 -kernel kernel7.img -sd 2017-11-29-raspbian-stretch.img -append "root=/dev/mmcblk0p2 rootwait" -dtb bcm2709-rpi-2-b.dtb -m 512M
```

