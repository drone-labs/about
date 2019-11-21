# Ardupilot Blue
### Building a lightweight buildroot based firmware for the BeagleBone Blue


## Credits
Another ardupilot setup for the BeagleBone Blue?
Yes. Mirko Denecke (https://github.com/mirkix/ardupilotblue) has ported
ardupilot core code for the board and imfatant maintains an excellent guide 
(https://github.com/imfatant/test) to make things work together.
So, why? The proposed solution relies on a Debian distribution
(stretch-console) which provides many, many features that I do not aim to use.
Instead, I want to have a small memory footprint system while staying
reliable, configurable, extendable and reproducible.
For that, I use my favorite build system: Buildroot (https://buildroot.org)

## Overview
In my buildroot fork, I added basic support for the board:
 
- The buildroot default configuration file : **configs/bbblue_defconfig**
- Some specific board settings directory   : **board/bbblue/**

For now, u-boot and linux consoles are reachable on ttyS0 (115200, 8N1) and
µUSB, via ssh (ssh root@192.168.7.2, default password is root). The Ethernet
connection relies on Linux USB gadget feature, ssh server is Dropbear. The
wireless connections are not active. the firmware is shipped with a complete
ardupilot binaries suite. The filesystem size is 256MB.

Main software versions

- linux-4.19-rt (from https://github.com/beagleboard/linux.git)
- busybox-1.29.3
- uboot-2019.10

I reworked the stock linux device tree source file (am335x-boneblue.dts) to
replace all the Beaglebone black P8 and P9 headers pins references by the
matching Beaglebone blue ones. The same way, I changed the included common
bone pins configuration (am335x-bone-common-universal-pins.dtsi)
by a new am335x-boneblue-pins.dtsi file. These changes are made upon linux
kernel build through a dedicated patch :  
>board/bbblue/patches/linux/0002-Clean-am335x-boneblue-dts-Enable-uio-pruss-bbblue.patch

Once booted, an arduplane instance is automatically created in background,
and can be managed the standard way :

	/etc/init.d/S60arduplane.sh start|stop|restart

  Default options for the program are

	-l /var/APM/logs -A /dev/ttyS1 -B /dev/ttyS2 -C /dev/ttyS5


Notes:
My PC : i5-4440@3.1GHz, 8GB RAM, Linux Mint 17.3
Firmware build from scratch takes about 30 minutes

In the rest of the document, ARDUPILOT_BLUE refers to an existing
common working directory
e.g : $ mkdir ~/ardupilot_blue
      $ export ARDUPILOT_BLUE=~/ardupilot_blue

Check free disk space; buildroot is quite gluttonous. 20GB is comfortable
Used disk usage after build
    buildroot : 7 666MB
  br_download : 3 553MB
    ardupilot :   677MB

## Build the firmware

Assuming all prerequisites are in place (and I have not made any mistake...) this is straightforward

1) Get the sources

		$ mkdir Ardupilot-blue
		$ cd Ardupilot-blue
		$ mkdir br_download
		$ git clone https://github.com/drone-labs/buildroot
		$ cd buildroot

	> br_download is the directory where buildroot will save the files it will download
	> during build. Using an out of tree directory make them easily available for others builds 

2) Select the working branch

		$ git checkout 2019.02.x

3) Update repository

		$ git fetch --prune

4) Configure buildroot

		$ make bbblue_defconfig

5) Review the configuration

		$ make menuconfig

5) Build the firmware

		$ make

6) Take a break...

7) Copy the firmware to a SD card

		$ sudo dd if=output/images/sdcard.img of=/dev/XXX

8) Fire it up



## Build Ardupilot
As stated above the firmware is shipped with a complete custom version of ardupilot suite.
Thanks to buildroot tools, cross-compile ardupilot is not so difficult. In order to avoid
any libc conflict, I use the fresh buildroot toolchain built in previous step. Programs
will be linked against uClibc. Building a static version of ardupilot will also work.
I forked ardupilot (https://github.com/ArduPilot/ardupilot) and created a new branch named
drone-labs.  
My first build attempt failed due to three gcc warnings treated as errors. These were caused
by gcc complaining about source and destination strings length mismatch upon a snprintf()
call. Giving a little more room to the destination strings did the job.
After some investigations about the annoying "RCOutputAioPRU.cpp:SIGBUS error generated"
error, I guessed this was caused by some device tree malformation, bypassed by three gpio
lines export done into the code (gpio lines 5, 65 and 105 in AP_HAL_Linux/GPIO_BBB.cpp,
GPIO_BBB::init() function). Using my provided device tree, this code section is no more
needed and has been commented out.

1) Get the sources

		$ cd Ardupilot-blue
		$ https://github.com/drone-labs/ardupilot
		$ cd ardupilot
    
2)  Install the prerequisites

	>The script Tools/environment_install/install-prereqs-ubuntu.sh is
	a good start point to see which packages are missing on the system...

3) Update the repository

		$ git fetch --prune
    
4) See all available branches

		$ git branch -a
			...

5) Switch to the drone-labs branch

		$ git checkout drone-labs

6) Update repository

		$ git submodule update --init --recursive

7) Configuration

	First, update the PATH environment variable (must be ajdusted to suit configuration) :

		$ AP_DIR=Ardupilot-Blue/ardupilot/Tools/autotest  
		$ GCC_DIR=Ardupilot-Blue/buildroot/output/host/bin  
		$ export PATH=$GCC_DIR:$AP_DIR:$PATH  
 
8) Configure the Ardupilot build engine (waf) to build programs for the BBBlue and use our toolchain

		$ ./waf configure --board=blue --toolchain=arm-linux

9) Apply the patch

	My first build attempt failed due to 3 warnings treated as errors.  
	I created a small patch to address these issues:

	> buildroot/board/bbblue/patches/ardupilot/0001-ardupilot-buildroot.patch

	Copy it to ardupilot root and apply it
 
		$ cp ~/Ardupilot-Blue/buildroot/board/bbblue/ardupilot/patches/0001-ardupilot-buildroot.patch ./
		$ patch -p1 --verbose -b < 0001-ardupilot-buildroot.patch
		$ rm 0001-ardupilot-buildroot.patch

10) Build the programs

		$ ./waf
 
	When build is finished, we can find progams in build/blue/bin/ directory

		BUILD SUMMARY  
		Build directory: /home/bruno/ardupilot/build/blue  
		Target               text     Data  BSS    Total  
		==================================================  
		bin/ardurover        1602527  1640  45092  1649259  
		bin/antennatracker   1308338  1612  41260  1351210  
		bin/arducopter       1804787  1652  48372  1854811  
		bin/arducopter-heli  1769187  1652  48060  1818899  
		bin/arduplane        1809853  1640  47884  1859377  
		bin/ardusub          1562833  1664  44036  1608533  

11) Update the target

	Assuming the board is running the previously built firmware,
	copy the programs to the right target filesystem location:

		$ scp ./build/blue/bin/a* root@192.168.7.2:/usr/bin/ardupilot

	And restart the application

		$ ssh root@192.168.7.2  (default password = root)
		# /etc/init.d/S60arduplane restart

	While logged in, take a first look at the used board resources

		# top
		Mem: 33052K used, 464664K free, 60K shrd, 528K buff, 5124K cached
		CPU:   2% usr  45% sys   0% nic  37% idle   0% io   0% irq  14% sirq
		Load average: 1.88 1.97 1.92 1/107 289
		PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
		278     1 root     S     8428   2%  24% /usr/bin/ardupilot/arduplane -l /var/APM/logs -A /dev/ttyS1 -B /dev/ttyS2 -C /dev/ttyS5
		...
	
		# df
		Filesystem           1K-blocks      Used Available Use% Mounted on
		/dev/root               245679     83523    144953  37% /
		devtmpfs                223768         0    223768   0% /dev
		tmpfs                   248856         0    248856   0% /dev/shm
		tmpfs                   248856        36    248820   0% /tmp
		tmpfs                   248856        24    248832   0% /run

	
I2C onboard sensors are OK?

	# /etc/init.d/S60arduplane stop
	
	# i2cdetect -r -y 2
	     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
	00:          -- -- -- -- -- -- -- -- -- 0c -- -- -- 
	10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
	60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- -- 
	70: -- -- -- -- -- -- 76 --
	
	/etc/init.d/S60arduplane start

- 0c = AKM AK8963 compass (inside the MPU-9250)  
- 68 = InvenSense MPU-9250 IMU  
- 76 = Bosch BMP280 barometer.

Finally close the ssh session

	# exit


## ToDo
Run from onboard flash  
List Software versions  
Enable Servo Power Rail  
Enable Pru input on E4 Header (PRU_E_B input)  
add QGroundControl tips  
Configure wifi  
plug in the Sbus RC receiver signal to DSM2 Header (ttyS4)  
plug in the GPS (ttyS2)  





































