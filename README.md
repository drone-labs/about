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


## Build Ardupilot
As stated above the firmware is shipped with a complete custom version of ardupilot suite.
Building a custom is not so difficult. In order to avoid any libc conflict, I use the fresh
buildroot toolchain built in previous step. Programs will be linked against uClibc.
Building a static version of ardupilot will also work.

1) Get the sources

		cd
		git clone https://github.com/ArduPilot/ardupilot
		cd ardupilot
    
2)  Install the prerequisites

		The script Tools/environment_install/install-prereqs-ubuntu.sh is a good
		start to see which packages are missing on my system...
    
3) Updates the repository

		git fetch --prune
    
4) See all available branches.

		git branch -a
		...

5) Select one of the ArduCopter branches.

		git checkout Copter-3.6
		or (not tested)
		git checkout master

6) Update repository

		git submodule update --init --recursive

7) Configuration

Create a setenv.sh file, edit it to add these three lines (Ajust it to your needs):

 > AP_DIR=~/Ardupilot-Blue/ardupilot/Tools/autotest
 > GCC_DIR=~/Ardupilot-Blue/buildroot/output/host/bin
 > export PATH=$GCC_DIR:$AP_DIR:$PATH
 
  Then, source the file

		$ . ./setenv.sh
 
8) Configure the Ardupilot build engine (waf) to build programs for the BBBlue and use our toolchain
		$ ./waf configure --board=blue --toolchain=arm-linux

9) Aplly the patch
 My first build attempt failed due to 3 warnings treated as errors.
 I created a small patch to address these issues:

		buildroot/board/bbblue/patches/0001-ardupilot-buildroot.patch

 Copy it to ardupilot root and apply it
 
		cp ~/Ardupilot-Blue/buildroot/board/bbblue/patches/0001-ardupilot-buildroot.patch ./
		patch -p1 --verbose -b < 0001-ardupilot-buildroot.patch
		rm 0001-ardupilot-buildroot.patch

10) Build the programs

		$ ./waf
 
When build is finished, we can find progams in build/blue/bin/ directory

 > BUILD SUMMARY
 > Build directory: /home/bruno/ardupilot/build/blue
 > Target               Text     Data  BSS    Total  
 > --------------------------------------------------
 > bin/ardurover        1602527  1640  45092  1649259
 > bin/antennatracker   1308338  1612  41260  1351210
 > bin/arducopter       1804787  1652  48372  1854811
 > bin/arducopter-heli  1769187  1652  48060  1818899
 > bin/arduplane        1809853  1640  47884  1859377
 > bin/ardusub          1562833  1664  44036  1608533






