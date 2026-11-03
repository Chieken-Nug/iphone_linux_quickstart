# iphone_linux_quickstart
Some guy on reddit requested this so here it is. 


A guide for iPhone Linux


This assumes you are in DFU. Run Palera1n, yada yada look up their docs if you have trouble with the payload. Sometimes Linux is better then Mac for this. Depends on your hardware. 

Once you are in pongOS:

1. Compile the Hoolock Kernel and get the device tree. (see the HoolockLinux Github) (Do this ahead of time if you want to save time, make sure to use correct flags)
* Download the Hoolock initramfs (for ramdisk)
* put iphone in DFU and Enter PongoOS using palera1n (use v2.4)
* Follow iPhomarcy on Github and follow their dual_boot.md and partion your disk correctly. 




Once you are in the serial shell for the ramdisk, you need to set some things up. 

To communicate using usb ethernet you will need to paste in this gadget script. 

mkdir -p /mnt/usr/local/sbin
cat > /mnt/usr/local/sbin/hl-usb-gadget.sh << 'EOF'
#!/bin/sh
set -e
G=/sys/kernel/config/usb_gadget/g1
mountpoint -q /sys/kernel/config || mount -t configfs none /sys/kernel/config
[ -d "$G" ] && exit 0
mkdir -p "$G"
echo 0x05ac > "$G/idVendor"
echo 0x4142 > "$G/idProduct"
mkdir -p "$G/strings/0x409"
echo "HoolockLinux" > "$G/strings/0x409/manufacturer"
echo "USB Device" > "$G/strings/0x409/product"
echo "hoolock0001" > "$G/strings/0x409/serialnumber"
mkdir -p "$G/configs/c.1/strings/0x409"
echo "ncm+acm" > "$G/configs/c.1/strings/0x409/configuration"
mkdir -p "$G/functions/ncm.usb0" "$G/functions/acm.usb0"
ln -s "$G/functions/ncm.usb0" "$G/configs/c.1/"
ln -s "$G/functions/acm.usb0" "$G/configs/c.1/"
echo "$(ls /sys/class/udc | head -n1)" > "$G/UDC"
EOF
chmod +x /mnt/usr/local/sbin/hl-usb-gadget.sh
cat /mnt/usr/local/sbin/hl-usb-gadget.sh

Along with a service to run it: 



[Unit]
Description=USB gadget (NCM network + ACM serial)
After=sys-kernel-config.mount
Before=network-pre.target systemd-networkd.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/sbin/hl-usb-gadget.sh

[Install]
WantedBy=multi-user.target


And to get the network config up, paste this in:

cat > /mnt/etc/systemd/network/10-usb0.network << 'EOF'
[Match]
Name=usb* enx*

[Network]
Address=172.16.42.1/16
DHCPServer=yes

[DHCPServer]
PoolOffset=10754
PoolSize=1
EmitRouter=no
EmitDNS=no
EOF
cat /mnt/etc/systemd/network/10-usb0.network


With these scripts you need to enter chroot with them, and obvioulsy enable them. 
This is the basis for how you communicate and ssh into the phone later. 

Once this is done, with your own kernel that you built, assemble the blob on your host machine.
If you want to double check if the services are enabled run:

chroot /mnt /bin/bash -c "systemctl is-enabled hl-usb-gadget.service systemd-networkd.service sshd.service serial-getty@ttyGS0.service"

If this is good and your partiosn all check out and are ready you are ready for the final boot. Force restart your phone and reenter DFU
Then send the new blob now, the oen for nvme boot “m1n1-linux-nvme2.bin”

give the host machine an adress and try and connect and ssh in. 
e.g. ssh alarm@172.16.42.1

Once you are logged in (password shoudl be root by default) 

sudo resize2fs /dev/nvme0n1p2
df -h /

Grow your image from whatever size it was to fill whatever partion you allocated to linux when using iPhomarchy’s dual_boot.md from github.


And boom, your very own linux install on your iphone.

If you every get untethered, just run the exploit again, e.g. 

   sudo PALERA1N_BYPASS_PASSCODE_CHECK=1 palera1n -lp -k ~/hoolock/docs/binaries/Pongo.bin

And once you boot into PongoOS, you might type smh like this example:

   printf "/send $HOME/hoolock/m1n1-linux-nvme2.bin\nbootm\n" | sudo pongoterm

