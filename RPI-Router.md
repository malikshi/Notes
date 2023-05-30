# Refferences 

setup static IP, Setup interfaces lan, wan,https://www.technicallywizardry.com/building-your-own-router-raspberry-pi/
firewall https://rob-ferguson.me/how-to-use-your-rpi-as-a-router/

# Tutorial

Firstly, define ip static of lan and wan, and note mac address.
check your mac address by command `ifconfig`  see line with `ether` will show your mac address. define your mac address.

Interface Name | Mac Address
-------------- | -----------
lan | d0:37:45:fc:2d:5a
wan | dc:a6:32:64:c5:0f

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
auto lan
allow-hotplug lan
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
```sh
sudo systemctl disable dhcpcd
sudo apt install isc-dhcp-server
```
Then edit `/etc/default/isc-dhcp-server`, replacing INTERFACESv4, and commented INTERFACESv6:
```sh
INTERFACESv4="lan"
#INTERFACESv6=""
```
Edit `/etc/dhcp/dhcpd.conf` to first change the domain name and domain name servers, also set lease time
```sh
option domain-name "rpi.local";
option domain-name-servers 1.1.1.1, 8.8.8.8, 9.9.9.9;

default-lease-time 600;
max-lease-time 604800;

ddns-update-style none;

authoritative;
```
Also edit / add the following:
```sh
subnet 192.168.100.0 netmask 255.255.255.0 {
        range 192.168.100.3 192.168.100.254;
        option routers 192.168.100.1;
        option subnet-mask 255.255.255.0;
        option domain-name "rpi.local";
        option domain-name-servers 1.1.1.1, 8.8.8.8, 9.9.9.9;
        default-lease-time 600;
        max-lease-time 604800;
}
host rpi {
        hardware ethernet d0:37:45:fc:2d:5a;
        fixed-address 192.168.100.1;
}
```
## Setup Firewall

we will setting firewall for forwarding for ipv4 and ipv6.
enable IP forwarding IPv4 and IPv6
edit `/etc/sysctl.conf`
```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.default.forwarding = 1
net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.default.forwarding = 1
```
Disable Firewalld
```sh
systemctl stop firewalld
systemctl disable firewalld
```
Install and Set Firewall
```sh
apt install -y iptables iptables-persistent
```
Add Routing between LAN and WAN
```sh
# lan
iptables -t nat -A POSTROUTING -o lan -j MASQUERADE
# wan
iptables -t nat -A POSTROUTING -o wan -j MASQUERADE
# allow connections between lan and wan
iptables -A FORWARD -i lan -o wan -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i wan -o lan -j ACCEPT
```
Check rules:
```sh
iptables -S
```
Saved Permanently
```sh
netfilter-persistent save
```
