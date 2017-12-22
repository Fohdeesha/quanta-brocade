
#Flashing the LB6M to a Brocade TurboIron 24X

##Disclaimer & Caveats
We are not responsible for any damaged devices or property resulting from this guide. This guide assumes you own a legitimate Brocade TurboIron and therefore have rights to the firmware & its use.
Two things will also change due to hardware differences:

* The SFP+ activity/status LEDs stop doing anything. The copper ports and chassis LEDs continue to work as normal. We believe the Brocade lights use an I2C I/O expander with a different I2C address than that of the Quanta's, or even a different set of I/O on the chipset entirely. Only an aesthetic change, everything else functions as normal.  

* The Brocade only has one Out Of Band management port. Your #2 OOB port will no longer do anything. You'll still have OOB management as usual on mgmt #1, and of course in-band management on all the normal ports.

* While it may be possible to flash back to Quanta, we haven't investigated this (the Brocade bootloader does not have the same raw copy commands), so for now assume this is a one way trip. A trip worth taking, we think.
##Prerequisites

This guide assumes you're familiar with the basics like tftp, obtaining a serial console to the device, etc. If you're not, this guide is probably not for you. Before touching your switch, read this document from beginning to end to get a basic idea of what you'll be doing - do **not** skip this step.  

The risk of doing this is low and mitigated if you're properly prepared and follow closely. It's a good idea to have the switch on a UPS while you do this, if you lose power after the ```erase``` command before you've flashed the new bootloader, your device will be bricked (however it can be recovered with a PowerPC capable JTAG unit).  

