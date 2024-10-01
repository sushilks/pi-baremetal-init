# setup needed on Raspberry PI to create a OTG Gadget for mass storage and ethernet.
## setup the pi such that it can connect to local wifi and is on network following standard pi setup instructions
## these instructions are with the following os version
```
> cat /etc/os-release
PRETTY_NAME="Raspbian GNU/Linux 11 (bullseye)"
NAME="Raspbian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=raspbian
ID_LIKE=debian
HOME_URL="http://www.raspbian.org/"
SUPPORT_URL="http://www.raspbian.org/RaspbianForums"
BUG_REPORT_URL="http://www.raspbian.org/RaspbianBugs"
```
## setup the gadget init file 
```
cat <<'EOF' | sudo tee /usr/bin/mount_otg_gadgets.sh
#!/bin/bash
modprobe libcomposite
mkdir -p /sys/kernel/config/usb_gadget/g2
cd /sys/kernel/config/usb_gadget/g2
echo 0x1d6b > idVendor # Linux Foundation
echo 0x0104 > idProduct # Multifunction composite gadget
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB
mkdir -p strings/0x409
echo "1234567890" > strings/0x409/serialnumber
echo "sushil" > strings/0x409/manufacturer
echo "PI USB Device" > strings/0x409/product
mkdir -p configs/c.1/strings/0x409
echo "Config 1: Mass Storage and Network" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower
mkdir -p functions/mass_storage.usb0
echo 0 > functions/mass_storage.usb0/lun.0/cdrom
echo 0 > functions/mass_storage.usb0/lun.0/ro
echo /opt/images/ubuntu-24.04.1-live-server-amd64-modded.iso > functions/mass_storage.usb0/lun.0/file
mkdir -p functions/ecm.usb0
HOST="00:dc:c8:f7:75:10" # "HostPC"
SELF="00:dd:dc:eb:6d:a9"
echo $HOST > functions/ecm.usb0/host_addr
echo $SELF > functions/ecm.usb0/dev_addr
ln -s functions/mass_storage.usb0 configs/c.1
ln -s functions/ecm.usb0 configs/c.1
ls /sys/class/udc > UDC
EOF
chmod +x /usr/bin/mount_otg_gadgets.sh
```
## create a service to run the gadget at boot
```
cat <<'EOF' | sudo tee /etc/systemd/system/otg_gadget.service
[Unit]
Description=otg gadget setup systemd service.

[Service]
Type=oneshot
ExecStart=/bin/bash /usr/bin/mount_otg_gadgets.sh


[Install]
WantedBy=multi-user.target
EOF
sudo chmod 644 /etc/systemd/system/otg_gadget.service
```
## activate the service to run at boot
```
sudo systemctl enable otg_gadget
```

## install dnsmasq to create a dhcp service for otg ethernet port.
```
sudo apt install dnsmasq
```

## uncomment dnsmasq to parse config from the conf directory

edit file /etc/dnsmasq.conf and uncomment the following lines
```
# Include all files in a directory which end in .conf
conf-dir=/etc/dnsmasq.d/,*.conf 
```
## create the dnsmasq config for gadget ethernet
``` 
cat <<'EOF' | sudo tee /etc/dnsmasq.d/usb.conf
dhcp-range=169.254.169.10,169.254.169.20,1h
dhcp-option=3
dhcp-option=6
interface=usb0
EOF
```
## create usb interface config
```
cat <<'EOF' | sudo tee /etc/network/interfaces.d/usb
audo usb0
allow-hotplug usb0
iface usb0 inet static
address 169.254.169.254
netmask 255.255.255.0
EOF
```

