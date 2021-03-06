# Ardupilot Blue
### Building a lightweight buildroot based firmware for the BeagleBone Blue


## Credits
Another ardupilot setup for the BeagleBone Blue?
Yes. Mirko Denecke (https://github.com/mirkix/ardupilotblue) has ported
ardupilot core code for the board and @imfatant maintains an excellent guide 
(https://github.com/imfatant/test) to make things work together.
So, why? The proposed solution relies on a Debian distribution
(stretch-console) which provides many, many features that I do not aim to use.
Instead, I want to have a simple, small memory footprint system while staying
reliable, configurable, extendable and reproducible.
For that, I used my favorite build system: Buildroot (https://buildroot.org)

## Overview
In my buildroot fork, I added basic support for the board:
 
- The buildroot default configuration file : **`configs/bbblue_defconfig`**
- Dedicated board settings directory       : **`board/bbblue/`**

For now, u-boot and linux console is reachable on ttyS0 (115200, 8N1) and
µUSB, via ssh (ssh root@192.168.7.2, default password is root). The Ethernet
connection relies on Linux USB gadget feature, ssh server is Dropbear. The
wireless connections are not active. the firmware is shipped with a complete
ardupilot binaries suite. The filesystem size is 256MB (less than 100MB used).

Main software versions

- linux-4.19-rt (from https://github.com/beagleboard/linux.git)
- busybox-1.29.3
- uboot-2019.10

I reworked the stock linux device tree source file (**`am335x-boneblue.dts`**) to
replace all the Beaglebone black P8 and P9 headers pins references by the
matching Beaglebone blue ones. The same way, I changed the included common
bone pins configuration (**`am335x-bone-common-universal-pins.dtsi`**) by a new
**`am335x-boneblue-pins.dtsi`** file. The PRU uio interface is also supported.
These changes are made upon linux kernel build through a dedicated patch :  
>**`board/bbblue/patches/linux/0002-Clean-am335x-boneblue-dts-Enable-uio-pruss-bbblue.patch`**

Once booted, an arduplane instance is automatically created in background,
and can be managed the standard way :

**`  /etc/init.d/S60arduplane.sh start|stop|restart`**

Default options for the program (defined in the script) are

**`  -l /var/APM/logs -A /dev/ttyS1 -B /dev/ttyS2 -C /dev/ttyS5`**

In the same way, I wrote a script to control the servos power rail (**`S55servopower`**) and
another to control the **`PRU_E_B`** line configuration (**`default`** or **`pruecapin_pu`**)
in order to use it as a Sbus RC input (**`S56pru_e_b`**). **`S55servopower`** is not active by default
(renamed to **`X55servopower`**)

**Notes:**

+ Host : i5-4440@3.1GHz, 8GB RAM, Linux Mint 17.3  
  (Firmware build from scratch takes about 30 minutes)

+ throughout the rest of the document, **`ARDUPILOT_BLUE`** refers to an existing common root working
directory, eg:

		$ mkdir ~/ardupilot_blue
		$ export ARDUPILOT_BLUE=~/ardupilot_blue

+ Check free disk space; buildroot is quite gluttonous. **20GB** is comfortable

+ Disk usage after build

		buildroot   : 7 666MB
		br_download : 3 553MB  
		ardupilot   :   677MB

+ Configurable pins through /sys/devices/platform/ocp

		Name                Signal      GPIO    Bone      Ball    Configuration
		ocp:DSM2_1_pinmux   UART4_RX    0_30    P9_11     T17     uart
		ocp:E4_4_pinmux     PRU_E_B     1_15    P8_15     U13     pruin_pu
		ocp:GP0_3_pinmux    GPIO1_25    1_25    -----     U16     gpio_pu
		ocp:GP0_4_pinmux    GPIO1_17    1_17    P9_23     V14     gpio_pd
		ocp:GP0_5_pinmux    GPIO3_20    3_20    P9_91     D13     gpio_pd
		ocp:GP0_6_pinmux    GPIO3_17    3_17    P9_28     C12     gpio_pd
		ocp:GP1_3_pinmux    GPIO3_2     3_2     -----     J15     gpio_pu
		ocp:GP1_4_pinmux    GPIO3_1     3_1     -----     H17     gpio_pu
		ocp:GPS_3_pinmux    UART2_RX    0_2     P9_22     A17     uart
		ocp:GPS_4_pinmux    UART2_TX    0_3     P9_21     B17     uart
		ocp:S11_6_pinmux    SPI1_SS1    0_29    -----     H18     spi
		ocp:S12_6_pinmux    SPI1_SS2    0_7     P9_42     C18     spi
		ocp:S1X_3_pinmux    SPI1_MOSI   3_16    P9_30     D12     spi
		ocp:S1X_4_pinmux    SPI1_MISO   3_15    P9_29     B13     spi
		ocp:S1X_5_pinmux    SPI1_SCK    3_14    P9_31     A13     spi_sclk
		ocp:UT1_3_pinmux    UART1_RX    0_14    P9_26     D16     uart
		ocp:UT1_4_pinmux    UART1_TX    0_15    P9_24     D15     uart

## Build the firmware

Assuming all prerequisites are met (and I have not made any mistake...) this is straightforward.
Take a look at buildroot documentation available online; Part 1, chapter 2: System requirements

### 1. Get the sources

	$ cd $ARDUPILOT_BLUE
	$ mkdir br_download
	$ git clone https://github.com/drone-labs/buildroot
	$ cd buildroot

> br_download is the directory where buildroot will save the files it will download
> during build. Using an out of tree directory make them easily available for others builds 

### 2. Select the working branch

	$ git checkout 2019.02.x

### 3. Update repository

	$ git fetch --prune

### 4. Configure buildroot

	$ make bbblue_defconfig

### 5. Review the configuration

	$ make menuconfig

### 6. Build the firmware

	$ make

### 7. Take a break...

### 8. Copy the firmware to a SD card

	$ sudo dd if=output/images/sdcard.img of=/dev/XXX

### 9. Fire it up

### 10. Extras
#### 10.1 Wifi
As stated above, wifi is not active by default (used for debug). In order
to enable it, 2 files need to be mofified :

**/etc/network/interfaces**

```
  # The loopback network interface.
  auto lo
  iface lo inet loopback

  # WiFi onboard device (dynamic IP)
  auto wlan0
  iface wlan0 inet dhcp
  #iface wlan0 inet static
  #address 192.168.1.28
  #netmask 255.255.255.0
  #gateway 192.168.1.254
  #dns-nameservers 8.8.8.8 1.1.1.1
  pre-up modprobe wl18xx
  pre-up modprobe wlcore_sdio
  pre-up wpa_supplicant -i wlan0 -c /etc/wpa_supplicant.conf -B

  # Ethernet/RNDIS gadget (u_ether)
  iface usb0 inet static
  address 192.168.7.2
  netmask 255.255.255.0
  network 192.168.7.0
  gateway 192.168.7.1
  post-up /usr/sbin/udhcpd

```
**/etc/wpa_supplicant.conf**

```
  ctrl_interface=/var/run/wpa_supplicant
  update_config=1
  ap_scan=1
  fast_reauth=0
  network={
    ssid="<Access_Point_SSID>"
    psk="<Password>"
    # or psk=generated_passphrase_with_wpa_passphrase_without_quotation_marks
  }
```

After a reboot, wifi should now be active, **but it is not the case!**
I still don't know why, but in a ssh session, a manual start works
(any help would be appreciated!) :

```
  # /etc/init.d/S60arduplane stop
  Stopping arduplane: OK
  # ifup wlan0
    Successfully initialized wpa_supplicant
    udhcpc: started, v1.29.3
    udhcpc: sending discover
    udhcpc: sending discover
    udhcpc: sending discover
    udhcpc: sending select for 192.168.1.28
    udhcpc: lease of 192.168.1.28 obtained, lease time 43200
    deleting routers
    adding dns 192.168.1.254
  # /etc/init.d/S60arduplane start
```

#### 10.2 Boot from the onboard Flash
Ok, everythings is now working fine; it is time to transfer the software on
the 4GB onboard Flash Memory. In this section, I use a Pololu USB AVR
Programmer V2.1 and a homemade cable to get a terminal console (minicom) :

```
    BeagleBone Blue            Pololu Adatpter
    UT0 : 4pins JST-SH         6pins 0.1"
          Female socket        Female socket
    ===========================================
         GND    1  -------------  1  GND
         3.3V   2  x     
         RX     3  <------------  4  Tx
         TX     4  ------------>  5  Rx
```

##### 10.2.1 Get some basic informations about available resources
Plug the USB to serial adapter to UT0 Header (UART0) then boot the board
from the SD Card. On Host side, open a serial terminal emulator :

```
  $ minicom -D /dev/ttyACM1 -b 115200
```

  **The SD Card :**
```
  # fdisk -l /dev/mmcblk0
    Disk /dev/mmcblk0: 15 GB, 15931539456 bytes, 31116288 sectors
    486192 cylinders, 4 heads, 16 sectors/track
    Units: sectors of 1 * 512 = 512 bytes

    Device       Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
    /dev/mmcblk0p1 *  0,0,2       2,10,9               1      32768      32768 16.0M  c Win95 FAT32 (LBA)
    /dev/mmcblk0p2    2,10,10     67,79,13         32769    1081344    1048576  512M 83 Linux
```

  **The onboard Flash :**
```
  # fdisk -l /dev/mmcblk1
    Disk /dev/mmcblk1: 3648 MB, 3825205248 bytes, 7471104 sectors
    116736 cylinders, 4 heads, 16 sectors/track
    Units: sectors of 1 * 512 = 512 bytes

    Disk /dev/mmcblk1 doesn't contain a valid partition table
    (I destroyed the MBR when testing with debian...)
```

##### 10.2.2 Configure u-boot

```
  $ cd $ARDUPILOT_BLUE/buildroot
  $ make uboot-menuconfig
```

In Environment menu check the Environment type location (FAT) and set
the correct device/partition :
(For now, don't know how to get a more generic solution which would work
whatever the boot device...)

```
  Environment --->
    ...
    [*] Environment is in a FAT filesystem
    ...
    (1:1) Device and partition for where to store the environemt in FAT
    ...
```

Rebuild the firmware and a SD Card.

##### 10.2.3 Clone the SD Card content
Power on the board from the freshly built SD Card, log in and from the
serial terminal emulator, make a binary copy : 

```
    # dd if=/dev/mmcblk0 bs=512 count=1048576 of=/dev/mmcblk1
    # sync
```

##### 10.2.4 Adjust boot settings
Power off the board then remove the SD Card. With the serial terminal emulator
still alive, power on the board and catch the u-boot prompt before he fire up
the OS (by default, a 2 second countdown).\
We now need to tell u-boot where to find the two partitions (boot on VFAT
and rootfs on ext4), then uEnv.txt (on the boot partition) will be adjusted.
When u-boot is properly configured, he loads **uEnv.txt** file and execute the
**uenvcmd** variable contents.

The **original uEnv.txt** (on the SD card) :

```
    bootpart=0:1
    devtype=mmc
    bootdir=
    bootfile=zImage
    bootpartition=mmcblk0p2
    set_mmc1=if test $board_name = A33515BB; then setenv bootpartition mmcblk1p2; fi
    set_bootargs=setenv bootargs console=ttyO0,115200n8 root=/dev/${bootpartition} rw rootfstype=ext4 rootwait
    uenvcmd=run set_mmc1; run set_bootargs;run loadimage;run loadfdt;printenv bootargs;bootz ${loadaddr} - ${fdtaddr}
```

Adjust **bootpart** (u-boot partition), **bootpartition** (linux partition) and the **boot arguments list** :

```
    uboot> setenv bootpart 1:1
    uboot> setenv bootpartition mmcblk1p2
    uboot> setenv bootargs "console=ttyS0,115200n8 root=/dev/mmcblk1p2 rw rootfstype=ext4 rootwait"
```

Make a last check :

```
    uboot> printenv bootpart
           bootpart=1:1
    uboot> printenv devtype
           devtype=mmc
    uboot> printenv bootdir
           ## Error: "bootdir" not defined
    uboot> printenv bootfile
           bootfile=zImage
    uboot> printenv bootpartition
           bootpartition=mmcblk1p2
    uboot> printenv loadimage
           loadimage=load ${devtype} ${bootpart} ${loadaddr} ${bootdir}/${bootfile}
    uboot> printenv loadfdt
           loadfdt=load ${devtype} ${bootpart} ${fdtaddr} ${bootdir}/${fdtfile}
```

Then repeat the boot sequence step by step :

```
    uboot> run loadimage
           8360448 bytes read in 541 ms (14.7 MiB/s)
    uboot> run loadfdt
           88567 bytes read in 8 ms (10.6 MiB/s)
    uboot> bootz ${loadaddr} - ${fdtaddr}
           ## Flattened Device Tree blob at 88000000
           Booting using the fdt blob at 0x88000000
           Loading Device Tree to 8ffe7000, end 8ffff9f6 ... OK
           Starting kernel ...
           [    0.000000] Booting Linux on physical CPU 0x0
           ...
```

Once logged in (root/root), uEnv.txt can be adjusted :

```
    # mkdir boot
    # mount -t vfat /dev/mmcblk1p1 /boot
    # nano /boot/uEnv.txt
      bootpart=1:1
      devtype=mmc
      bootdir=
      bootfile=zImage
      bootpartition=mmcblk1p2
      set_bootargs=setenv bootargs console=ttyS0,115200n8 root=/dev/${bootpartition} rw rootfstype=ext4 rootwait
      uenvcmd=run set_bootargs;run loadimage;run loadfdt;printenv bootargs;bootz ${loadaddr} - ${fdtaddr}
```

A last reboot and the work should be done :

```
    # umount /dev/mmcblk1p1
    # reboot
```
#### 10.3 Monitoring Battery Level
A 2-cell LIPO Battery can be used to power the Board. The Battery balancing plug has
to be plugged into the JST-XH **LIPO** header. **Check this point before buying**
(Connector type and wiring diagram).
Charging logic and protections are handled by the hardware. Monitoring the status Leds
(BATT_LED_1 to BATT_LED_4) has to be done by software. Below is a simplified diagram
of the BeagleBone Blue power distribution circuit (USB not shown):

![Power Distribution Diagram](images/Syno-pwr-1a.png)

The solution I found relies on a module provided by the librobotcontrol project
(https://github.com/StrawsonDesign/librobotcontrol) : **rc_battery_monitor** \
From what I understand, it is aimed to be run as a service in a debian based
distribution. Following are the steps I used to build  it with buildroot tools
from my forked version of the project :

* Get the sources
* Build the library, the examples (optional) and the rc_battery_monitor binary
* Copy the binary to the firmware image
* Copy the library to the firmware image and create symlinks
* Update /etc/inittab to start rc_battery_monitor as a daemon

As in previous section, I use a Pololu USB AVR Programmer V2.1 and a homemade
cable to get a terminal console (minicom). The Ethernet connection (via USB)
is used to make the copies.

##### 10.3.1 Get the sources

```
  $ export LC_ALL=C
  $ cd $ARDUPILOT_BLUE
  $ git clone https://github.com/drone-labs/librobotcontrol.git

  $ cd librobotcontrol
  $ git config user.name drone-labs
  $ git commit -a --allow-empty-message -m ''
  $ git fetch --prune
  $ git submodule update --init --recursive
    
  See all available branches
  $ git branch -a
    * master
      remotes/origin/HEAD -> origin/master
      remotes/origin/master
      remotes/origin/pwmfix
      remotes/origin/testing
      remotes/origin/v1.1

  Create a new branch (the master branch needs to be up to date)
  $ git pull
  Create the local branch
  $ git checkout -b dlabs
    Switched to a new branch 'dlabs'

  Add a new remote for the branch
  $ git remote add dlabs https://github.com/drone-labs/librobotcontrol.git
```
##### 10.3.2 Build the library, the examples (optional) and rc_battery_monitor
In each subdirectory (library, examples and services/rc_battery_monitor), I have
just changed the Compiler and Linker path in the Makefile :
```
  $ cd library | examples | services/rc_battery_monitor
  $ cat Makefile
    ...
    # buildroot Toolchain
    PREFIX    := arm-linux
    GCC_DIR   := ../../buildroot/output/host/bin
    # compiler and linker binaries
    CC        := ${GCC_DIR}/${PREFIX}-gcc
    LINKER    := ${GCC_DIR}/${PREFIX}-gcc
    ...

```
The builds should then work from each subdirectory :
```
  $ cd library | examples | services/rc_battery_monitor 
  $ make
```
##### 10.3.3 Copy the binary to the firmware image
```
  $ cd $ARDUPILOT_BLUE/services/rc_battery_monitor
  $ scp bin/rc_battery_monitor root@192.168.7.2:/usr/sbin/
```
##### 10.3.4 Copy the library to the firmware image and create symlinks
```
  $ cd ../../library
  $ scp lib/librobotcontrol.so.1.0.4 root@192.168.7.2:/usr/lib/
```
Switch to Target side, then :
```
  # cd /usr/lib
  # ln -sf librobotcontrol.so.1.0.4 librobotcontrol.so.1
  # ln -sf librobotcontrol.so.1.0.4 librobotcontrol.so

```
##### 10.3.5 Update /etc/inittab to start rc_battery_monitor as a daemon
Still on Target side :
```
  # cat /etc/inittab
    ...
    # Startup the system
    ::sysinit:/bin/mount -t proc proc /proc
    ::sysinit:/bin/mount -o remount,rw /
    ::sysinit:/bin/mkdir -p /dev/pts /dev/shm
    ::sysinit:/bin/mount -a
    ::sysinit:/sbin/swapon -a
    null::sysinit:/bin/ln -sf /proc/self/fd /dev/fd
    null::sysinit:/bin/ln -sf /proc/self/fd/0 /dev/stdin
    null::sysinit:/bin/ln -sf /proc/self/fd/1 /dev/stdout
    null::sysinit:/bin/ln -sf /proc/self/fd/2 /dev/stderr
    ::sysinit:/bin/hostname -F /etc/hostname
    # start the Battery monitor as a daemon
    null::respawn:/usr/sbin/rc_battery_monitor
    # now run any rc scripts
    ::sysinit:/etc/init.d/rcS

    # Put a getty on the serial port
    console::respawn:/sbin/getty -L  console 0 vt100 # GENERIC_SERIAL

    # Stuff to do for the 3-finger salute
    #::ctrlaltdel:/sbin/reboot

    # Stuff to do before rebooting
    ::shutdown:/etc/init.d/rcK
    ::shutdown:/sbin/swapoff -a
    ::shutdown:/bin/umount -a -r

```




## Build Ardupilot
As stated above the firmware is shipped with a complete custom version of ardupilot suite.
Thanks to buildroot tools, cross-compile ardupilot is not so difficult. In order to avoid
any libc conflict, I use the fresh buildroot toolchain built in previous step. Programs
will be linked against uClibc. Building a static version of ardupilot will also work.
I forked ardupilot (https://github.com/ArduPilot/ardupilot) and created a new branch named
drone-labs.  
My first build attempt failed due to three gcc warnings treated as errors. These were caused
by gcc complaining about source and destination strings length mismatch upon a **`snprintf()`**
call. Giving a little more room to the destination strings did the job.
After some investigations about the annoying "**`RCOutputAioPRU.cpp:SIGBUS error generated`**"
error, I guessed this was caused by some device tree malformation, bypassed by three gpio
lines export done into the code (gpio lines 5, 65 and 105 in **`AP_HAL_Linux/GPIO_BBB.cpp`**,
**`GPIO_BBB::init()`** function). Using my provided device tree, this code section is no more
needed and has been commented out.

In **`libraries/AP_HAL_Linux/HAL_Linux_Class.cpp`**, I changed "**`ttyO4`**" to "**`ttyS4`**" where the
RC inputs drivers are instantiated (around line 140) :

	static RCInput_Multi rcinDriver {
	   2,
	   new RCInput_AioPRU,
	   new RCInput_RCProtocol(NULL, "/dev/ttyS4")
	};

From what I understand, two driver classes are instantiated; the first, **`RCInput_AioPRU`**
is "attached" to the **`PRU_E_B`** signal (E4 pin4). The second, **`RCInput_RCProtocol(NULL, "/dev/ttyS4")`**
is "attached" to the **`UART4_RX`** signal (DSM2 pin3). It can handle Sbus (here NULL) or dsm protocol
over the given serial port (here /dev/ttyS4).

### 1. Get the sources

	$ cd $ARDUPILOT_BLUE
	$ git clone https://github.com/drone-labs/ardupilot
	$ cd ardupilot
    
### 2.  Install the prerequisites

>The script Tools/environment_install/install-prereqs-ubuntu.sh is
a good start point to see which packages are missing on the system...

### 3. Update the repository

	$ git fetch --prune
    
### 4. See all available branches

	$ git branch -a
	...

### 5. Switch to the drone-labs branch

	$ git checkout drone-labs

### 6. Update repository

	$ git submodule update --init --recursive

### 7. Configuration

First, update the PATH environment variable, according to **`ARDUPILOT_BLUE`** setting:

	$ export AP_DIR=$ARDUPILOT_BLUE/ardupilot/Tools/autotest  
	$ export GCC_DIR=$ARDUPILOT_BLUE/buildroot/output/host/bin  
	$ export PATH=$GCC_DIR:$AP_DIR:$PATH

Then Configure the Ardupilot build engine (waf) to build programs for the BBBlue
and use the buildroot toolchain:

	$ ./waf configure --board=blue --toolchain=arm-linux

### 8. Build the programs

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

### 9. Update the target

Assuming the board is running the previously built firmware,
copy the applications to the target filesystem correct location

First, ssh into the board and stop the application:

	$ ssh root@192.168.7.2  (default password = root)
	# /etc/init.d/S60arduplane stop

Exit ssh session and do the copies

	# exit
	$ scp ./build/blue/bin/a* root@192.168.7.2:/usr/bin/ardupilot

Relog into the board and restart the application (a **reboot** is safer)

	$ ssh root@192.168.7.2
	# /etc/init.d/S60arduplane start

While logged in, take a first look at the board resources consumption :

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

### 10. Extras
#### 10.1 Serial Ports
On the BeagleBone blue, up to 8 Serial ports can be used for various
purposes. 




#### 10.2 Enable MAVLink2
In order to enable MAVLink2 on a particular serial port, the manual says
to set the corresponding **SERIALN_PROTOCOL** parameter to **MAVLink2**,
with **N** the port number (0 to 7). Whenever this parameter is changed,
Arduplane must be restarted.
My first attempts showed that despite a correct SERIAL0_PROTOCOL value,
no MAVLink2 message was sent on this channel.
Digging a little bit in the source code, I found the reason in
**`GCS_MAVLink/GCS_Common.cpp/init()`** function :

```
       ...
  155  if (mavlink_protocol == AP_SerialManager::SerialProtocol_MAVLink2) {
  156    // load signing key
  157    load_signing_key();
  158    if (status->signing == nullptr) {
  159      // if signing is off start by sending MAVLink1
  160      status->flags |= MAVLINK_STATUS_FLAG_OUT_MAVLINK1;
  161    }
  162  }
  163  else {
  164    // user has asked to only send MAVLink1
  165    status->flags |= MAVLINK_STATUS_FLAG_OUT_MAVLINK1;
  166  }
  167
  168  // Always start with MAVLink2 on first port
  169  if (chan == MAVLINK_COMM_0) {
  170    // Always start with MAVLink2 on first port
  171    status->flags &= ~(MAVLINK_STATUS_FLAG_OUT_MAVLINK1);
  172  }
       ...

```

Thus, if the serial protocol is set to MAVLink2 and signing is disabled, the
AP falls back to MAVLink1, and whatever the SERIAL0_PROTOCOL setting, the
protocol is forced to MAVLink1... In order to enable MAVLink2 without signing
and not force the first serial port to MAVLink1, I changed this portion of
code to (the quick and dirties printf() can be safely removed) :


```
       ...
  153  printf("GCS_MAVLINK::init() : Channel #%u Protocol = MAVLink", (uint8_t)chan);
  154
  155  if (mavlink_protocol == AP_SerialManager::SerialProtocol_MAVLink2) {
  156    printf("2\n");
  157    // Enable MAVLink2 without signing
  158    // load signing key
  159    // load_signing_key();
  160    //if (status->signing == nullptr) {
  161    // if signing is off start by sending MAVLink1
  162    //  status->flags |= MAVLINK_STATUS_FLAG_OUT_MAVLINK1;
  163    // }
  164    status->flags &= ~(MAVLINK_STATUS_FLAG_OUT_MAVLINK1);
  165  }
  166  else {
  167    // user has asked to only send MAVLink1
  168    printf("1\n");
  169    status->flags |= MAVLINK_STATUS_FLAG_OUT_MAVLINK1;
  170  }
  171
  172  // Always start with MAVLink2 on first port
  173  //if (chan == MAVLINK_COMM_0) {
  174  //  printf("GCS_MAVLINK::init() : MAVLINK_COMM_0 Forced to MAVLink2\n");
  175  // Always start with MAVLink2 on first port
  176  //  status->flags &= ~(MAVLINK_STATUS_FLAG_OUT_MAVLINK1);
  177  //}
       ...
```

## Ground Station
I use QGroundstation-3.4.4 (newer versions fail because of my outdated Linux
Mint version...). My RC Radio transmitter is a **Radiolink AT10**.
The **Radiolink R12DS** Receiver SBus Signal is connected straight to pin4
of E4 Header (PRU_E_B).

### 1. Discrete Outputs
The goal is to control some GPIO outputs available on the BeagleBone
Blue from an equal number of **2 positions switches** of the transmitter.
This can be done with the help of the **AP_RELAY** library.
On the BeagleBone Blue, 2 GPIO Headers (6 pins JST-SH) are available :
**GP0** and **GP1**

**GP0 Header**

	                  ZCZ     GPIO       GPIO
	Pin   Signal      Pin     Id.        Num.     BBblack
	======================================================
	 1    GND
	 2    3.3V
	 3    GPIO1_25    U16    GPIO1_25     57      N/A
	 4    GPIO1_17    V14    GPIO1_17     33      P9_23
	 5    GPIO3_20    D13    GPIO3_20    116      P9_91
	 6    GPIO3_17    C12    GPIO3_17    113      P9_28

**GP1 Header**

	                  ZCZ     GPIO       GPIO
	Pin   Signal      Pin     Id.        Num.     BBblack
	======================================================
	 1    GND
	 2    3.3V
	 3    GPIO3_2     J15    GPIO3_2      98      N/A
	 4    GPIO3_1     H17    GPIO3_1      97      N/A
	 5    LED_RED     R7     GPIO2_2      66      P8_07
	 6    LED_GRN     T7     GPIO2_3      67      P8_08

For this test, I use the GPIOs available on **GP0** Header. On the Transmitter,
4 channels are bound to **4 switches** :

	Switch  Channel
	===============
	   A       7
	   B       8
	   E       9
	   F      10

From QGroundControl Parameters window, the selected **RC channels** are bound to
Ardupilot **Relay Output Functions** :

	  Transmitter    |           QGroundControl
	Switch  Channel  |  Parameter     Value     Function
	=================|======================================
	  A        7     |  RC7_OPTION     28     Relay1 ON/OFF
	  B        8     |  RC8_OPTION     34     Relay2 ON/OFF
	  E        9     |  RC9_OPTION     35     Relay3 ON/OFF
	  F       10     |  RC10_OPTION    36     Relay4 ON/OFF

Finaly, from the same window, the **Relay Output Pins** are bound to
the **GP0 Header Pins** :

	Parameter     Value    Caption
	======================================================
	RELAY_PIN       57     BB Blue GP0 pin3
	RELAY_PIN2      49     BB Blue GP0 pin4
	RELAY_PIN3     116     PX4IO ACC2/BB Blue GP0 pin5
	RELAY_PIN4     113     PX4IO Relay1/BB Blue GP0 pin6
	RELAY_PIN5      -1     Disabled
	RELAY_PIN6      -1     Disabled


### 2. Discrete Inputs
The goal is to send some GPIO inputs status to the Ground Station.
This can be done with the help of the **AP_BUTTON** library.
The GPIOs mentionned above (**GP0** and **GP1**) can be used, or
any other GPIO available on the board. For this test I use 2 inputs,
the first connected to the **MODE** Button and the second connected
to the **PAUSE** Button.

	           ZCZ     GPIO       GPIO
	Button     Pin     Id.        Num.     BBblack
	===============================================
	MODE       U6     GPIO2_4      68      P8_10
	PAUSE      T6     GPIO2_5      69      P8_09

From QGroundControl Parameters window, **Button Change Event**
reporting is enabled, the selected **GPIOs** pins are bound
to **BTN_PIN1** and **BTN_PIN2** and the number of MAVLink
**BUTTON_CHANGE** Messages to send (each time such an event
is caught) is set to 5 :

	Parameter         Value     Function
	====================================================
	BTN_ENABLE          1       Enable Button Reporting
	BTN_PIN1           68       Button1 GPIO pin
	BTN_PIN2           69       Button2 GPIO pin
	BTN_PIN3           -1       Disable Button3 input
	BTN_PIN4           -1       Disable Button4 input
	BTN_REPORT_SEND     5       Number of time the Message
	                            is to be sent

For now, the **BUTTON_CHANGE** message is well received by the Ground
Station but can only be observed in the **MAVLink Inspector** Window.
Still investigating on how to make it more friendly in the GUI (Custom
Command, Plugin ?).

**NOTES**

>The **BUTTON_CHANGE** Message is MAVLink2 (Id=257); therefore the Serial
port used to communicate with the Ground Station must be configured to use
this Protocol Version (eg. SERIAL0_PROTOCOL = MAVLink2 for the first Serial
port, defined through the **-A** argument of Arduplane command line).


## ToDo
add QGroundControl tips  
Build ardupilot examples  








































