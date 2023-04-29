# Flashing Hat EEPROM

## Connect EEPROM

The Raspberry Pi should be powered off before making any connections to the GPIO pins.

The hat is wired to connect the EEPROM to the Raspberry Pi's ID pins with the required pull up resistors.

Connect the hat to the Pi and switch the Raspberry Pi on.

## Install required tools

We will need to install some tools that are not included in the base Raspberry Pi OS image, at the command prompt:

```bash
pi@raspberrypi:~ $ sudo apt update
pi@raspberrypi:~ $ sudo apt install device-tree-compiler git i2c-tools -y
```

## Test EEPROM is Connected

Run the following command as explained in the [EEPROM Utils Docs](https://github.com/raspberrypi/hats/tree/master/eepromutils).

```bash
pi@raspberrypi:~ $ sudo dtoverlay i2c-gpio i2c_gpio_sda=0 i2c_gpio_scl=1 bus=9
pi@raspberrypi:~ $ i2cdetect -y 9
```

You should then be able to see the EEPROM Chip at address 50:

```bash
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:                         -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: 50 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --
```

## Clone Hats Git Repository

Next we need to get the code from the Raspberry Pi Hats Repository.

```bash
pi@raspberrypi:~ $ git clone https://github.com/raspberrypi/hats.git
```

Once we have cloned the repository we need to compile the tools to make the EEPROM image.

```bash
pi@raspberrypi:~ $ cd hats/eepromutils/
pi@raspberrypi:~/hats/eepromutils $ make -j4
```

## Get the required Files

We need 3 files to configure the EEPROM flash file:

- eeprom_settings.txt - [the Klipper Fan Hat EEPROM settings file](https://github.com/mikepthomas/Klipper-Fan-Hat/blob/main/EEPROM/eeprom_settings.txt)
- klipper-fan-hat.dts - [the Klipper Fan Hat device tree source file](https://github.com/mikepthomas/Klipper-Fan-Hat/blob/main/EEPROM/klipper-fan-hat.dts)
- klipper-fan-hat.cfg - [the Klipper config source file](https://github.com/mikepthomas/Klipper-Fan-Hat/blob/main/Software/klipper-fan-hat.cfg)

Save these into the `~/hats/eepromutils` directory

To allow the hat to automatically enable I2C and SPI compile the binary and set the correct permissions to the output file:

```bash
pi@raspberrypi:~/hats/eepromutils $ sudo dtc -@ -I dts -O dtb -o klipper-fan-hat.dtb klipper-fan-hat.dts
pi@raspberrypi:~/hats/eepromutils $ sudo chown pi:pi klipper-fan-hat.dtb
```

## Zero EEPROM

The Recommended EEPROM in the [Design Guide](https://github.com/raspberrypi/hats/blob/master/designguide.md) at the time of writing is CAT24C32 which is 32kbit (4kbyte).

As we don't know the state of the EEPROM it is best to clear it by setting it all to zeros.
If your EEPROM is a different size you will need to set `count` to the value of kbyte of your chip.

Make a blank image using dd...

```bash
pi@raspberrypi:~/hats/eepromutils $ dd if=/dev/zero ibs=1k count=4 of=blank.eep
```

...and flash it to the chip.

```bash
pi@raspberrypi:~/hats/eepromutils $ sudo ./eepflash.sh -w -f=blank.eep -t=24c32
```

## Flash EEPROM

Now that we have a blank EEPROM chip, We can can configure the hat EEPROM image and embed the device tree binary & klipper config into the flash file.

```bash
pi@raspberrypi:~/hats/eepromutils $ ./eepmake eeprom_settings.txt klipper-fan-hat.eep klipper-fan-hat.dtb -c klipper-fan-hat.cfg
```

...then flash it to the EEPROM chip.

```bash
pi@raspberrypi:~/hats/eepromutils $ sudo ./eepflash.sh -w -f=klipper-fan-hat.eep -t=24c32
```

## Test EEPROM

The Hat EEPROM is read on system boot so we will need to reboot the Pi before we can test it:

```bash
pi@raspberrypi:~/hats/eepromutils $ sudo reboot
```

Log back in and you will be able to see the devices `/dev/i2c-1`, and `/dev/spidev0.0` and `/dev/spidev0.1` only when the hat EEPROM is connected.

```bash
pi@raspberrypi:~ $ ls /dev
autofs         dma_heap   i2c-1    loop4         mmcblk0    ram1   ram5     spidev0.0  tty12  tty21  tty30  tty4   tty49  tty58  ttyAMA0    vcs1   vcsa4     vcsu6    video20
block          dri        i2c-2    loop5         mmcblk0p1  ram10  ram6     spidev0.1  tty13  tty22  tty31  tty40  tty5   tty59  ttyprintk  vcs2   vcsa5     vhci     video21
btrfs-control  fd         initctl  loop6         mmcblk0p2  ram11  ram7     stderr     tty14  tty23  tty32  tty41  tty50  tty6   uhid       vcs3   vcsa6     video10  video22
bus            full       input    loop7         mqueue     ram12  ram8     stdin      tty15  tty24  tty33  tty42  tty51  tty60  uinput     vcs4   vcsm-cma  video11  video23
cachefiles     fuse       kmsg     loop-control  net        ram13  ram9     stdout     tty16  tty25  tty34  tty43  tty52  tty61  urandom    vcs5   vcsu      video12  video31
cec0           gpiochip0  log      mapper        null       ram14  random   tty        tty17  tty26  tty35  tty44  tty53  tty62  v4l        vcs6   vcsu1     video13  watchdog
char           gpiochip1  loop0    media0        ppp        ram15  rfkill   tty0       tty18  tty27  tty36  tty45  tty54  tty63  vchiq      vcsa   vcsu2     video14  watchdog0
console        gpiochip2  loop1    media1        ptmx       ram2   serial1  tty1       tty19  tty28  tty37  tty46  tty55  tty7   vcio       vcsa1  vcsu3     video15  zero
cuse           gpiomem    loop2    media2        pts        ram3   shm      tty10      tty2   tty29  tty38  tty47  tty56  tty8   vc-mem     vcsa2  vcsu4     video16
disk           hwrng      loop3    mem           ram0       ram4   snd      tty11      tty20  tty3   tty39  tty48  tty57  tty9   vcs        vcsa3  vcsu5     video18
```

We can then read the data using the device tree...

```bash
pi@raspberrypi:~ $ cd /proc/device-tree/hat/
pi@raspberrypi:/proc/device-tree/hat $ ls
custom_0  custom_1  name  product  product_id  product_ver  uuid  vendor
pi@raspberrypi:/proc/device-tree/hat $ more name
hat
pi@raspberrypi:/proc/device-tree/hat $ more vendor
Voron Design
pi@raspberrypi:/proc/device-tree/hat $ more product
Klipper Fan Hat
pi@raspberrypi:/proc/device-tree/hat $ more product_id
0x4b46
pi@raspberrypi:/proc/device-tree/hat $ more product_ver
0x0002
pi@raspberrypi:/proc/device-tree/hat $ more uuid
fef562f0-9e28-4453-88c2-c073303e6ab2
```

...and also copy the config out of the device tree into the klipper config directory.

```bash
cat /proc/device-tree/hat/custom_1 > ~/printer_data/config/klipper-fan-hat.cfg
```
