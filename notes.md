sudo apt install dnsmasq
sudo vi /etc/dnsmasq.conf
sudo vi /etc/dnsmasq.d/usb.conf

sushil@piusb1:/etc/network/interfaces.d $ cat /etc/dnsmasq.d/usb.conf
dhcp-range=169.254.169.10,169.254.169.20,1h
dhcp-option=3
dhcp-option=6
interface=usb0

sushil@piusb1:/etc/network/interfaces.d $ cat /etc/network/interfaces.d/usb
audo usb0
allow-hotplug usb0
iface usb0 inet static
address 169.254.169.254
netmask 255.255.255.0

# follow the instructions to create a new ISO
# https://www.jimangel.io/posts/automate-ubuntu-22-04-lts-bare-metal/
# 

apt install xorriso squashfs-tools python3-debian gpg liblz4-tool python3-pip -y

git clone https://github.com/mwhudson/livefs-editor

cd livefs-editor/

python3 -m pip install .

## get grub.conf file
set timeout=5

loadfont unicode

set menu_color_normal=white/black
set menu_color_highlight=black/light-gray

menuentry "Try or Install Ubuntu Server" {
        set gfxpayload=keep
        linux   /casper/vmlinuz  --- autoinstall quit ds=nocloud-net\;s=http://169.254.169.254/
        initrd  /casper/initrd
}
grub_platform
if [ "$grub_platform" = "efi" ]; then
menuentry 'Boot from next volume' {
        exit 1
}
menuentry 'UEFI Firmware Settings' {
        fwsetup
}
else
menuentry 'Test memory' {
        linux16 /boot/memtest86+x64.bin
}
fi

# Create a new iso
# copy command exactly as is, it appends `-modded` to the new filename
export ORIG_ISO="ubuntu-24.04.1-live-server-amd64.iso"
export MODDED_ISO="${ORIG_ISO::-4}-modded.iso"
livefs-edit ../$ORIG_ISO ../$MODDED_ISO --cp /tmp/grub.cfg new/iso/boot/grub/grub.cfg


# Create systemd service for the cloud init server
sushil@piusb1:~/work/cinit $ cat cloud_init_server.sh
DATE=`date '+%Y-%m-%d %H:%M:%S'`
echo "cloud-init-server service started at ${DATE}" | systemd-cat -p info

python -m http.server --directory /opt/cloud-init-data 80 | systemd-cat -p info


# create a systemd service
sushil@piusb1:~/work/cinit $ cat cloud_init_server.service
[Unit]
Description=cloud init server systemd service.

[Service]
Type=simple
ExecStart=/bin/bash /usr/bin/cloud_init_server.sh

[Install]
WantedBy=multi-user.target

# setup systemd
sudo cp cloud_init_server.sh /usr/bin/
sudo chmod +x /usr/bin/cloud_init_server.sh
sudo cp cloud_init_server.service /etc/systemd/system/
sudo chmod 644 /etc/systemd/system/cloud_init_server.service
sudo systemctl enable cloud_init_server