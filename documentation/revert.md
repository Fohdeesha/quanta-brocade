

# Reverting To Stock Fastpath

## Introduction 
If you'd like to flash back to garbage stock Fartpath, that's now possible. You'll need the firmware ZIP linked on the main page. This guide can also be used to flash the Fastpath OS to an actual Brocade TurboIron, if you wanted to do that for some ungodly reason.  

In the ```Fastpath Revert``` folder, copy both ```ubootenv.bin``` and ```lb6m.1.2.0.18.img``` to your TFTP server. If you don't see a ```ubootenv.bin``` but instead have ```uboot.bin```, you have an old version of the ZIP. Delete it and re-download. 

## Preparing U-Boot

As we are overwriting the boot sector again, the same warnings still apply. Copy and paste commands only (no typing). Have the device on a UPS if possible. 

Connect to the serial console port of the switch and open a terminal window (9600 8N1). Also be sure to connect the #1 management port on the switch to a network that has layer 2 access to your TFTP server.

Reboot the switch (```reload``` at the enable CLI level) while watching the serial output - it should prompt you to hit the ```b``` key to interrupt boot. Do so, which should take you here: 

```
Monitor>
```

Use the memory read command to verify your Brocade bootloader is where it should be - this ensures the commands to follow will use the correct location:

```
dd fff80000
```

the output should match this exactly:

```
Monitor>dd fff80000
fff80000: 4d554348 02057be5 0005a2d6 00004058
fff80010: 00000000 00012f2c 0004d880 00600028
fff80020: 00030030 0004ffff ffffffff 00000000
fff80030: 4e6ab6ae 07030000 74727a30 37333030
```

If the output on your switch does not match this exactly, **STOP!** Pastebin your switches output and get in touch with us on [ServeTheHome](https://forums.servethehome.com/index.php?threads/turbocharge-your-quanta-lb6m-flash-to-brocade-turboiron.17971/).

## Getting u-boot into RAM
Carrying on, assuming your output matched ours: It's time to load in the u-boot bootloader and prepare it for flashing. First you'll need to set a temporary IP for the switch:
```
ip address 192.168.1.50/24
```
Now copy the u-boot bootloader to a file in onboard flash named ```quanta``` (substitute the IP with the IP of your TFTP server):
```
copy tftp flash 192.168.1.49 ubootenv.bin quanta
```

After some dots it should say ```Done```. You now have u-boot stored in onboard flash. Now we need to copy it to RAM, then to the final boot sector.

Copy our ```quanta``` file to a location in RAM:

```
copy flash memory 08000000 quanta
```

Now we need to verify u-boot has been stored in RAM succesfully. Verify the beginning of u-boot:
```
dd 08000000
```
The output should match the below exactly:
```
Monitor>dd 08000000
08000000: 27051956 552d426f 6f742032 3030392e
08000010: 30362028 41707220 31392032 30313120
08000020: 2d203135 3a35373a 30362900 60000000
08000030: 3c20d000 60213f80 38000000 9401fffc
```
If your output matches, move on to the next section. If it doesn't match, you can safely abort and reboot.

## Erasing and replacing the bootloader

You now have u-boot stored in RAM - The last step is to copy it from that RAM address to the bootloader address. From here on be incredibly careful, and follow the commands exactly.


Copy u-boot from RAM to the boot sector. This single command handles erasing and writing flash properly:
```
copy memory boot 08000000 524288
```

Congratulations, you've installed the u-boot bootloader (which can now load the Quanta software image). **DO NOT REBOOT YET!** First verify u-boot is in the bootloader location:

```
dd fff80000
```

The output from your switch should match the below exactly:

```
Monitor>dd fff80000
fff80000: 27051956 552d426f 6f742032 3030392e
fff80010: 30362028 41707220 31392032 30313120
fff80020: 2d203135 3a35373a 30362900 60000000
fff80030: 3c20d000 60213f80 38000000 9401fffc
```
Verify the end instruction of the bootloader was copied correctly:

```
dd FFFFFFFC
```
The output should match the below exactly:

```
Monitor>dd FFFFFFFC
fffffffc: 4bfff004 xxxxxxxx xxxxxxxx xxxxxxxx
0000000c: xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx
0000001c: xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx
0000002c: xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx
```

If it matches, skip on to **Booting Quanta** below - the risky part is over. However if it doesn't match, don't panic. You either entered a command wrong, or skipped one. Do not reboot yet! To recover the original Brocade bootloader back, run the below command. You'll need to make sure brocadeboot.bin from the zip is on your TFTP server:

```
#for recovery only!
copy tftp flash 192.168.1.49 brocadeboot.bin boot
```
If that command finishes succesfully, you can reboot into the bootloader again and try once more.




## Booting Quanta
You now have u-boot in the proper section of flash - we just need to reboot. The "reset" command in Brocade's bootloader is bugged (it will just freeze), so to make it reboot you must pull power to the switch, then re-apply. It should boot up to a u-boot prompt:

```
=>
```
Now we must set the MAC address for u-boot and Fastpath. Your chassis MAC should be on a sticker on the side of the unit generally, or you can [generate a random MAC.](https://www.miniwebtool.com/mac-address-generator/)  Either way, replace the MAC in the command below with your target MAC, making sure to adhere to the same formatting:

```
setenv ethaddr 54:AB:3A:42:0B:42
saveenv
reset
```
After it reboots back into u-boot, we can now boot Fastpath in order to flash it. First we need to set our temporary IP address plus the IP of our tftp server, then pull the image and boot it:

```
setenv ipaddr 192.168.1.50
setenv serverip 192.168.1.49
tftpboot 0x08000000 lb6m.1.2.0.18.img
boot
```

It should boot into a FASTPATH Startup menu:

```
FASTPATH Startup -- Utility Menu

 1  - Start FASTPATH Application
 2  - Load Code Update Package
 3  - Load Configuration
 4  - Select Serial Speed
 5  - Retrieve Error Log
 6  - Erase Current Configuration
 7  - Erase Permanent Storage
 8  - Select Boot Method
 9  - Activate Backup Image
10  - Start Diagnostic Application
11  - Run Manufacturing Diagnostics
12  - Delete Manufacturing Diagnostics
13  - Reboot
```
Select ```option 2``` and follow its instructions. For transfer mode enter ```T``` for tftp, then fill out the IP's as it asks, as well as our image filename. You can leave ```gateway``` and ```subnet mask``` blank assuming your tftp server is on the same /24 network:

```
Select Mode of Transfer (Press T/X/Y/Z for TFTP/XMODEM/YMODEM/ZMODEM) []:t
Enter Server IP []:192.168.1.49
Enter Host IP []:192.168.1.50
Enter Host Subnet Mask [255.255.255.0]:
Enter Gateway IP []:
Enter Filename []:lb6m.1.2.0.18.img
Do you want to continue? Press(Y/N):  y
```
It will flash the image then ask to reboot - hit yes. It should reboot as normal all the way into the Fastpath software it was originally shipped with, and you're done. If you forgot, the default fastpath login is ```admin``` with no password.  

If you're worried about "wearing out" the onboard flash by flashing back and forth, it's a non-issue. As you can see in the [S29GL256P datasheet](http://brokeaid.com/files/flashdata.pdf), the onboard flash IC is good for 100,000 erase-write cycles per sector. 


### Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  

### Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.  