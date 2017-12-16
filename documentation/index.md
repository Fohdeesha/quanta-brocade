
#Flashing the LB6M to a Brocade TurboIron 24X
##Prerequisites & Preparation 

This guide assumes you're familiar with the basics like tftp, obtaining a serial console to the device, etc. If you're not, this guide is probably not for you. Before touching your switch, read this document from beginning to end to get a basic idea of what you'll be doing. 

If you miss a part of it you'll render your switch unusable, but that risk can be entirely mitigated by being prepared and following closely. It's also a good idea to have the switch on a UPS while you do this, if you lose power after the *erase* command before you've flashed the new bootloader, your device is a brick (however it can be recovered with a PowerPC capable JTAG unit, if you're desperate).

Firstly grab this [Brocade Firmware Zip](http://brokeaid.com/files/Brocade-TI.zip) - it contains your bootloader, OS, and all the documentation you'll need. We're not promising it will be available here long, so keep it somewhere safe. If you redistribute it, please do not exclude or rename any files (keep it complete). Start a tftp server and make sure both *brocadeboot.bin* and *brocadeimage.bin* are being served by your tftp server. 
Connect to the serial console  port on the switch and open a terminal window. Also be sure to connect one of the management ports on the switch to a network that has layer 2 access to your tftp server, so it can succesfully retrieve them while in u-boot.


##Flash Preparation 

Reboot the switch while watching the serial output, it should prompt you to hit any key to interrupt boot and drop you into the u-boot console, which should look like this:

```
=>
```

With all the following commands, copy and paste them *exactly* as you see them. If a line starts with # it's a comment. Don't try to run that line. 

Now use the memory read command to verify your quanta bootloader is where it should be, to ensure the commands to follow will use the correct location:

```
md 0xfff80000 20
```

the output should match this exactly:

```
fffff000: 38000002 7c12fba6 7c13fba6 7c304aa6    8...|...|...|0J.
fffff010: PLACEHOLDER PLZ REPLACE ME 7c52fba6    |0K.<@..`B..|R..
fffff020: 4c00012c 7c53fba6 4c00012c 7c0004ac    L..,|S..L..,|...
fffff030: 3c20fff8 7c3f0ba6 38200100 7c3063a6    < ..|?..8 ..|0c.
```

If the output on your switch does not match this exactly, **STOP!** Pastebin your switches output and get in touch with us on [ServeTheHome](https://forums.servethehome.com/index.php?threads/quanta-lb6m-10gbe-discussion.8002/) - we'll help you figure it out.

Carrying on, assuming your **md** output matched ours: It's time to load in the brocade bootloader to a safe temporary location in RAM. You also need to set a temporary IP for the switch, as well as set the IP of your tftp server destination:


```
#give the switch a temporary unique IP
setenv ipaddr 192.168.1.50 #VERIFY THIS IS CORRECT
#give it the IP of your tftp server
setenv serverip 192.168.1.49 #VERIFY THIS IS CORRECT
#copy the brocade bootloader to a temp address in RAM
tftpboot 0x100000 brocadeboot.bin
```

The tftpboot command should have output like below:
```
proper tftpboot output goes here #PLACEHOLDER
```

Now you need to verify that the temporary address contains the brocade bootloader:

```
md 0x100000 20 #PLACEHOLDER
```

The output should match the below exactly:

```
fffff000: 38000002 7c12fba6 7c13fba6 7c304aa6    8...|...|...|0J.
fffff010: PLACEHOLDER PLZ REPLACE ME 7c52fba6    |0K.<@..`B..|R..
fffff020: 4c00012c 7c53fba6 4c00012c 7c0004ac    L..,|S..L..,|...
fffff030: 3c20fff8 7c3f0ba6 38200100 7c3063a6    < ..|?..8 ..|0c.
```
If it doesn't match, **STOP**. You can safely reboot back to quanta by typing reset or power cycling it. if you'd like, pastebin the output and get in touch with us on ServeTheHome. If it does match, continue on.

##Erasing and replacing the bootloader

You now have the brocade bootloader we need stored in RAM. We need to erase the existing bootloader, then copy the brocade BL from that ram address to the bootloader address. It doesn't need saying that from here on, be incredibly careful, and follow the commands exactly:

```
#disable flash write protection
protect off all
#erase the quanta bootloader
erase 0xfff80000 0xffffffff
#copy the brocade bootloader
cp.b 0x100000 0xfff80000 0x80000
```
Congratulations, you've installed the brocade bootloader (which can load the brocade software image). **DO NOT REBOOT YET!** First verify the brocade bootloader is in the bootloader location:

```
md 0xfff80000 20
```

The output from your switch should match the below exactly:

```
fffff000: 38000002 7c12fba6 7c13fba6 7c304aa6    8...|...|...|0J.
fffff010: PLACEHOLDER PLZ REPLACE ME 7c52fba6    |0K.<@..`B..|R..
fffff020: 4c00012c 7c53fba6 4c00012c 7c0004ac    L..,|S..L..,|...
fffff030: 3c20fff8 7c3f0ba6 38200100 7c3063a6    < ..|?..8 ..|0c.
```

If it matches, continue on to **Booting Brocade** below - the scary part is over. However if it doesn't, stay calm. Does it match the output you got earlier when you ran "md 0xfff80000 100" at the beginning of this guide? If so, that means the Quanta bootloader is still there. Either you didn't properly disable write protection, or something else has gone wrong. You can reboot into quanta like normal, and contact us on the forums. 

However if it matches neither, something has gone very wrong. Be sure you're running the exact commands here, and do the guide again from "tftpboot 0x100000 brocadeboot.bin" and onwards until you get the bootloader where it should be. If you follow the commands, it should work. **Do not reboot until this is resolved.** If there is not a valid bootloader in that location, it will not boot itself. As a last resort you can try flashing the quanta bootloader back by substituting the uboot.bin in the recovery folder in all the commands mentioning brocadeboot.bin - just use uboot.bin instead. If successful, the output of  "md 0xfff80000 20" should match the example at the beginning of this guide, then you can reboot.

##Booting Brocade##
You now have the Brocade bootloader in the bootloader section of the PowerPC flash. Now we just need to reboot! 

```
reset
```
It will now reboot into the Brocade bootloader. We need to tftpboot the brocade software image:

```
#give the bootloader a temporary unique IP
ip address 192.168.1.50/24
#boot the firmware, using the IP of your tftp server
boot system tftp 192.168.1.49 brocadeimage.bin
```

It will now boot into the full brocade firmware, however we still need to actually flash it to the device flash:

```
enable
#for user, use root. If it asks for a password, just hit enter
#give the mgmt interface an address so it can contact your tftp server
conf t
int mgmt 1
ip addr 192.168.1.50/24
exit
write mem
#now load and write the firmware
copy tftp flash 192.168.1.49 brocadeimage.bin primary
```

That's it! You can now type reload and hit enter, and it will reboot the switch. It'll come up as a TurboIron. Some commands to check out your new system:

```
show version
show flash
show chassis
show media
```
This is the full layer 3 image that has all the layer 2 and layer 3 features, so please follow the included userguide to configure your new switch. A quick guide is available on the left.