# Installation
1. Install the luci-app-samba4 package in LuCI. Any dependencies, such as samba4-server, are installed automatically.

   - Alternatively install via SSH: `opkg update && opkg install luci-app-samba4 samba4-server ntfs-3g wsdd2`
   - Optional check available version using `opkg list | grep -i samba`

2. Configure Samba in LuCI on the Services → Network Shares page. It is recommended that you use LuCI for the initial configuration and only edit /etc/samba/smb.conf.template if needed via LuCI Edit Template tab or from the shell. Basic LuCI configuration guidance is provided below:

```
Interface: lan
Workgroup: WORKGROUP
Enable Extra Tuning: checked (for more throughput. Note for Apple Time Machine do not check as it is incompatible with macOS)
Enable macOS compatible shares: checked
Allow legacy (insecure) protocols/authentication: checked
```

# Template
Edit and modify `/etc/samba/smb.conf.template`
+ uncommented `passdb backend = smbpasswd`
+ uncommented `smb passwd file = /etc/samba/smbpasswd`
+ modify to `null passwords = yes`
+ add line `local master = yes`
+ add line `preferred master = yes`

# Mounting SSD/Storages
* SSD/Storages is using format NTFS.

We will mount SSD or storages to this path `/root/Family`, check path of SSD in Systems > Mount Points > Mount Points, e.g devices path `/dev/sdb` so we need mount points the devices to `/root/Family`. Add this below to `/etc/rc.local`:
```sh
sleep 1
ntfs-3g /dev/sdb /root/Family -o rw,lazytime,noatime,big_writes
smbd -D
nmbd -D
```
* Create Folders

we will create folder and set permission to the folder. Login to OpenWRT terminal
```sh
cd /root
mkdir /root/Family
```
Set Permission to folder to allow users write and read.
```sh
chmod -R 777 /root/Family
chown -R nobody /root/Family
```
Reboot OpenWRT Devices before continues next steps.
# Management Samba Users
Modify the `/etc/passwd` and `/etc/group` files to add the users and groups, you can follow these steps on OpenWrt:

1. SSH into your OpenWrt router using an SSH client.

2. Open the `/etc/group` file with a text editor:

   ```bash
   nano /etc/group
   ```

3. Add the following line at the end of the file to create the "family" group:

   ```
   family:x:1002:
   ```

   Save and exit the file.

4. Open the `/etc/passwd` file with a text editor:

   ```bash
   vi /etc/passwd
   ```

5. Add the following lines at the end of the file to create the user accounts for Father, Mother, Adult, and Kids with their respective settings:

   ```
   Father:x:1003:1002::/dev/null:/bin/false
   Mother:x:1004:1002::/dev/null:/bin/false
   Adult:x:1005:1002::/dev/null:/bin/false
   Kids:x:1006:1002::/dev/null:/bin/false
   ```

   Save and exit the file.

6. If you are using Samba on your OpenWrt router and want to set Samba passwords for these users, you can use the "smbpasswd" command. Run the following command for each user and provide a password when prompted:

   ```bash
   smbpasswd -a Father
   smbpasswd -a Mother
   smbpasswd -a Adult
   smbpasswd -a Kids
   ```

   This will create Samba passwords for the users and store them in the Samba password file.

By following these steps, you should have directly added user accounts for Father, Mother, Adult, and Kids to the `/etc/passwd` file, assigned them to the "family" group in the `/etc/group` file, set their passwords, and configured their home directory and login shell settings.

# Setting Shared Directories
* Creates each folders for each users and family folder for all family members

Login to terminal OpenWRT
```sh
mkdir -p /root/Family/SMB-Family
mkdir -p /root/Family/SMB-Personal-Data-Father
mkdir -p /root/Family/SMB-Personal-Data-Mother
mkdir -p /root/Family/SMB-Personal-Data-Adult
mkdir -p /root/Family/SMB-Personal-Data-Kids
```
* Edit Samba4 configuration throuhg terminal `/etc/config/samba4`

