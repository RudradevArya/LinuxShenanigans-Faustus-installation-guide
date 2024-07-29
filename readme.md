# Linux Shenanigans :: Faustus and TUF Controller UI installation on Asus TUF A15 FA506II

## My System info for reference
OS: Linux Mint 21.3 x86\_64
Host: ASUS TUF Gaming A15 FA506II\_FA
Kernel: 5.15.0-113-generic
DE: Cinnamon 6.0.4

BIOS Information
Vendor: American Megatrends Inc.
Version: FA506II.316
Release Date: 03/12/2021

Base Board Information
Manufacturer: ASUSTeK COMPUTER INC.
Product Name: FA506II
Version: 1.0

## Faustus Installation

* **Notes:**

1. Fan mode control is not supported.
2. RGB hot keys (Fn-Left, Fn-Right) are not functional.

See "Contributing" section for other versions.

To check your exact model run

``` default
sudo dmidecode | less
```

and scroll down to check BIOS Information / Version (2nd column) and Base Board Information / Product name (1st column).

## Features

* Additional Fn-X Hotkeys
* Keyboard backlight intensity
* Color and mode control for RGB keyboard backlight
* Fan boost mode switching

## UI

* Works fine, fulfils my needs
[icodelifee/TUF-Control](https://github.com/icodelifee/TUF-Control)
A Keyboard Lighting And Fan Mode Controller GUI App - awesome Electron-based frontend for this driver (WIP).
* Another option which is actively maintained but didnt work for me
[legacyO7/Aurora](https://github.com/legacyO7/Aurora)
* Opens but did not work for me
[CalcProgrammer1/OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)
Open source RGB lighting control that doesn't depend on manufacturer software - supports multiple RGB controllers, including this driver.
* Vague description no proper installation guide
[cromer/tuf-manager](https://git.cromer.cl/cromer/tuf-manager)
The software includes 2 different user interfaces, CLI and GUI. It is written in Vala and uses GTK3 for the GUI.

## Installation

How to: first disable old drivers, then proceed using make to test that it works at all, then install via DKMS permanently and enable on boot.

### Disable original modules

Create file /etc/modprobe.d/faustus.conf with the following contents and reboot the system.

```
blacklist asus_wmi
blacklist asus_nb_wmi
```

You could also try unloading the modules instead of reboot before proceeding by issuing:

```
sudo rmmod asus_nb_wmi
sudo rmmod asus_wmi
```

Some reports may suggest that you need to reboot after blacklisting, as the modules fail cleaning up on errors (do if you see AE\_ALREADY\_ACQUIRED in dmesg).

### Install build dependencies and DKMS

```
$ sudo apt-get install dkms
```

```
$ git clone https://github.com/hackbnw/faustus
cd faustus
```

### Using make

Compile and load the driver temporarily

```
$ make
$ sudo modprobe sparse-keymap
$ sudo modprobe wmi
$ sudo modprobe video
$ sudo insmod src/faustus.ko
```

and check `dmesg | tail` to verify the driver is loaded and no errors are present.

```
[ 8295.475755] faustus: DMI checK: FX505GM
[ 8295.476475] faustus: Initialization: 0x1
[ 8295.477057] faustus: BIOS WMI version: 8.1
[ 8295.477680] faustus: SFUN value: 0x4a0061
[ 8295.477687] faustus faustus: Use DSTS
[ 8295.477691] faustus faustus: Enable event queue
[ 8295.490603] input: Asus WMI hotkeys as /devices/platform/faustus/input/input34
[ 8295.492695] faustus: Number of fans: 1
```

If you see:

```
ERROR: could not insert module src/faustus.ko: No such device
```

it most likely means that your system is not in the "Systems" list above and not in the DMI table. This is not a bug and does not necessarily mean that the module does not work, but as there is no evidence that it does work on your system, it will fail fast with the above error message (see "Contributing" section below for bypassing the check if you feel adventurous).

Check that everything works, the system is stable. Also try unloading the driver with
markdown
```
$ sudo rmmod faustus
```

and inserting it back using.

```
$ sudo insmod src/faustus.ko let_it_burn=1
```

(use `let_it_burn=1` only if you got `ERROR: could not insert module src/faustus.ko: No such device` before)

this will show is the faustus modules are loaded or not

```
$ lsmod | grep faustus
```

something like this

```
faustus                45056  0
sparse_keymap          16384  2 faustus,asus_wmi
wmi                    32768  4 faustus,asus_wmi,wmi_bmof,mfd_aaeon
video                  65536  2 faustus,asus_wmi
```

### Using DKMS

```
$ make dkms
```

The source code will probably be installed in `/usr/src/faustus-<version>/` and the module itself will be compiled and installed in the current kernel module directory `/lib/modules/...`. It should also be automatically rebuilt when the kernel is upgraded.

Next, try to load the module

```
$ sudo modprobe faustus
```

To uninstall the DKMS module execute

```
$ sudo make dkmsclean
```

or

```
$ sudo dkms remove faustus/<version> --all
```

NOTE: The DKMS install does work with secure boot on Ubuntu 18.04.

### Load on boot

On Ubuntu execute

```
$ sudo make onboot
```

or add it to your config files in the other way. Revert with

```
$ sudo make noboot
```

This is OS dependent, check the internet on how to do it right.

## Usage

### Keyboard backlight intensity

Is exposed via ledclass device `/sys/class/leds/asus::kbd_backlight` takes values 0 to 3. The driver changes brightness by itself when hotkeys are pressed.

### RGB backlight

TLDR: Run the `./set_rgb.sh` script as root.

NOTE: The interface will most definitely switch to LED subsystem when submitted to mainline. This here is sort of hack.

Driver exposes sysfs attributes in `/sys/devices/platform/faustus/kbbl/`. You have to write all the parameters and then write 1 to `kbbl_set` to write them permanently or 2 to write them temporarily (the settings will reset on restart or hibernation).

The list of settings is:

* `kbbl_red` \- red component in hex \[00 \- ff\]
* `kbbl_green` \- green component in hex \[00 \- ff\]
* `kbbl_blue` \- blue component in hex \[00 \- ff\]
* `kbbl_mode` \- mode:
    * 0 - static color
    * 1 - breathing
    * 2 - color cycle (the color component parameters have no effect)
    * 3 - strobe (epileptic mode, speed parameter has no effect)
* `kbbl_speed` \- speed for modes 1 and 2:
    * 0 - slow
    * 1 - medium
    * 2 - fast
* `kbbl_flags` \- enable flags \(must be ORed to get the value\)\, use 2a or ff to set all
    * 02 - on boot (before module load)
    * 08 - awake
    * 20 - sleep
    * 80? - should be logically shutdown, but I have genuinely no idea what it does

### Fan mode

Is controlled by default by the driver itself when `Fn-F5` is pressed switching three modes:

* 0 - normal
* 1 - overboost
* 2 - silent

There are two mode files available depending on the laptop model:

* `/sys/devices/platform/faustus/fan_boost_mode`
* `/sys/devices/platform/faustus/throttle_thermal_policy`.

In case if the `throttle_thermal_policy` is present, it has always all 3 modes available, whereas individual modes of `fan_boost_mode` may or may not be available. The mode will not be preserved on reboot or hibernation.

## Contributing

If you own a machine of this series from the table above it would be much appreciated if you test the driver and write your feedback (successful and otherwise) in an issue on GitHub.

If you machine of this series is not in the list, likely it is supported too. You can check the DSDT yourself and try loading the driver without DMI verification by passing `let_it_burn=1`:

```
sudo insmod ./src/faustus.ko let_it_burn=1
```

and send feedback if it works.

### Information to include in feedback

Always OS / kernel version and

```
$ sudo dmidecode | grep "BIOS Inf\|Board Inf" -A 3 
BIOS Information
	Vendor: American Megatrends Inc.
	Version: FX505GM.301
	Release Date: 09/21/2018
--
Base Board Information
	Manufacturer: ASUSTeK COMPUTER INC.
	Product Name: FX505GM
	Version: 1.0
```

and additionally for a new model

```
$ sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.aml
```

# TUF Controller Installation

## How to compile .deb package

1. git clone https://github.com/icodelifee/TUF-Control.git
2. cd TUF-Control
3. npm install
4. npm install electron -g
5. npm install electron-packager -g
6. npm install electron-installer-debian -g
7. npm run-script build
8. electron-installer-debian --src dist/tufcontrol-electron/ --dest dist/installers/ --arch amd64
9.  check if in `~/TUF-Control/master/` has `tufcontrol-electron-linux-x64` using `ls dist/`
10. Package will be present in dist/installers
11. go to the .deb package and run it
12. enjoy or if on non .deb system use official guide fir .rpm