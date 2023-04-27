# Refferences 
setup static IP, Setup interfaces lan, wan,https://www.technicallywizardry.com/building-your-own-router-raspberry-pi/
firewall https://rob-ferguson.me/how-to-use-your-rpi-as-a-router/

# Tutorial
Firstly, define ip static of lan and wan, and note mac address.
lan mac address d0:37:45:fc:2d:5a
wan mac address dc:a6:32:64:c5:0f
no need to setup wlan0 since we are using wan with UTL(USB to LAN).

## Set Interfaces Name
Edit `/etc/udev/rules.d/10-network.rules` add interfaces name and mac address into it with format;
```sh
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="d0:37:45:fc:2d:5a", NAME="lan"
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="dc:a6:32:64:c5:0f", NAME="wan"
```
also setting iftab for naming interfaces name in `/etc/iftab`
```sh
lan mac d0:37:45:fc:2d:5a
wan mac dc:a6:32:64:c5:0f
```
## Set-Up interfaces name in network
Add new file for every interfaces we want to creates,
LAN interface
`/etc/network/interfaces.d/lan` fill with,
```sh
allow-hotplug lan
#iface lan inet6 static
#    address 2a01:7c8:d001:61::1
#    netmask 48
#    gateway 2a01:7c8:d001::1
iface lan inet static
        address 192.168.100.1
        netmask 255.255.255.0
        gateway 192.168.100.1
```

WAN interface
`/etc/network/interfaces.d/wan` fill with,
```sh
auto wan
allow-hotplug wan
iface wan inet dhcp
```
## Setup DCHP SERVER
We are plan to use isc-dhcp-server instead of dhcpcd also didn't setup for ipv6
Disable dhcpcd and install isc-dhcp-server
``sh
sudo systemctl disable dhcpcd
sudo apt install isc-dhcp-server
```
Then edit `/etc/default/isc-dhcp-server`, replacing INTERFACESv4:
```sh
INTERFACESv4="lan"
```
Edit `/etc/dhcp/dhcpd.conf` to first change the domain name and domain name servers, also set lease time
```sh
option domain-name "rpi.local";
option domain-name-servers 1.1.1.1, 8.8.8.8, 9.9.9.9;

default-lease-time 600;
max-lease-time 604800;
```
Also add add the following:
```sh
authoritative;
subnet 192.168.100.0 netmask 255.255.255.0 {
range 192.168.100.3 192.168.100.254;
option routers 192.168.100.1;
option subnet-mask 255.255.255.0;
}
host rpi {
        hardware ethernet d0:37:45:fc:2d:5a;
        fixed-address 192.168.100.1;
}
```
## Setup Firewall
we will setting firewall for forwarding for ipv4.
