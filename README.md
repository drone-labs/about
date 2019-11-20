# Ardupilot Blue
 Building a lightweight buildroot based firmware for the BeagleBone Blue.


## Credits
 Another ardupilot setup for the BeagleBone Blue?
 Yes. Mirko Denecke (https://github.com/mirkix/ardupilotblue) has ported
 ardupilot core code for the board and imfatant maintains an excellent guide 
 (https://github.com/imfatant/test) to make things work together.
 So, why? The existing solution relies on a Debian distribution
 (stretch-console) which provides many, many features that i do not aim to use.
 Instead, i want to have a small memory footprint system though, staying
 reliable, configurable, extendable and reproducible.
 For that, i use my favorite build system: Buildroot (https://buildroot.org)

