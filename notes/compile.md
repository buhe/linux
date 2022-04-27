> https://www.raspberrypi.com/documentation/computers/linux_kernel.html

## 树莓派4 上编译

```
sudo apt install git bc bison flex libssl-dev make

git clone --depth=1 https://github.com/raspberrypi/linux
```

我是 32bit 树莓派4 如下：

```
cd linux
KERNEL=kernel7l
make bcm2711_defconfig
```

如何看自己的位宽呢？

```
uname -m
```

你的结果是 armv7l 或 armv8.

armv7l 是 32bit,而 armv8 是 64bit

```
make -j4 zImage modules dtbs
sudo make modules_install
sudo cp arch/arm/boot/dts/*.dtb /boot/
sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/overlays/
sudo cp arch/arm/boot/dts/overlays/README /boot/overlays/
sudo cp arch/arm/boot/zImage /boot/$KERNEL.img
```

重启

```
sudo reboot
```

