




## Getting Started


Brocade's CLI is 80% identical to Cisco, with the majority of the differences being related to VLAN configuration. This guide will introduce you to the basics like vlans, SSH, inter-vlan routing, etc. The Layer 3 firmware comes with all ports in VLAN 1 by default, so if you just need layer 2 switching, you can leave the config as-is and use the out of band management port to talk to the switch.  

This guide was intended for the v8 layer 3 OS image. Everything should work in the v7 layer 3 image as well, but it's not guaranteed. If you're running the L2 only image from either codetrain, most of these commands will not work.  

Please keep in mind that any commands you run take effect immediately - however they have not been saved in flash, so they will disappear on reboot. To commit changes to flash, use the command ```write memory```.


To enter the enable level:
```
enable
```
To make changes you'll need to then enter the configure level:
```
configure terminal
```
Everything can be shortened as long as it doesn't match another command, so the below would also work instead of the above:

```
con t
```
You have tab completion - make use of it. If you're ever curious about a command or arguments it will take, just type it and hit tab, it'll give you info and options. You can also just hit tab with no text and it'll show you all commands available at the current level.  

Let's start by giving the switch a name:

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

## SSH Access
To enable SSH access, we just need to generate a key pair. This enables the SSH server:
```
crypto key generate rsa
```
You can now SSH to the switch. Since we haven't configured user accounts, when it prompts with ```login as```, just hit enter. Remember that anyone can still SSH or telnet to your switch with no authentication!

Remember SSH (and serial console) on Brocade devices requires shift+backspace to backspace. This can be fixed by setting your putty session settings to "Control+H" for backspace method under ```Terminal > Keyboard```. Telnet does not have this "feature", backspace will work as normal in telnet sessions without special putty settings.


## Securing The Switch

If you wish to leave the switch unsecured (home lab for instance), skip this whole section. To secure the switch, we need to create an account - "root" can be any username string you wish:
```
username root password yourpasshere
```
We also need to tell it to use our new local user account(s) to authorize attempts to log in as well as attempts to enter the ```enable``` CLI level:
```
aaa authentication login default local
aaa authentication enable default local
```
We should also tell it to use our account to authorize serial console access, as well as telnet access:
```
enable aaa console
enable telnet authentication
```
On next switch reboot, the serial console will ask you to first login before any commands whatsoever are available. (If you forget your login for some reason, you can get back in via the bootloader, google fastiron password recovery).

On the topic of telnet, we should disable it entirely as it's very insecure (all data including passwords is sent in cleartext):
```
no telnet server
```
The switch console is now password protected, and you can 	SSH to it - use your new credentials to log in.



### Key Based SSH Access

If you wish to disable password-based SSH login and set up a key pair, follow this section. If not, skip it. Enable key login, and disable password login:
```
ip ssh key-authentication yes
ip ssh password-authentication no
```
Now we have to generate our key pair with [puttygen](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) on windows or ```ssh-keygen -t rsa``` on linux. The default settings of RSA @ 2048 bits works without issue. Generate the pair and save out both the public and private key. 

Copy the public key file to your TFTP server. Then use the following command to import it into your switch:
```
ip ssh pub-key-file tftp 192.168.1.49 public.key
```
You shouldn't need to be told basic key management if you're following this section, but just in case - copy your private key to the proper location on the *nix machine you'll be SSH'ing from, or if you're on windows, load it using [pageant](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html). Now when you SSH to the switch, it will authenticate using your private key.

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
This has been a brief touch on 5% of the OS if that -  we highly recommend referring to the included documentation to continue further.  

We also highly recommend Terry Henry's [youtube channel](https://www.youtube.com/user/terryalanhenry/videos). He's an engineer at Brocade and has hundreds of short, concise videos on how to do anything you can think of in the OS. Some of the newer videos might not apply to our TurboIron codebase, but 90% of them will.  

### Thanks:
[**Jon Sands**](http://fohdeesha.com/)  
[**Bengt-Erik Norum**](http://amateurfoundation.org/)  

### Contributing:
The markdown source for these guides is hosted on [**our Github repo.**](https://github.com/Fohdeesha/quanta-brocade) If you have any suggested changes or additions feel free to submit a pull request.
