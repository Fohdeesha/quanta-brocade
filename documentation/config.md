##NEEDS SERIOUS CLEANUP && FORMATTING
-----------------get in to enable level terminal-----------------

enable

-----------------------erase existing config then reload to take effect-------------

erase startup-config

reload

-----------------enter config level, start generating public/private keys------------

once it's done reloading and you're back in

enable

config t

crypto key generate

-----------------assign IP to switch-------------------------------

vlan 1 

router-interface ve 1

exit

interface ve 1

ip address 192.168.1.2/24

exit

---------------------------configure authentication, set username/pass, set hostname--------------

aaa authentication web-server default local
aaa authentication enable default local
aaa authentication login default local

hostname whatevernameyouwant

username root password yourpasshere

( If you want no password, do the commands that are in brackets below instead of above command )
( username root nopassword )
( ip ssh permit-empty-passwd yes )

web-management frame bottom

ip ssh key-authentication no


----------------see chassis details, write config changes to memory, show config to verify, reboot-------------

show chassis

write memory

show configuration

exit

once you have gotten the "finished generating ssh keys" message, run the below 

boot system flash primary yes

the switch will reboot, ssh will be available at port 22 and a web interface will be available at normal port 80, 
use the root/pass combo you configured earlier for both