Firstly grab this [Brocade Firmware Zip](http://brokeaid.com/files/Brocade-TI.zip) - it contains your bootloader, OS, and all the documentation you'll need. We're not promising it will be available here long, so keep it somewhere safe. If you redistribute it, please do not exclude or rename any files (keep it complete).  

Start a tftp server and make sure both ```brocadeboot.bin``` and ```brocadeimage.bin``` are being served by your tftp server.  

Connect to the serial console  port on the switch and open a terminal window (9600 8N1). Also be sure to connect the #1 management port on the switch to a network that has layer 2 access to your tftp server, so it can succesfully retrieve them while in u-boot.


##Flash Preparation 

Reboot the switch while watching the serial output, it should prompt you to hit any key to interrupt boot and drop you into the u-boot console, which should look like this:

```
=>
```

With all the following commands, copy and paste them *exactly* as you see them.  

Use the memory read command to verify your Quanta bootloader is where it should be - this ensures the commands to follow will use the correct location:

```
md 0xfff80000 20
```

the output should match this exactly:

```
fff80000: 27051956 552d426f 6f742032 3030392e    '..VU-Boot 2009.
fff80010: 30362028 41707220 31392032 30313120    06 (Apr 19 2011 
fff80020: 2d203135 3a35373a 30362900 60000000    - 15:57:06)....
fff80030: 3c20d000 60213f80 38000000 9401fffc    < ..!?.8.......
fff80040: 9401fffc 9421fff8 3c00ffff 6000fffc    .....!..<......
fff80050: 9421fff8 9001000c 48000005 7dc802a6    .!......H...}...
fff80060: 800e171c 7dc07214 480020dd 3c600002    ....}.r.H. .<..
fff80070: 60631200 7c600124 4c00012c 48002065    c..|.$L..,H. e
```

If the output on your switch does not match this exactly, **STOP!** Pastebin your switches output and get in touch with us on [ServeTheHome](https://forums.servethehome.com/index.php?threads/turbocharge-your-quanta-lb6m-flash-to-brocade-turboiron.17971/) - we'll help you figure it out.

Carrying on, assuming your ```md``` output matched ours: It's time to load in the Brocade bootloader to a safe temporary location in RAM. You also need to set a temporary IP for the switch, as well as set the IP of your tftp server destination:  

Give the switch a temporary unique IP, as well as the IP of your tftp server:
```
setenv ipaddr 192.168.1.50
setenv serverip 192.168.1.49
```
Now copy the Brocade bootloader to a temporary address in RAM for holding:
```
tftpboot 0x100000 brocadeboot.bin
```

The tftpboot command should have output similar to the below:
```
=> tftpboot 0x100000 brocadeboot.bin
Enet starting in 1000BT/FD
Speed: 1000, full duplex
Using TSEC0 device
TFTP from server 192.168.1.51; our IP address is 192.168.1.142
Filename 'brocadeboot.bin'.
Load address: 0x100000
Loading: Got error 4
####################################
done
Bytes transferred = 524288 (80000 hex)
```

If you see Error 4 that's normal, just be sure the bytes transferred matches. Now you need to verify that the temporary address contains the Brocade bootloader:

```
md 0x100000 20
```

The output should match the below exactly:

```
00100000: 4d554348 02057be5 0005a2d6 00004058    MUCH..{.......@X
00100010: 00000000 00012f2c 0004d880 00600028    ....../,.....`.(
00100020: 00030030 0004ffff ffffffff 00000000    ...0............
00100030: 4e6ab6ae 07030000 74727a30 37333030    Nj......trz07300
00100040: 00000000 00000000 00000000 00000000    ................
00100050: 00000000 00000000 00000000 00000000    ................
00100060: 00000000 00000000 00000000 00000000    ................
00100070: 00000000 00000000 00000000 00000000    ................
```
If it doesn't match, **STOP**. You can safely reboot back to Quanta by typing ```reset``` or power cycling it. If you'd like, pastebin the output and get in touch with us on ServeTheHome. If it does match, continue on.

##Erasing and replacing the bootloader

You now have the Brocade bootloader we need stored in RAM. We need to erase the existing bootloader, then copy the Brocade loader from that RAM address to the bootloader address. It doesn't need saying that from here on, be incredibly careful, and follow the commands exactly.  


Disable the flash write protection:
```
protect off all
```
Erase the Quanta bootloader:
```
erase 0xfff80000 0xffffffff
```
Copy the Brocade bootloader:
```
cp.b 0x100000 0xfff80000 0x80000
```
Congratulations, you've installed the Brocade bootloader (which can now load the Brocade software image). **DO NOT REBOOT YET!** First verify the Brocade bootloader is in the bootloader location:

```
md 0xfff80000 20
```

The output from your switch should match the below exactly:

```
fff80000: 4d554348 02057be5 0005a2d6 00004058    MUCH..{.......@X
fff80010: 00000000 00012f2c 0004d880 00600028    ....../,.....`.(
fff80020: 00030030 0004ffff ffffffff 00000000    ...0............
fff80030: 4e6ab6ae 07030000 74727a30 37333030    Nj......trz07300
fff80040: 00000000 00000000 00000000 00000000    ................
fff80050: 00000000 00000000 00000000 00000000    ................
fff80060: 00000000 00000000 00000000 00000000    ................
fff80070: 00000000 00000000 00000000 00000000    ................
```

If it matches, continue on to **Booting Brocade** below - the scary part is over. However if it doesn't, stay calm. Does it match the output you got earlier when you ran ```md 0xfff80000 20``` at the beginning of this guide? If so, that means the Quanta bootloader is still there. Either you didn't properly disable write protection, or something else has gone wrong. You can reboot into Quanta like normal, and contact us on the forums. 

However if it matches neither, something has failed. We have yet to see this, but just in case you do -  be sure you're running the exact commands here, and do the guide again from ```tftpboot 0x100000 brocadeboot.bin``` and onwards until you get the bootloader where it should be. If you follow the commands, it should work.  

**Do not reboot or pull power until this is resolved.** If there is not a valid bootloader in that location, it will not boot itself. As a last resort you can try flashing the quanta bootloader back by substituting the uboot.bin in the recovery folder in all the commands mentioning brocadeboot.bin - just use uboot.bin instead. If successful, the output of  ```md 0xfff80000 20``` should match the example at the beginning of this guide, then you can reboot.

##Booting Brocade##
You now have the Brocade bootloader in the bootloader section of the PowerPC flash. Now we just need to reboot! 

```
reset
```
It will now reboot into the Brocade bootloader. In the Brocade software over serial or telnet, you need to use shift+backspace to backspace. You can remedy this by changing your Putty/terminal settings to "Control+H" for backspace method under Terminal>Keyboard and backspace won't require shift. Once you get it up and running, you can also configure SSH which uses normal backspaces.  

Firstly boot the OS image via tftp. You need to first give the bootloader a temporary unique IP, then boot the firmware file using the IP address of your tftp server:
```
ip address 192.168.1.50/24
boot system tftp 192.168.1.49 brocadeimage.bin
```

It will now boot into the full Brocade firmware, however we still need to actually flash it to the device flash as well as fix flash permissions by re-flashing the bootloader using Brocade's official bootloader flashing routine.  
  
First give the management interface an address so it can contact your tftp server:
```
enable
conf t
int management 1
ip addr 192.168.1.50/24
exit
write mem
```
Reflash the bootloader using Brocade's flash routine to fix permissions, substitute the IP with your tftp server:
```
copy tftp flash 192.168.10.49 brocadeboot.bin bootrom
```
Reboot the switch so the new bootloader fixes perms. You won't be able to write to flash until you do this:
```
reload
```

It will reboot to the bootloader because we still haven't flashed the OS. Just like above, temporarily tftp-boot the OS:
```
ip address 192.168.1.50/24
boot system tftp 192.168.1.49 brocadeimage.bin
```
If your management IP config from earlier didn't save, you'll need to redo those steps to give it an IP again. Now load and write the firmware:
```
enable
copy tftp flash 192.168.1.49 brocadeimage.bin primary
```
It now perfectly matches a stock Brocade TurboIron. Reboot and it will come up on it's own like a stock device:
```
reload
```

That's it! You can ditch the serial cable and telnet to the management IP. If you want to use SSH, you'll need to enable it - follow the included Brocade docs or the Quick Guide on the left. Some commands to check out your new system:

```
show version
show flash
show chassis
show media
```
This is the full layer 3 image that has all the layer 2 and layer 3 features, so please follow the included guides in the Documentation folder to configure your new switch. A quick guide is available on the left, but this site is not a substitute for learning Brocade's documentation.



###Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  
[**FBOM**](http://fbom.club/)  
**fvanlint** from STH for being our first method tester

###Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.  

```Documentation version: v1.2 (12-21-17)```