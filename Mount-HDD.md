# Script to Auto-Mount HDD/SSD
Script to auto-mount hdd/ssd external in openwrt devices. The script will running at reboot and every 5 minutes checking. Script using devices file instead of devices uuid.
## Create Script
Create file `/usr/bin/auto-mount` just copy paste this
```sh
cat >/usr/bin/auto-mount << EOF
#!/bin/bash

MOUNT_POINT="/mnt/nas"   # Replace with the desired mount point
DEVICE="/dev/sdb" # Replace with the appropriate HDD device

# Check if the device is already mounted
if grep -qs "$DEVICE" /proc/mounts; then
    echo "\$(date): \$DEVICE is already mounted." > /tmp/mount.log
else
    # Mount the device
    mount -t ntfs-3g "\$DEVICE" "\$MOUNT_POINT" -o rw,lazytime,noatime,big_writes
    if [ \$? -eq 0 ]; then
        echo "\$(date): \$DEVICE mounted successfully." > /tmp/mount.log
    else
        echo "\$(date): Failed to mount \$DEVICE." > /tmp/mount.log
    fi
fi
EOF
```

## Setup Scheduled Task
Go to System > Scheduled Task, then modified and add value with this,
```sh
@reboot /usr/bin/auto-mount >/dev/null 2>&1
*/5 * * * * /usr/bin/auto-mount >/dev/null 2>&1
```

Credits:
* https://github.com/frizkyiman/Auto-Mount-HDD-Script-Openwrt