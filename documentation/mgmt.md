
# Management Port Behavior

## What Happens
The out of band management port is gigabit PHY - however for some users, after flashing the Brocade firmware it will instead start linking up at 100mbit. In even rarer cases, some users have seen it link at 10mbit. In all of these cases it continues to operate fine as it's only passing management traffic. In some extremely rare cases (observed on 2 chassis), it will refuse to link at all.  

You can obviously still use in-band management IP's to access the switch, or even designate another switch port in its own isolated VLAN as a management-only port via ACLs. Worst case you can revert to the stock Fastpath OS, and the management port will function perfectly as before (and your switch will be 100% back to stock).

## Why It Happens
It's hard to say with 100% certainty without using an oscilloscope on the various clocks and data lines for the management PHY, and comparing scoped values to values seen while running Fastpath (when the port works properly). What is most likely happening is the Brocade firmware tells the management CPU to use slightly different clock speed values to drive the management PHY, as the traces from the management CPU to the management PHY are shorter in the Brocade.  

It's also possible the pre-emphasis configuration registers on the SGMII data transmitter were tuned for the trace layout and length of the Brocade chassis, and are therefore slightly out of spec when that configuration is loaded via the Brocade software in the Quanta chassis.  

Sadly neither of these are fixable without disassembling the bootloader (the Brocade bootloader is what's responsible for initializing the PHY. It also remains running at all times acting as a scheduler for the main OS). Once you find these configuration values in the disassembled bootloader they can be changed to appropriate values for the Quanta chassis and re-compiled. I received an estimate from a firmware engineer of approximately $2000 USD for this amount of reverse engineering, which I'm not willing to spend for a management port I never use on a $300 switch.

## Emergency Workaround
If you're in the 1% where it stops linking completely under the Brocade firmware, you can temporarily bring it back. As it's entirely a software issue (the port goes right back to working perfectly when reverting to Fastpath software), it can be forced to link in emergency situations with a temporary software trick.

If you find yourself in a situation where you must have the management port link up and it won't, follow the below. To reiterate, we've only seen it completely cease linking twice in a hundred flashed chassis.  

Assuming you're in the Brocade bootloader, enter the ```t2``` test menu, by simply typing t2 into the bootloader prompt:

```
t2
```

You'll be dropped into a hardware test menu that looks like this:
```
     Host Peripherals Test Menu

[1  ]  PCI Bus Test

[2  ]  UART Loopback Test

[3  ]  SDRAM Test

[4  ]  FLASH and File CRC32 Checksum Test

[5  ]  System Timer Test

[6  ]  I2C Test

[7  ]  FAN Test

[8  ]  Ethernet Test

[9  ]  RPSU Test

[A  ]  Test All Items

[ESC]  Exit


 Enter your choice:
```
Enter in ```8``` for the ethernet test. On the following submenu, enter in ```6``` for "test all items" - it will run through a test of the ethernet PHY, and some of them will fail. Thankfully this test seems to have been "borrowed" from Fastpath, and it re-initializes the PHY with the MII registers/clock values/etc that Fastpath uses on the LB6M - so the port will begin working perfectly as it does in Fastpath.  

Once it finishes, your management port should be succesfully linked up. Hit ```ESC``` a few times to get out of the test menus and back to the bootloader prompt. You can now succesfully TFTP firmware images in, or whatever else you needed the management port for.  

**Note 1:** This only works until the PHY is reinitialized with the Brocade values, eg next time it reboots, or you pull power.  

**Note 2:** Unless you know what you're doing, do not run any of the other tests in the t2 menu. For instance, the I2C test wipes the onboard EEPROM, which is where your MAC address and serial are stored. If you run that test, you'll need to redo the Customizing MAC guide.


### Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  

### Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.  
