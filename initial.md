to work on lan 
go to your linux machine 
go to this file 
sudo nano /etc/network/interfaces

then add this 
bash```
auto eth0
iface eth0 inet static
address 192.168.1.10
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 8.8.8.8 1.1.1.1
```
### then rastern your network interface 

sudo systemctl restart networking
