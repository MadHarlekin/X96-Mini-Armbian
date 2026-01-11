
Maybe you are like me and like to play with random hardware and wonder how far you can get it. 

The X96-Mini i found in a 4-pack for 15€ on ebay. According to the datasheet each box comes with Quad-Core A53, a small Mali-GPU, 2GB Ram and 16Gb emmc. Enough for some fun little testing like a docker swarm or similar.  

I used Armbian and it's flashing tool to get a current image installed on a usb. As it is community driven there is a bit of extra work to do: 
1. Find the right dtb (device tree blob) and link it in the extlinux.conf fdt-section
2. Rename the correct binary to "u-boot.ext"
3. Hope that it boots with the toothpick method!
Inital tests didn't work due to the first finding: 

## 1. Rebrands, Lies and NANDs: 

![CPU](https://github.com/MadHarlekin/X96-Mini-Armbian/blob/main/SOC.png)


The provided dtb-files will not work e.g. S905X-p212.dtb (which is usually good guess) or S905W-p281.dtb as this is not a S905W but a 905L3 (the edging on the left side) which can use the S905L-p271.dtb. 
From what i gathered, the supplier edged a wrong name on the chip on purpose to save money, as the L-Variant has  less capabilities than the S905W. 

![MB](https://github.com/MadHarlekin/X96-Mini-Armbian/blob/main/X96-Board.png)

The emmc is in fact a SK Hynix H26M / **H26M52103FMR** [Datasheet](https://www.alldatasheet.net/datasheet-pdf/view/1425081/HYNIX/H26M52001FMR.html) but that will only come in handy later. 

With the provided dtb we can successfully boot into armbian, great. But wait... where is everything? No HDMI, no emmc.... no wifi? Only ethernet and SSH works? 


## 2.DTS:

Welcome to the world of dts/dtb and the simple example of "frankenboards". The good news first, the clock-generators and power-supply-points are usually correct... for the most part as it is basically a cut down version of the S905X2/3! 

The ethernet works but all other pointers and values to wifi, storage and graphics are not correct. 

So let's take our first problem, the storage. 

So in armbian we can check what the OS knows about the emmc: 

dmesg | grep -i mmc 

Which in my cased revealed: 

```
[ 10.123456] mmc1: mmc_select_hs200 failed, error -84 
[ 10.123500] mmc1: error -84 whilst initialising eMMC card 
[ 10.150000] mmcblk1: error -5 sending status command, retrying 
[ 10.160000] mmcblk1: error -5 sending stop command, original command response 0x900, card status 0x900 
[ 10.170000] blk_update_request: I/O error, dev mmcblk1, sector 2048 
[ 10.180000] Aborting journal on device mmcblk1p2-8. 
```
So CRC and I/O-Error and a final timeout. The chip is not happy with the status quo. It isn't broken, something in the communication is not to the liking of the SoC.

The solution was to fingerprint (see the emmc-chip datasheet) and adjust/force certain points in the device blob tree. 

In the original dtb the emmc had max-frequency of 200Mhz via "mmc-hs200-1_8v" and usage of 1.8V. Because those boards aren't too great in quality sometimes, lowering values or disabeling certain modes. 

emmc usually splits itself in: 
Low: 25-26Mhz
Middle: 50-52Mhz
HS200-HS400: 100-200Mhz

So if the 100Mhz isn't working maybe we can just try to go slower. Let's see if we can adjust that frequency. 



Let's turn the dtb into a workable format first of all with dtc (device-tree-compiler): 
```
dtc -I dtb -O dts -o my-test.dts meson-gxlx-s905l-p271.dtb

nano/vim/whatevereditoryoulike my-test.dts
```
I adjusted my values to the following: 

```
mmc@74000 {
				compatible = "amlogic,meson-gx-mmc", "amlogic,meson-gxbb-mmc";
				reg = <0x00 0x74000 0x00 0x800>;
				interrupts = <0x00 0xda 0x04>;
				status = "okay";
				clocks = <0x03 0x60 0x03 0x7d 0x03 0x04>;
				clock-names = "core", "clkin0", "clkin1";
				resets = <0x11 0x2e>;
				pinctrl-0 = <0x29>;
				pinctrl-1 = <0x2b>;
				pinctrl-names = "default", "clk-gate";
				bus-width = <0x08>;
				cap-mmc-highspeed;
				max-frequency = <0x17d7840>;
				non-removable;
				disable-wp;
				mmc-pwrseq = <0x2c>;
				vmmc-supply = <0x2d>;
				vqmmc-supply = <0x25>;
				phandle = <0xa0>;
			};
```


Looks a bit much in the beginning but important are the following: 
[Reference for mmci-values](https://www.kernel.org/doc/Documentation/devicetree/bindings/mmc/mmci.txt) 

max-frequency: in this picture set to 25Mhz (in hex) as the lowest standard, just for initial testing. 

phandle: Very important as a pointer within the device trees. It is the unique identifier for devices. Great example, where is the emmc getting it's power from? vmmc-supply <0x2d> which corresponds to: 

```
regulator-vcc-3v3 {
		compatible = "regulator-fixed";
		regulator-name = "VCC_3V3";
		regulator-min-microvolt = <0x325aa0>;
		regulator-max-microvolt = <0x325aa0>;
		phandle = <0x2d>;
	};

	emmc-pwrseq {
		compatible = "mmc-pwrseq-emmc";
		reset-gpios = <0x28 0x23 0x01>;
		phandle = <0x2c>;
	};
```

The 3.3V regulator which has a set voltage and below it is mmc-pwrseq which is also mapped in our EMMC-Device. 
We turn the modified dts into a dtb. Load it up on the USB and give it a go. 
So here is now the storage pinned to a 3.3V and 25Mhz frequency on a 8bit-bus: 

```
aml-s9xx-box:~:# dmesg | grep -I "mmc"
[    2.221838] meson-gx-mmc d0074000.mmc: allocated mmc-pwrseq
[    2.221861] meson-gx-mmc d0070000.mmc: allocated mmc-pwrseq
[    2.272390] meson-gx-mmc d0070000.mmc: card claims to support voltages below defined range
[    2.299671] mmc2: new high speed SDIO card at address 0001
[    2.464858] mmc1: new high speed MMC card at address 0001
[    2.466329] mmcblk1: mmc1:0001 HAG2e\x05 14.7 GiB
[    2.472143]  mmcblk1: p1 p2
[    2.473712] mmcblk1boot0: mmc1:0001 HAG2e\x05 4.00 MiB
[    2.481654] mmcblk1boot1: mmc1:0001 HAG2e\x05 4.00 MiB
[    2.485971] mmcblk1rpmb: mmc1:0001 HAG2e\x05 4.00 MiB, chardev (240:0)
```


great, the OS finally has onboard storage! But how fast is it you may ask: 

```aml-s9xx-box:~:# sudo hdparm -tT /dev/mmcblk1

/dev/mmcblk1:
 Timing cached reads:   1524 MB in  2.00 seconds = 761.77 MB/sec
 Timing buffered disk reads:  68 MB in  3.05 seconds =  22.28 MB/sec (this is the value we care for)
```

Not great but usable for a start. 

![ARMBIAN](https://github.com/MadHarlekin/X96-Mini-Armbian/blob/main/Armbian.png)


To be continued as I still need to dig further into the VPU/GPU/HDMI even if it is only a "nice to have". 

## 3. HDMI and embedded systems
