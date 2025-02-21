# BYAI-mcu_spi0
Out of the box, the BeagleY-AI's spi0 device is driven by an `spi-gpio` device (a bitbanged spi). This is a problem for SPI peripherals that require fast comms (displays, ADCs, etc.) since the bitbanged spi is limited to ~1MHz. Fortunately, these bitbanged spi0 pins are backed by an actual hardware SPI peripheral: `mcu_spi0`. This  guide shows how to activate it.

## Prerequisites
This guide was tested on a BeagleY-AI board running the 6.1.83-ti-arm64-r67 kernel version.

The required programs are `dtc`, `gcc`, and `make`.
Install `dtc` with `sudo apt install device-tree-compiler`.

You'll need a copy of the device-tree source for your kernel version. Clone the device tree source from [here](https://openbeagle.org/beagleboard/BeagleBoard-DeviceTrees) and switch to the v6.1.x-Beagle branch, or make a copy of the existing `/opt/source/dtb-6.1-Beagle` directory in your filesystem.

## 1. Adding a custom `spidev0` Device-Tree Overlay
Configuring the `mcu_spi0` peripheral with an associated `spidev` device requires a custom device tree overlay.

Copy the provided overlay file [`k3-am67a-beagley-ai-spidev0-mcu.dts`]() into the `src/arm64/overlays/` directory of your copy of the device-tree source.

## 2. Changing the spi0 Device-Tree Alias
The additional custom overlay file is enough to get the `mcu_spi0` peripheral working in place of the bitbanged spi. However, if you install the device tree as is, you will find that `mcu_spi0` gets assigned the devices `spidev1.0` and `spidev1.1` instead of the expected  `spidev0.0` and  `spidev0.1`. This is ultimately not a huge problem since those devices will still function properly in spite of their indices, but there is a fix for it.

It appears that a lingering reference in the device-tree keeps `spidev0` reserved to the bitbanged spi even though it isn't actually bound with any device. To fix this, we will need to modify the main device-tree source file [`src/arm64/ti/k3-am67a-beagley-ai.dts`]().
Open your copy of this file in a text editor and go to the section under `aliases`. This should be near the top of the file. In this section, change the line `spi0 = &spi_gpio` to `spi0 = &mcu_spi0`:
```
// DEVICE-TREE_SOURCE_ROOT/src/arm64/ti/k3-am67a-beagley-ai.dts
  ...
  aliases {
    serial0 = &wkup_uart0;
    serial2 = &main_uart0;
    serial3 = &main_uart1;
    serial6 = &main_uart6;
    mmc1 = &sdhci1;
    mmc2 = &sdhci2;
    rtc0 = &rtc;
  --spi0 = &spi_gpio;
  ++spi0 = &mcu_spi0;
    usb0 = &usb0;
    usb1 = &usb1;
    i2c1 = &mcu_i2c0;
  };
  ...
```
Now save and close the file.

## 3. Building and Installing the Modified Device-Tree
Change your working directory to the root of the device-tree source.

To build the device-tree, execute
```
$ make
```
To install the device-tree, execute
```
$ sudo make install_arm64
```
To activate the overlay, append `/overlays/k3-am67a-beagley-ai-spidev0-mcu.dtbo` to the line beginning with `fdtoverlays` in the last section of the file `/boot/firmware/extlinux/extlinux.conf`:  
```
// /boot/firmware/extlinux/extlinux.conf
...
label microSD (default)
  kernel /Image
  append console=ttyS2,115200n8 root=/dev/mmcblk1p3 ro rootfstype=ext4 resume=/dev/mmcblk1p2 rootwait net.ifnames=0 quiet
  fdtdir /
  fdt /ti/k3-am67a-beagley-ai.dtb
--fdtoverlays
++fdtoverlays /overlays/k3-am67a-beagley-ai-spidev0-mcu.dtbo
  initrd /initrd.img
```
Finally, reboot your system
```
$ sudo reboot
```
and verify that
```
$ ls /dev/spidev*
```
shows two devices named `spidev0.0` and `spidev0.1`.
