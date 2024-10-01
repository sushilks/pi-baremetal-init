
# Setup Ubuntu ISO for cloud init 
This step create a boot parameter for live image such that it will start to look for cloud init data source "nocloud" at net address "http://169.254.169.254"
The ISO setup can be ommited if the cloud init files are mounted as a usb drive in parallel to the usb drive holding the base image. This is not possible with PI zero as it's only able to create a single USB drive. 

## This is written following the instructions to create a new ISO
## https://www.jimangel.io/posts/automate-ubuntu-22-04-lts-bare-metal/
## 

## setup the packages (Need to run as root)
```
sudo apt install xorriso squashfs-tools python3-debian gpg liblz4-tool python3-pip -y
sudo su 
cd
git clone https://github.com/mwhudson/livefs-editor

cd livefs-editor/

python3 -m pip install .
```
## create a grub file. 
```
cat <<'EOF' | sudo tee /tmp/grub.conf
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
EOF
```
## download the iso image 

```
mkdir /opt/images
cd /opt/images
wget https://cofractal-ewr.mm.fcix.net/ubuntu-releases/24.04.1/ubuntu-24.04.1-live-server-amd64.iso
```
## Create a new iso
## copy command exactly as is, it appends `-modded` to the new filename
```
export ORIG_ISO="/opt/images/ubuntu-24.04.1-live-server-amd64.iso"
export MODDED_ISO="${ORIG_ISO::-4}-modded.iso"
livefs-edit $ORIG_ISO $MODDED_ISO --cp /tmp/grub.cfg new/iso/boot/grub/grub.cfg
```

# Create systemd service for the cloud init server
sushil@piusb1:~/work/cinit $ cat cloud_init_server.sh
DATE=`date '+%Y-%m-%d %H:%M:%S'`
echo "cloud-init-server service started at ${DATE}" | systemd-cat -p info

python -m http.server --directory /opt/cloud-init-data 80 | systemd-cat -p info


