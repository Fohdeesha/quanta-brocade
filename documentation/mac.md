

# Fixing The MAC Address

## Why Do This?
The flashing procedure resets the switches base MAC to 00e0.5200.0100 - this is the MAC address used for the management port, as well as the base address used to generate MACs for all the individual ports and vlan interfaces. Being set to this default MAC isn't a problem on it's own, but will cause serious issues if you use multiple flashed switches together.

## Customizing The Chassis
There's a diagnostic menu that allows you to easily enter a new MAC as well as serial information (the serial string is what's displayed in the output of ```show version```). You'll need serial port access to the switch (just like you had during the flash guide). Assuming that's taken care of, reboot the switch, then hit b to enter the bootloader prompt:
```
enable
reload
#watch the output and hit b when prompted
```
You should be dropped into the Brocade bootloader with a prompt that looks like this:
```
Monitor>
```
Run the following command to enable a hidden diag menu on next boot:
```
set diag_mode
```
Now tell the system to boot:
```
boot system flash primary
```
It will boot into the diagnostic menu, which should resemble the below:

```
     Diagnostic Test Main Menu v1.0 ( LB6M )

[1  ]  File Management

[2  ]  Board Information

[3  ]  Diagnostic Test

[4  ]  Manufacturing Test Mode Set

[5  ]  Test Error Log File Management

[6  ]  PING

[7  ]  Reset

[8  ]  For Vibration Test

 Enter your choice:
```
Yes, it does indeed say "LB6M" in the menu title, and no, this is not a piece of software left over from the Quanta firmware. Brocade seemingly re-used a bit of code from Quantas/Broadcoms reference, which again goes to show how similar the Brocade TurboIron is to our LB6M.  

Anyway, we need to customize the chassis information. Enter ```2``` into the menu for ```Board Information``` - on the menu following, enter ```1``` for ```Configure Board Info```. You will enter an interactive menu that allows you to input new values for each item. 
After some trial and error it will tell you the length and format each item needs to be in. No need to be careful, you cannot break anything here. Other than the MAC address, every item in here is aesthetic only. When it asks for the MAC Address, this is where you enter in your original unique MAC. It can usually be found on the side of your switch on a sticker. If you can't find it but want to set a unique MAC, you can use an [online MAC generator](https://www.miniwebtool.com/mac-address-generator/).  

After finishing each item, it will write to EEPROM and you're done. Hit ESC until you're back to the main diagnostic menu, then enter ```7``` to reset the switch. It will reboot into the OS as normal, and now ```show chassis``` should have your new MAC at the bottom of the output. If the MAC displayed there is off by the end character, that's normal - it's using your specified base MAC to iterate upon. If you set a custom serial, you can also see that with ```show version``` - your chassis is now properly customized.  

**Note to advanced users:** This custom information is all stored in EEPROM. In the ```t2``` test menu (initiated from the bootloader or hidden OS prompt), there are tests that wipe all this data. Namely the I2C and EEPROM tests - they both write garbage data to the EEPROM. If for some reason you need to run these tests, you'll need to run this customization procedure again.



### Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  
### Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.
