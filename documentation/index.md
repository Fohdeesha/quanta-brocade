

# Flashing the LB6M to a Brocade TurboIron 24X

## Disclaimer & Caveats
We are not responsible for any damaged devices or property resulting from this guide. This guide assumes you own a legitimate Brocade TurboIron and therefore have rights to the firmware & its use.
Two things will also change due to hardware differences:

* The SFP+ activity/status LEDs stop doing anything. The copper ports and chassis LEDs continue to work as normal. The Quanta uses a CPLD to multiplex the LED signals, while the Brocade uses native CPU I/O. There's no way around this difference.

* The Brocade only has one Out Of Band management port. Your #2 OOB port will no longer do anything. You'll still have OOB management as usual on mgmt #1, and of course in-band management on all the normal ports.

If you're looking to purchase an LB6M, we recommend [UnixPlus](https://www.unixplus.com/products/quanta-lb6m-24-port-10gbe-sfp-4x-1gbe-l2-l3-switch) - their stock is all brand new and of known origin.
## Prerequisites

This guide assumes you're familiar with the basics like tftp, obtaining a serial console to the device, etc. If you're not, this guide is probably not for you. Before touching your switch, read this document from beginning to end to get a basic idea of what you'll be doing - do **not** skip this step.  

The risk of doing this is mitigated if you're properly prepared and follow closely. It's a good idea to have the switch on a UPS while you do this, if you lose power after the ```erase``` command before you've flashed the new bootloader, your device will be bricked (however it can be recovered with a PowerPC capable JTAG unit).  

First grab this [Brocade Firmware Zip](http://brokeaid.com/files/Brocade-TI.zip) ```(zip updated: 2-14-2018)``` - it contains your bootloader, OS, and all the documentation you'll need.  

Start a tftp server and make sure both ```brocadeboot.bin``` and ```brocadeimage.bin``` are being served by it. They can be found in the ```Main Flash``` folder.  

Connect to the serial console  port on the switch and open a terminal window (9600 8N1). Also be sure to connect the #1 management port on the switch to a network that has layer 2 access to your tftp server.


## Flash Preparation 

Reboot the switch while watching the serial output, it should prompt you to hit any key (do so) to interrupt boot and drop you into the u-boot console, which should look like this:

```
=>
```

With all the following commands, copy and paste them *exactly* as you see them. Do not try to manually type them. 
After 50+ successful flashes, we had our first report of a brick. It was from someone trying to manually type each command, and mistaking a "O" for a "0". Copy and paste only!   

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

However if it matches neither, something has failed. We have yet to see this, but just in case you do -  be sure you're running the exact commands here, and do the guide again from ```tftpboot 0x100000 brocadeboot.bin``` and onwards until you get the bootloader where it should be. If you follow the commands, it should work.  

**Do not reboot or pull power until this is resolved.** If there is not a valid bootloader in that location, it will not boot itself. As a last resort you can try flashing the quanta bootloader back by substituting the uboot.bin in the ```Fastpath Revert``` folder in all the commands mentioning brocadeboot.bin - just use uboot.bin instead. If successful, the output of  ```md 0xfff80000 20``` should match the example at the beginning of this guide, then you can reboot.

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
This is the full layer 3 image that ships with all features enabled, so please follow the included guides in the Documentation folder to configure your new switch. A quick guide is available on the left, but this site is not a substitute for learning Brocade's documentation.

## Fixing The MAC Address

Flashing the switch resets the base MAC address to a default of 00e0.5200.0100 - on it's own this isn't a problem, but if you connect multiple flashed switches they're all going to have the same base MAC, and you're going to have serious collision issues. We highly recommend taking the extra 2 minutes to follow the [MAC Reset Guide](http://brokeaid.com/mac/) - it also allows you to customize the serial number you see in the output of ```show version```.

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

```Documentation version:``` [ v2.1 (02-21-18)](https://github.com/Fohdeesha/quanta-brocade/commits/master) 