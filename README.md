# StMicroelectonics external tree

This repository contains the ST external tree to customize Buildroot.
It contains small and demo configurations for DK1 and DK2, that use Kernel, U-boot and TFA
from STMicroelectronics.

## Configurations details

* `st_stm32mp157a_dk1_defconfig & st_stm32mp157c_dk2_defconfig`:
  * Skeleton defconfig with only the basics configurations to make a DK2 boot properly.
* `st_stm32mp157a_dk1_demo_defconfig & st_stm32mp157c_dk2_demo_defconfig`:
  * OP-TEE with examples and tests
  * gcano-binaries to provide Opengl
  * Qt5 with examples
  * Use of external devicetrees imported from STM32CubeMX

## Build procedure

This external tree currently depends on our own Buildroot repository because there are few commit added to follow the ST needs.
The aim is to merge these additional commits to the mainline Buildroot in order to remove the dependency to our Buildroot repository.  
  
Clone the Buildroot from Bootlin Github repository alongside to this repository.  
Move to the Buildroot directory.  
Checkout to the same branch as this repository.

```bash
$ git clone git@github.com:bootlin/buildroot.git
$ cd buildroot
buildroot/ $ git checkout st/2021.02
```

Configure Buildroot to use the external tree and select the wanted defconfig.

```bash
buildroot/ $ make BR2_EXTERNAL=/path/to/st_external_tree st_stm32mp157c_dk2_defconfig
```

Compile Buildroot as usual

```bash
buildroot/ $ make
```

## Flash & boot the images

### Using SDCard
For the DK1 or the DK2, you simply need to flash the SDcard with the sdcard.img image file.

```bash
buildroot/ $ dd if=output/images/sdcard.img of=/dev/sdX bs=1M
```

Then configure the SW1 switch to boot on SDcard and finally power-on the board.

### Using STM32CubeProgrammer
Download the tool on the [ST website](https://www.st.com/en/development-tools/stm32cubeprog.html "STM32CubeProgrammer tool").  
The DK2 does not have emmc but only a SD removable device. We will use the tool to flash the SD card, but the process and logic would be the same for any other (non-removable) storage device.
A tutorial to use this tool on has been written in this [blog post](https://bootlin.com/blog/building-a-linux-system-for-the-stm32mp1-implementing-factory-flashing/ "Factory flashing a STM32").  

TL;DR
1. Switch the boot mode switches to USB boot.
2. Plug the second USB C on CN7.
3. Run these commands to flash the SDCard:
```bash
$ cd output/images/
$ sudo ~/stm32cube/bin/STM32_Programmer_CLI -c port=usb1 -w ../../../buildroot-external-st/board/stmicroelectronics/stm32mp157/flash.tsv
```
4. Switch back the boot mode switches to SD boot.
5. Boot on the SDCard

Take a look at the [user manual](https://www.st.com/resource/en/user_manual/dm00403500-stm32cubeprogrammer-software-description-stmicroelectronics.pdf "STM32CubeProgrammer User Manual") if you want to use GUI interface.  

## Demo image
The demo images have several features enabled, see the list above.
In this paragraph we will show you how to use them.

The login for all the images is `root` with no password.

### Use devicetree generated by STM32CubeMX
The [STM32CubeMX tool](https://www.st.com/en/development-tools/stm32cubemx.html "STM32CubeMX tool") allows you to automatize the generation of devicetrees with an easy to use interface.
You can configure the state of each pin of the stm32mp1 component.  
The tool generates devicetree for Linux, U-Boot and Arm-Trusted-Firmware (ATF).  
The two defaults devictrees examples for DK1 and DK2 generated by ST32CubeMX have been moved here: `board/stmicroelectronics/stm32mp157/\*-dts`
To use these devicetrees in Buildroot you need to change 2 things:
* The Buildroot defconfig. Update the devicetree name and the `BR2_*_CUSTOM_DTS_PATH` configurations, for the Kernel, U-Boot and ATF.
* Update the devicetree name described in the `extlinux.conf` file. This file is located in the overlay.

You might need to modify the post-image.sh script to use other naming of ATF binary.  
Please see the commit _configs/st: support external devicetrees for kernel, U-boot and ATF_ as an example of using devicetree generated by STM32CubeMX.

### Test DSI and HDMI Display
You can test the DSI and the HDMI display with the `modetest` command.  
First you need to find the ids of the display connectors, then show the test image on the dedicated display.

```
# modetest -c
...
Connectors:
id      encoder status          name            size (mm)       modes   encoders
32      0       connected       HDMI-A-1        480x270         5       31
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 1280x720 60.00 1280 1390 1430 1650 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #1 1280x720 50.00 1280 1720 1760 1980 720 725 730 750 74250 flags: phsync, pvsync; type: driver
  #2 800x600 75.00 800 816 896 1056 600 601 604 625 49500 flags: phsync, pvsync; type: driver
  #3 720x576 50.00 720 732 796 864 576 581 586 625 27000 flags: nhsync, nvsync; type: driver
  #4 720x480 59.94 720 736 798 858 480 489 495 525 27000 flags: nhsync, nvsync; type: driver
...
34      0       connected       DSI-1           52x86           1       33
  modes:
        index name refresh (Hz) hdisp hss hse htot vdisp vss vse vtot
  #0 480x800 50.00 480 578 610 708 800 815 825 839 29700 flags: ; type: preferred, driver
...

# modetest -s 34:480x800
# modetest -s 32:1280x720
``` 

### Use Wifi

The STM32MP157C-DK2 Discovery kit has Wifi feature available.  
To use Wifi you first need to load the `brcmfmac` module with the firmware configuration. For the demo I am using the Raspberrypi3 configuration firmware which is delivered in the linux-firmware package.

```
# cp /lib/firmware/brcm/brcmfmac43430-sdio.raspberrypi,3-model-b.txt /lib/firmware/brcm/brcmfmac43430-sdio.txt
# modprobe brcmfmac
```

Then you can scan close access points with `iw` command, and connect to them with `wpa_supplicant`
```
# ip link set wlan0 up
# iw dev wlan0 scan
# wpa_passphrase $SSID $PSK > /etc/wpa_supplicant.conf
# wpa_supplicant -i wlan0 -c /etc/wpa_supplicant.conf -B
# udhcpc -i wlan0 -q
```

### Qt examples

You can find the Qt examples at this path: `/usr/lib/qt/examples/`  
To run them you first need to load the galcore module.

```
# modprobe galcore
# /usr/lib/qt/examples/hellogl2/hellogl2
```

By default it will use HDMI if plugged or DSI otherwise. 
You can select the wanted display by using a KMS/DRM configuration file.  
You can see more information here: https://doc.qt.io/qt-5/embedded-linux.html#eglfs-with-the-eglfs-kms-backend

### OP-TEE examples

You can try the OP-TEE examples to verify it is working.
```
# optee_example_hello_world
# optee_example_random
# optee_example_aes
```

## References

https://buildroot.org/downloads/manual/manual.html#outside-br-custom  
https://bootlin.com/blog/tag/stm32mp1/
