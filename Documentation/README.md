- [Components](#components)
  - [Indicator LEDs](#indicator-leds)
  - [Serial Header](#serial-header)
  - [Power In](#power-in)
  - [Fan Mosfets](#fan)
  - [I2C Header](#i2c-header)
  - [1-wire Header](#i2c-header)
  - [GPIO Headers](#gpio-headers)
  - [SPI Header](#spi-header)
- [Setup](#setup)

# Components

## Indicator LEDs

![Indicator LEDs](../Images/indicator-leds.jpg)

![Indicator LEDs Schematic](../Images/indicator-leds-schematic.jpg)

## Serial Header

![Serial Header](../Images/serial-header.jpg)

![Serial Header Schematic](../Images/serial-header-schematic.jpg)

## Power In

![Power In](../Images/power-in.jpg)

![Power In Schematic](../Images/power-in-schematic.jpg)

## Fan Mosfets

![Fan Mosfets](../Images/fan-mosfets.jpg)

![Fan Mosfets Schematic](../Images/fan-mosfets-schematic.jpg)

## I2C Header

![I2C Header](../Images/i2c-header.jpg)

![I2C Header Schematic](../Images/i2c-header-schematic.jpg)

## 1-wire Header

![1-wire Header](../Images/1-wire-header.jpg)

![1-wire Header Schematic](../Images/1-wire-header-schematic.jpg)

## GPIO Header

![GPIO Header](../Images/gpio-header.jpg)

![GPIO Header Schematic](../Images/gpio-header-schematic.jpg)

## SPI Header

![SPI Header](../Images/spi-header.jpg)

![SPI Header Schematic](../Images/spi-header-schematic.jpg)

# Setup

You should just be able to go through the [Klipper RPi micro-controller setup guide](https://www.klipper3d.org/RPi_microcontroller.html), and that page will be the most up-to-date with the latest Klipper version.

## Install the RC Script

If you want to use the host as a secondary MCU the klipper_mcu process must run before the klippy process.

After installing Klipper, install the script. run:

```bash
cd ~/klipper/
sudo cp ./scripts/klipper-mcu.service /etc/systemd/system/
sudo systemctl enable klipper-mcu.service
```

## Building the Micro-Controller Code

To compile the Klipper micro-controller code, start by configuring it for the `Linux process`:

```bash
cd ~/klipper/
make menuconfig
```

In the menu, set `Micro-controller Architecture` to `Linux process`, then save and exit.

![Klipper Config](../Images/klipper-config.jpg)

To build and install the new micro-controller code, run:

```bash
sudo service klipper stop
make flash
sudo service klipper start
```

If klippy.log reports a `Permission denied` error when attempting to connect to `/tmp/klipper_host_mcu` then you need to add your user to the tty group. The following command will add the `pi` user to the `tty` group:

```bash
sudo usermod -a -G tty pi
```

## Remaining configuration

You can copy the config from the EEPROM chip of the hat into the Klipper config directory:

```bash
cat /proc/device-tree/hat/custom_1 > ~/printer_data/config/klipper-fan-hat.cfg
```

## Optional: Enabling SPI

SPI should be enabled automatically by the hat EEPROM, If you have trouble, you can enable it manually by running `sudo raspi-config` and enabling SPI under the `Interfacing options` menu.

![Enable SPI](../Images/enable-spi.jpg)

## Optional: Enabling I2C

I2C should be enabled automatically by the hat EEPROM, If you have trouble, you can enable it manually by running `sudo raspi-config` and enabling I2C under the `Interfacing options` menu.

![Enable I2C](../Images/enable-i2c.jpg)

If planning to use I2C for the MPU accelerometer, it is also required to set the baud rate to 400000 by: adding/uncommenting `dtparam=i2c_arm=on,i2c_arm_baudrate=400000` in `/boot/config.txt` (or `/boot/firmware/config.txt` in some distros).
This should also be automatically be enabled by the hat EEPROM however you can do it manually if you have any problems.

## Optional: Enabling 1-wire

If you require, you can enable the 1-wire interface by running `sudo raspi-config` and enabling 1-wire under the `Interfacing options` menu.

![Enable 1-wire](../Images/enable-1-wire.jpg)

you can then find any conneted sensors serial numbers with: `ls /sys/bus/w1/devices/`

## Optional: Hardware PWM

Raspberry Pi's have two PWM channels (PWM0 and PWM1) which are exposed on the header or if not, can be routed to existing gpio pins. The Linux mcu daemon uses the pwmchip sysfs interface to control hardware pwm devices on Linux hosts. The pwm sysfs interface is not exposed by default on a Raspberry and can be activated by adding a line to `/boot/config.txt`:

```bash
# Enable pwmchip sysfs interface
dtoverlay=pwm,pin=12,func=4
```

This example enables only PWM0 and routes it to gpio12. If both PWM channels need to be enabled you can use `pwm-2chan`:

```bash
# Enable pwmchip sysfs interface
dtoverlay=pwm-2chan,pin=12,pin2=13,func=4,func2=4
```

The overlay does not expose the pwm line on sysfs on boot and needs to be exported by echo'ing the number of the pwm channel to `/sys/class/pwm/pwmchip0/export`:

```bash
echo 0 > /sys/class/pwm/pwmchip0/export
```

If you have enabled `pwm-2chan` you can enable PWM1 too with:

```bash
echo 1 > /sys/class/pwm/pwmchip0/export
```

This will create device `/sys/class/pwm/pwmchip0/pwm0` and optionally `/sys/class/pwm/pwmchip0/pwm1` in the filesystem. The easiest way to do this is by adding them to `/etc/rc.local` before the `exit 0` line.

With the sysfs in place, you can now use either the pwm channel(s) by adding the following piece of configuration to your `klipper-fan-hat.cfg`:

```bash
[fan_generic fan1]
pin: rpi:pwmchip0/pwm0
hardware_pwm: true
cycle_time: 0.000001
```

This will add hardware pwm control to gpio12 on the Pi (because the overlay was configured to route pwm0 to pin=12).

PWM0 can be routed to gpio12 and gpio18, PWM1 can be routed to gpio13 and gpio19:

| PWM | gpio PIN | Func |
| --- | -------- | ---- |
| 0   | 12       | 4    |
| 0   | 18       | 2    |
| 1   | 13       | 4    |
| 1   | 19       | 2    |