```conf
config sambashare
        option read_only 'no'
        option guest_ok 'no'
        option create_mask '0666'
        option dir_mask '0777'
        option timemachine '1'
        option name 'SMB-Family'
        option path '/root/Family/SMB-Family'

config sambashare
        option name 'SMB-Personal-Data-Father'
        option path '/root/Family/SMB-Personal-Data-Father'
        option read_only 'no'
        option create_mask '0666'
        option dir_mask '0777'
        option users 'Father'
        option guest_ok 'no'
        option timemachine '1'

config sambashare
        option name 'SMB-Personal-Data-Mother'
        option path '/root/Family/SMB-Personal-Data-Mother'
        option read_only 'no'
        option create_mask '0666'
        option dir_mask '0777'
        option users 'Mother'
        option guest_ok 'no'
        option timemachine '1'

config sambashare
        option name 'SMB-Personal-Data-Adult'
        option path '/root/Family/SMB-Personal-Data-Adult'
        option read_only 'no'
        option create_mask '0666'
        option dir_mask '0777'
        option users 'Adult'
        option guest_ok 'no'
        option timemachine '1'

config sambashare
        option name 'SMB-Personal-Data-Kids'
        option path '/root/Family/SMB-Personal-Data-Kids'
        option read_only 'no'
        option users 'Kids'
        option guest_ok 'no'
        option create_mask '0666'
        option dir_mask '0777'
        option timemachine '1'
```
Save with using command `uci commit samba4`

Restart Samba4 service
```sh
/etc/init.d/samba restart
```
You will now be able to read/write network shares on your LAN similar to Network-attached_storage. For example browsing a share named 'storage' on your router default IP using Windows File Explorer: \\192.168.1.1\SMB-Family\.
Windows, most Linux distributions, and macOS include SMB support in their File Browsers. Android (like OpenWrt is also Linux based) is supported and shares can be browsed in free apps like X-plore, or VLC or Kodi for media playback. If your OS is missing support, simply install some client software.


# Firewall Samba

* Throughput

Since netfilter tracks every connection, it may improve throughput to disable conntrack for Samba connections if you use NAT.

```sh
uci -q delete firewall.samba_nsds_nt
uci set firewall.samba_nsds_nt="rule"
uci set firewall.samba_nsds_nt.name="NoTrack-Samba/NS/DS"
uci set firewall.samba_nsds_nt.src="lan"
uci set firewall.samba_nsds_nt.dest="lan"
uci set firewall.samba_nsds_nt.dest_port="137-138"
uci set firewall.samba_nsds_nt.proto="udp"
uci set firewall.samba_nsds_nt.target="NOTRACK"
uci -q delete firewall.samba_ss_nt
uci set firewall.samba_ss_nt="rule"
uci set firewall.samba_ss_nt.name="NoTrack-Samba/SS"
uci set firewall.samba_ss_nt.src="lan"
uci set firewall.samba_ss_nt.dest="lan"
uci set firewall.samba_ss_nt.dest_port="139"
uci set firewall.samba_ss_nt.proto="tcp"
uci set firewall.samba_ss_nt.target="NOTRACK"
uci -q delete firewall.samba_smb_nt
uci set firewall.samba_smb_nt="rule"
uci set firewall.samba_smb_nt.name="NoTrack-Samba/SMB"
uci set firewall.samba_smb_nt.src="lan"
uci set firewall.samba_smb_nt.dest="lan"
uci set firewall.samba_smb_nt.dest_port="445"
uci set firewall.samba_smb_nt.proto="tcp"
uci set firewall.samba_smb_nt.target="NOTRACK"
uci commit firewall
/etc/init.d/firewall restart
```
* Remote Access

For remote access configure your firewall as per below. See port explanation. Use caution here, as you may eventually expose your network to security concerns. Samba and many other packages are not always updated to the latest CVEs between releases. This is not needed for LAN access to your shares, file sharing such as SMB and NAS are typically best used for LAN access for this reason.

```sh
uci -q delete firewall.samba_nsds
uci set firewall.samba_nsds="rule"
uci set firewall.samba_nsds.name="Allow-Samba/NS/DS"
uci set firewall.samba_nsds.src="lan"
uci set firewall.samba_nsds.dest_port="137-138"
uci set firewall.samba_nsds.proto="udp"
uci set firewall.samba_nsds.target="ACCEPT"
uci -q delete firewall.samba_ss
uci set firewall.samba_ss="rule"
uci set firewall.samba_ss.name="Allow-Samba/SS"
uci set firewall.samba_ss.src="lan"
uci set firewall.samba_ss.dest_port="139"
uci set firewall.samba_ss.proto="tcp"
uci set firewall.samba_ss.target="ACCEPT"
uci -q delete firewall.samba_smb
uci set firewall.samba_smb="rule"
uci set firewall.samba_smb.name="Allow-Samba/SMB"
uci set firewall.samba_smb.src="lan"
uci set firewall.samba_smb.dest_port="445"
uci set firewall.samba_smb.proto="tcp"
uci set firewall.samba_smb.target="ACCEPT"
uci commit firewall
/etc/init.d/firewall restart
```