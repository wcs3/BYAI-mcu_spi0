# BYAI-mcu_spi0
Out of the box, the BeagleY-AI's spi0 device is driven by an `spi-gpio` driver (a bitbanged spi). This is a problem for SPI peripherals that require fast comms (displays, ADCs, etc.) since the bitbanged spi is limited to ~1MHz. Fortunately, these bitbanged spi0 pins just so happen to be backed by an actual hardware SPI peripheral: `mcu_spi0`. This guide shows how to activate it.

## Prerequisites
This guide was tested on a BeagleY-AI board running the 6.1.83-ti-arm64-r67 kernel version.

The required programs are `dtc`, `gcc`, and `make`.
Install `dtc` with `sudo apt install device-tree-compiler`.

You'll need a copy of the device-tree source for your kernel version. Clone the device tree source from [here](https://openbeagle.org/beagleboard/BeagleBoard-DeviceTrees) and switch to the v6.1.x-Beagle branch, or make a copy of the existing `/opt/source/dtb-6.1-Beagle` directory in your filesystem. You can place this copy anywhere that is convenient (and you may delete it once you have everything installed).

## 1. Adding a custom `spidev0` Device-Tree Overlay
Configuring the `mcu_spi0` peripheral with an associated `spidev` device requires a custom device tree overlay.

Copy the provided overlay file [`k3-am67a-beagley-ai-spidev0-mcu.dts`](/k3-am67a-beagley-ai-spidev0-mcu.dts) into the `src/arm64/overlays/` directory of your copy of the device-tree source.

## 2. Changing the spi0 Device-Tree Alias
The additional custom overlay file is enough to get the `mcu_spi0` peripheral working in place of the bitbanged spi. However, if you install the device tree as is, you will find that `mcu_spi0` gets assigned `spidev1` (spi bus number 1) instead of the expected `spidev0` (bus number 0). This is ultimately not a huge problem since those devices will still function properly in spite of their bus numbers, but since we wish to retain the original behavior of the spi configuration, we will fix it.

If you open the main device-tree source file `src/arm64/ti/k3-am67a-beagley-ai.dts` in a text editor and go to the section under `aliases`, you'll notice the line `spi0 = &spi_gpio`:
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
    spi0 = &spi_gpio;
    usb0 = &usb0;
    usb1 = &usb1;
    i2c1 = &mcu_i2c0;
  };
  ...
```
This is the cause of our problem. When assigning bus numbers to SPI controllers, the [linux spi driver](https://github.com/torvalds/linux/blob/27102b38b8ca7ffb1622f27bcb41475d121fb67f/drivers/spi/spi.c#L3281) checks the device-tree aliases to see if there exists a `spi<n>` alias for the controller. If there is a `spi<n>` alias for the controller, then the controller is assigned bus number `n`. If there is no `spi<n>` alias for the controller, then the controller gets the bus number `1 + max(highest n of spi<n> aliases, highest assigned bus number)`. 

Since `spi_gpio` has alias `spi0`, bus number 0 will always be reserved for it. OTOH, `mcu_spi0` currently doesn't have an alias, and no other controllers are registered, so it gets bus number 1.

Thus, the fix is to simply update the `spi0` alias. Open your copy of the file `src/arm64/ti/k3-am67a-beagley-ai.dts` in a text editor and change the offending line to `spi0 = &mcu_spi0`:
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
Save and close the file.

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

## Appendix
### SPI Transfer Limit
`spidev` devices are by default limited to transferring 4096 bytes at a time. Some transfers, like flushing frames to an LCD, require a much larger transfer size. The obvious workaround is to split up large transfers into many smaller transfers. The downside to this approach is that there may be some non-negligible overhead in setting up each individual transfer, which is something we would hope to avoid if we wish to squeeze the most performance out of the SPI peripheral. Instead, we can just configure the `spidev` limit to our desired value.

To change the transfer limit of `spidev` devices, open the file `/boot/firmware/extlinux/extlinux.conf` and edit the line in the last section beginning with `append`:
```
// /boot/firmware/extlinux/extlinux.conf
...
label microSD (default)
  kernel /Image
