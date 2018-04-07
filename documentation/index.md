








# Flashing the LB6M to a Brocade TurboIron 24X

## Disclaimer & Caveats
We are not responsible for any damaged devices or property resulting from this guide. This guide assumes you own a legitimate Brocade TurboIron and therefore have rights to the firmware & its use. The software itself requires no license or activation to fully function, but legally you need the rights to the firmware. Some things will also change due to hardware differences:

* The SFP+ port activity/status LEDs stop doing anything. The copper ports and chassis LEDs continue to work as normal. The Quanta uses a CPLD to multiplex the LED signals, while the Brocade uses native CPU I/O. There's no way around this difference.

* The Brocade only has one Out Of Band management port, so the code is only aware of the #1 OOB port. The #2 management port will no longer do anything. 
* Due to a difference in PCB trace layout for the management port PHY, the management-only port may link at slower speeds than it did under Fastpath (eg at 100mbit). In a rare case, we saw it stop linking altogether until a software revert. For more information, please [click here](http://brokeaid.com/mgmt/). This only affects the management port. All other ports, including copper ports function perfectly. In-band management functions perfectly, it is only the OOB port PHY affected.
* If you don't like the Brocade OS, or have other issues, you can always flash back to 100% stock Fastpath using the revert guide to the left, so none of this is permanent.  

If you're looking to purchase an LB6M, we recommend [UnixPlus](https://www.unixplus.com/products/quanta-lb6m-24-port-10gbe-sfp-4x-1gbe-l2-l3-switch) - their stock is all brand new and of known origin.
## Prerequisites

This guide assumes you're familiar with the basics like TFTP, obtaining a serial console to the device, etc. If you're not, this guide is probably not for you. Before touching your switch, read this document from beginning to end to get a basic idea of what you'll be doing - do **not** skip this step.  

The risk of doing this is mitigated if you're prepared and follow closely. It's a good idea to have the switch on a UPS while you do this, if you lose power after the ```erase``` command before you've flashed the new bootloader, your device will be bricked (however it can be recovered with a PowerPC capable JTAG unit).  

Download the firmware ZIP below. It contains your bootloader, OS, and all the documentation you'll need.  

[```Brocade Firmware Zip```](http://brokeaid.com/files/Brocade-TI.zip)  
```Zip Updated: 03-01-2018```  
```MD5: 44e00c38d996c2dfa33cda264b6be070```  

Connect to the serial console  port on the switch and open a terminal window (9600 8N1). Also be sure to connect the #1 management port on the switch to a network that has layer 2 access to your tftp server.

## Image Selection
You need to decide which image you want to flash. There are two codetrains for this switch - v7 and v8. v8 is the "latest and greatest", and technically a beta release. However v7 is the "LTS" branch and has had more QA and more bug fixes (several months worth) compared to v8.  

In 90% of cases, I recommend using the v7 codetrain. The v8 has one known issue already (severe LACP flapping in an STP environment), while the v7 image is flawless. Keep in mind both versions have the same L2/L3 features - so you're not missing out on anything using v7.

Once you decide which version to use, you need to decide between the L2 and L3 image. The Layer 3 image is the full OS including everything - all routing protocols as well as all L2 features. This is what should be used in 99% of cases, especially if you want to use the config guide on this site. The L2 image is layer 2 switching only. If you have a special case where this is needed, it's included.  

**Note:** In 99% of cases, you should choose the v7 layer 3 image. If you're unsure about anything, use this version.

Now that you've chosen, copy ```brocadeboot.bin``` from the Bootloader folder to your TFTP server root directory. Then take whatever OS image you decided upon, and rename it to ```brocadeimage.bin``` - this ensures all the commands in this guide match. Put your new ```brocadeimage.bin``` on your TFTP server as well. If you're on windows and need a temporary TFTP server, I recommend [Tftpd32 Portable Edition](http://tftpd32.jounin.net/tftpd32_download.html).




## Flash Preparation 

Reboot the switch while watching the serial output, it should prompt you to hit any key (do so) to interrupt boot and drop you into the u-boot console, which should look like this:

```
=>
```

With all the following commands, copy and paste them *exactly* as you see them. Do not try to manually type them. 
After 50+ successful flashes, the first brick was a result of a typo from someone trying to manually type.

Use the memory read command to verify your Quanta bootloader is where it should be - this ensures the commands to follow will use the correct location:

```
md 0xfff80000 20
```

the output should match this exactly:

```
fff80000: 27051956 552d426f 6f742032 3030392e    '..VU-Boot 2009.
fff80010: 30362028 41707220 31392032 30313120    06 (Apr 19 2011 
fff80020: 2d203135 3a35373a 30362900 60000000    - 15:57:06).`...
fff80030: 3c20d000 60213f80 38000000 9401fffc    < ..`!?.8.......
fff80040: 9401fffc 9421fff8 3c00ffff 6000fffc    .....!..<...`...
fff80050: 9421fff8 9001000c 48000005 7dc802a6    .!......H...}...
fff80060: 800e171c 7dc07214 480020dd 3c600002    ....}.r.H. .<`..
fff80070: 60631200 7c600124 4c00012c 48002065    `c..|`.$L..,H. e
```

If the output on your switch does not match this exactly, **STOP!** Pastebin your switches output and get in touch with us on [ServeTheHome](https://forums.servethehome.com/index.php?threads/turbocharge-your-quanta-lb6m-flash-to-brocade-turboiron.17971/).

Carrying on, assuming your ```md``` output matched ours: It's time to load in the Brocade bootloader to a safe temporary location in RAM. You also need to set a temporary IP for the switch, as well as set the IP of your tftp server destination:  

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

## Erasing and replacing the bootloader

You now have the Brocade bootloader we need stored in RAM. We need to erase the existing bootloader, then copy the Brocade loader from that RAM address to the bootloader address. From here on, be incredibly careful, and follow the commands exactly.  


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

If it matches, continue on to **Booting Brocade** below - the risky part is over. However if it doesn't, don't panic. Does it match the output you got earlier when you ran ```md 0xfff80000 20``` at the beginning of this guide? If so, that means the Quanta bootloader is still there. Either you didn't properly disable write protection, or something else has gone wrong. You can reboot into Quanta like normal, and contact us on the forums. 

However if it matches neither, something has failed. We have yet to see this, but just in case you do -  be sure you're running the exact commands here, and do the guide again from ```tftpboot 0x100000 brocadeboot.bin``` and onwards until you get the bootloader where it should be. If you follow the commands, it should work.  **Do not reboot or pull power until this is resolved.**  

## Booting Brocade
You now have the Brocade bootloader in the proper section of the PowerPC flash. Now we just need to reboot! 

```
reset
```
It will reboot into the Brocade bootloader which should drop you at a prompt similar to this:

```
Monitor>
```
 In the Brocade software, over serial or telnet, you need to use shift+backspace to backspace. You can remedy this by changing your Putty/terminal settings to "Control+H" for backspace method under Terminal>Keyboard and backspace won't require shift.  

First we need to give the bootloader a temporary IP, then reflash the bootloader using Brocade's flash routine. This fixes the boot sector flash permissions:
```
ip address 192.168.1.50/24
copy tftp flash 192.168.1.49 brocadeboot.bin boot
```
After a few seconds it should complete the flashing process. Now that the boot sector matches stock, we can flash the main OS:
```
copy tftp flash 192.168.1.49 brocadeimage.bin primary
```

Your switch is now fully flashed to Brocade. To boot into the OS for the first time run the following:
```
boot system flash primary
```

That's it! This first official boot will take a few minutes as it copies the primary image to the backup secondary image partition. The bootloader still has not been re-run since we reflashed it using Brocade's routine, so it's a good idea to make that happen by reloading the switch once it finishes booting:

```
enable
reload
```
The switch will completely reboot, re-initializing the brocade bootloader, and you're ready to go. To ditch the serial cable and telnet/SSH to the switch, follow the [L3 Quick Guide](http://brokeaid.com/config/) on this site to give it an IP.

Some commands to check out your new system:

```
show version
show flash
show chassis
show media
```
Please check out and follow the included guides in the Documentation folder to configure your new switch. A quick guide is available on the left, but this site is not a substitute for learning Brocade's documentation.

## Fixing The MAC Address

Flashing the switch resets the base MAC address to a default 00e0.5200.0100 - on its own this isn't a problem, but if you connect multiple flashed switches you're going to have serious collision issues. We highly recommend taking the extra minute to follow the [MAC Reset Guide](http://brokeaid.com/mac/).

## Fan Speeds
The Brocade firmware has the ability to set fan speeds and quiet it down. [This video](https://www.youtube.com/watch?v=QbMITnNv2FM) shows the audible difference. The OS has 3 fan speeds it automatically cycles through as temperature rises and falls. To bypass this and lock the fan speeds at the lowest level, run the below once the switch has booted into the OS:
```
enable
conf t
fan-speed 1
write memory
```
Take a look at the output of ```show chassis``` and make sure your temperatures are below the indicated warning level. For 90% of environments, ```fan-speed 1``` will still keep it plenty cool.  

If you'd like to get more advanced, there's also the ```fan-threshold``` command, which allows you to customize the temperature thresholds for each fan level, instead of locking it to one speed - but that's beyond this guide. 

## SFP+ Information

Brocade does not restrict the use of optics by manufacturer, they'll take anything given it's the right protocol. However optical monitoring information is disabled unless it sees Brocade or Foundry optics.  

So if you want to see information like this :

```
telnet@Route2(config)#sh optic 5
 Port  Temperature   Tx Power     Rx Power       Tx Bias Current
+----+-----------+--------------+--------------+---------------+
5       32.7460 C  -002.6688 dBm -002.8091 dBm    5.472 mA
        Normal      Normal        Normal         Normal
```
You'll need to pick up some official Brocade or Foundry optics on ebay, or buy some flashed optics from FiberStore.


### Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  
**fvanlint** from STH for being our first method tester

### Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.  

```Documentation version:``` [ v3.1 (04-07-18)](https://github.com/Fohdeesha/quanta-brocade/commits/master) 