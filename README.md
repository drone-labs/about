# Ardupilot Blue
Building a lightweight buildroot based firmware for the BeagleBone Blue.


## Credits
Another ardupilot setup for the BeagleBone Blue?
Yes. Mirko Denecke (https://github.com/mirkix/ardupilotblue) has ported
ardupilot core code for the board and imfatant maintains an excellent guide 
(https://github.com/imfatant/test) to make things work together.
So, why? The existing solution relies on a Debian distribution
(stretch-console) which provides many, many features that i do not aim to use.
Instead, i want to have a small memory footprint system while staying
reliable, configurable, extendable and reproducible.
For that, i use my favorite build system: Buildroot (https://buildroot.org)

## Overview
In my buildroot fork, i added basic support for the board:
 
- The buildroot default configuration file : **configs/bbblue_defconfig**
- Some specific board settings directory   : **board/bbblue/**

For now, u-boot and linux consoles are reachable on ttyS0 (115200, 8N1) and
µUSB, via ssh (ssh root@192.168.7.2, default password is root). The Ethernet
connection relies on Linux USB gadget feature, ssh server is Dropbear. The
wireless connections are not active. the firmware is shipped with a complete
custom version of ardupilot suite.
Once booted, an arduplane instance is automatically launched in background,
and can be managed the standard way :

		/etc/init.d/S60arduplane.sh start|stop|restart

  Current options for the program are
  
		-l /var/APM/logs -A /dev/ttyS1 -B /dev/ttyS2 -C /dev/ttyS5

## Build the firmware

Assuming all prerequisites are in place (and I have not made any mistake...) this is straightforward

1) Get the sources

		git clone https://github.com/drone-labs/buildroot

2) Select the working branch

		git checkout 2019.02.x

3) Update repository

		git submodule update --init --recursive

4) Configure buildroot

		make bbblue_defconfig

5) Review the configuration

		make menuconfig

5) Build the firmware

		make

6) Take a break...

7) Copy the firmware to a SD card

		sudo dd if=output/images/sdcard.img of=/dev/XXX