--append console=ttyS2,115200n8 root=/dev/mmcblk1p3 ro rootfstype=ext4 resume=/dev/mmcblk1p2 rootwait net.ifnames=0 quiet
++append console=ttyS2,115200n8 root=/dev/mmcblk1p3 ro rootfstype=ext4 resume=/dev/mmcblk1p2 rootwait net.ifnames=0 quiet spidev.bufsiz=<NEW_SPIDEV_LIMIT>
  fdtdir /
  fdt /ti/k3-am67a-beagley-ai.dtb
  fdtoverlays
  fdtoverlays /overlays/k3-am67a-beagley-ai-spidev0-mcu.dtbo
  initrd /initrd.img
```
Replace `<NEW_SPIDEV_LIMIT>` with your desired value. I'm not sure if there is an eventual hardware limit to the transfer size, but I have at least tested it up to 2^17 = 131072 bytes (enough to transfer a whole frame of a 240x240 16-bit color display).

After making the above change, save the file and reboot your board. You can verify that the change applied by checking the output of
```
$ cat /sys/module/spidev/parameter/bufsiz
```

### Interword Delay
If you set the spi clock speed to say 48MHz, you would expect a byte throughput of 48MHz/8 = 6 million bytes / sec. What you will instead find is that the actual byte throughput is roughly half that. This is due to a synchronization delay being inserted between each byte being transferred. Though each byte itself is transferred at the set clock speed, the interword delay results in an overall decreased throughput. 

A [suggested method](https://e2e.ti.com/support/processors-group/processors/f/processors-forum/1356551/faq-am6x-optimizing-spi-transfer-inter-byte-gaps-using-the-dma-in-linux) to decrease this delay is to configure the SPI controller to use DMA. Unfortunately, it appears that only the `main_spi*` controllers can be configured to use DMA, so we can't use this method for `mcu_spi0`.

My suggestion to optimizing the interword delay is to conduct transfers with larger words. Consider a scenario where we want to transmit a 16-byte buffer. If we do this with 8-bit words, then 16 words are needed for the 16-byte buffer, incurring 15 interword delays. OTOH, with 32-bit words, only 4 words are needed for the 16-bytes, so there would only be 3 delays. I found that the interword delay is roughly equal to the transmission time of an 8-bit word, or 8 spi clock cycles. So with 8-bit words and a clock frequency of 48MHz, a 16-byte transfer would take `8*(16+15)/48000000 = 5.17uS`. With 16-bit words, it would take `8*(16+7)/48000000 = 3.83uS`, and with 32-bit words, only `8*(16+3)/48000000 = 3.17uS`. 

The problem with changing the transfer word-size is alignment and byte-ordering. Your buffer size must be divisible by the word size, otherwise you will have to split off the remainder into a seperate transfer at a smaller word size. Moreover, you must take care to set up your data in the transfer buffer so that when it is transferred with larger words, the byte ordering at the receiver is still correct.   

## References
[AM67x Processors datasheet](https://www.ti.com/lit/ds/symlink/am67a.pdf?ts=1740114925407&ref_url=https%253A%252F%252Fpinout.beagleboard.io%252F)

[SPI Enablement and Validation on TDA4 Family](https://www.ti.com/lit/an/sprad26/sprad26.pdf?ts=1740138654464&ref_url=https%253A%252F%252Fwww.ti.com%252Fproduct%252FAM67)

[devicetree Specification](https://www.devicetree.org/specifications/)

[BeagleY-AI Pinout](https://pinout.beagley.ai/)

[BeagleBoard Device-Tree Source](https://git.beagleboard.org/beagleboard/BeagleBoard-DeviceTrees)

[Configuring spidev.bufsiz on Raspberry Pi](https://forums.raspberrypi.com/viewtopic.php?t=124472)
