



## Getting Started


Brocade's CLI is 80% identical to Cisco, with the majority of the differences being related to VLAN configuration. This short intro will get you set up with an account, SSH, and an IP for the switch. The Layer 3 firmware comes with all ports in VLAN 1 by default, so if you just need layer 2 switching, you can leave the VLAN/virtual interface config as-is and use the out of band management port to talk to the switch.  

This guide was intended for the v8 layer 3 OS image. Everything should work in the v7 layer 3 image as well, but it's not guaranteed. If you're running the L2 only image from either codetrain, most of these commands will not work.  


To enter the enable level:
```
enable
```
To make changes you'll need to enter the configure level:
```
configure terminal
```
Everything can be shortened as long as it doesn't match another command, so the below would also work instead of the above:

```
conf t
```
You also have tab completion so if you're ever curious about a command, type it and hit tab, it'll give you a list of all arguments. You can also just hit tab with no text and it'll show you all commands available at the current level.  

Now we need to tell it to start generating our keys, so we can enable SSH:
```
crypto key generate rsa
#This creates a 1024-bit keypair
#For more options, enter the command but hit tab instead of enter
```

Give the switch a name/hostname:

```
hostname blinkenmaschine
```

## Assigning an out-of-band management IP

If you plan on using the OOB management port to talk to the switch, follow this section. If you'd rather use an in-band management IP on your VLANs, skip this section. Assuming you're still at the ```configure terminal``` level from before, run the following, replacing the IP with your own choice:

```
int management 1
ip addr 192.168.1.50/24
exit
```
You can now telnet to that IP over the OOB management port. For SSH, you'll need to complete the ```Authentication``` section.

## Assigning an in-band management IP + VLAN VE Config
You can also assign an IP to one of the VLANs, so the switch is accessible from the normal ports. You don't have to choose one or the other, you can have in-band and out-of-band management both configured simultaneously, but the IP's will need to be on different subnets.  

To give the switch an IP that's accessible via the normal ports, we need to a assign a virtual interface to the VLAN you'd like it to be accessible on, and give it an IP.  That virtual interface is configurable just like a normal physical interface. At it's config level you can set routing protocols, IP addr, etc. This is also how you enable inter-VLAN routing, you just give multiple VLAN's a router interface with an IP like below, and that's it.  

Enter the config level for a vlan:
```
 vlan 1
```
Assign it a virtual interface. The number doesn't have to match the vlan, you could give it ve 10, but it's easier to keep track of if they match:
```
router-interface ve 1
```
Exit the vlan config level:

```
exit
```
 Now we enter the interface config level for the VE we just assigned:
```
interface ve 1
```
(NOTE: the above command will give you an error if the VLAN it is assigned to has no ports. So if you're setting up VLAN 2 for example, give it a port member while you're at it's config level, and you won't encounter the error.)  

Now we just give that VE interface an IP. Note, addresses for VE's cannot be in the same subnet you've assigned to your out of band management port, for obvious reasons:
```
ip address 192.168.1.2/24
```
That's it, now anything in that VLAN can access the switch via that IP.  Exit the VE config level:
```
exit
```
## Inter-VLAN Routing
If you'd like inter-VLAN routing, follow this section. if you don't, skip down to Authentication. To enable VLAN routing you just need to do everything we did above, but for another VLAN:

```
vlan 2
untagged ethe 11
#you can also assign ranges of ports, using: untagged ethe 11 to 20 
router-interface ve 2
exit
interface ve 2
ip address 192.168.50.2/24
exit
```
They'll now route between each other, assuming your devices have gateways properly set etcetera.


## Authentication

First we need to tell it to use local user accounts for authentication instead of RADIUS etc. Assuming you're still at the ```configure terminal``` level:
```
aaa authentication enable default local
aaa authentication login default local
```

Now give the root user a password:
```
username root password yourpasshere
```
Now when logging in, it will ask for the user, use root. When you try to enter the ```enable``` level, you'll need to enter said password.

If you want no password to leave the switch unsecured, run these commands:
```
username root nopassword
ip ssh permit-empty-passwd yes
```
If you want to be able to SSH to the switch without setting up a key pair, run this command:
```
ip ssh key-authentication no
```
If you do wish to enable key based SSH login, it's beyond the scope of this intro. Refer to the Security Guide PDF included. It has information on all of the above, SSH, etc. 

## SNMP
To quickly enable SNMPv2, follow the below. SNMP v3 is available but you'll have to refer to the included documentation:
```
snmp-server community public ro
```

##Default Route & DNS

If you'd like the switch to be able to get out to the internet (For example, NTP or ping commands to external hosts or hostnames), do the following, substituting your gateway IP:

```
ip dns server-address 192.168.1.1
ip route 0.0.0.0/0 192.168.1.1
```

## NTP
To have the switch keep it's time synced via NTP, use the following. If you live in an area that doesn't use Daylight Savings, skip the ```clock summer-time``` command. Use tab completion for the timezone command to see what's available:
```
clock summer-time
clock timezone gmt GMT-05
sntp server 216.239.35.0
sntp server 216.239.35.4
```
You can specify up to 8 or as little NTP servers as you'd like. Only IP addresses are allowed, no hostnames. The IP's in the example above are Google's public NTP servers.


## Saving & Conclusions
Now of course none of the changes have actually been saved to flash, so on next boot they'll disappear. To save your current running configuration so they remain on reboot:
```
write memory
```
That's enough to get you going. Some more useful general commands:

Show a table of all interfaces:
```
show interface brief
```
To show one interface in detail:
```
show interfaces ethernet 1
#Also works for virtual interfaces:
show interfaces ve 1
```
Give a port a friendly name:
```
interface ethernet 1
port-name freenas
show interface brief ethernet 1
exit
```

To remove commands from your config, just put ```no``` in front of them:
```
interface ethernet 1
no port-name
exit
```
Show the currently active configuration:
```
show run
```
Show the configuration that's saved to flash for startup:
```
show configuration
```

Some system info:
```
show cpu
show memory
show chassis
show version
show log
```
These have all been very basic commands and most of them will take many more arguments for advanced configuration. We highly recommend referring to the included documentation to continue further.  

We also highly recommend Terry Henry's [youtube channel](https://www.youtube.com/user/terryalanhenry/videos). He's an engineer at Brocade and has hundreds of short, concise videos on how to do anything you can think of in the OS. Some of the newer videos might not apply to our TurboIron codebase, but 90% of them will.  

### Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  

### Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.
