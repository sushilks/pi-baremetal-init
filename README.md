# Goal
This project is intended to show how a pi zero can be used to create a usb stick for simplifying the bara-metal provisioning using cloud init tech. 


# BOM
The project was build using 
- a raspberry pi w 2w  (https://www.raspberrypi.com/products/raspberry-pi-zero-2-w/)
- usb hat (https://a.co/d/6fPu3LR)

![USB PI setup](./images/key.png)


# Setup 
Software setup needed
- Setup the Key to be in gadget mode such that it will project a usb-key and a usb-ethernet device [link](./setup-otg-gadget.md)
- Setup a http server to host the cloud init file on the key [link](./setup-http-server.md)
- Update the standard ubuntu ISO to work with cloud init by passing a boot parameter [link](./setup-ubuntu-iso.md)
# Create a cloud init file
You will need to create few files in "/opt/cloud-init-data" to reflect the cloud init data source. Refer to https://cloud-init.io/
create two empty files 
```
touch /opt/cloud-init-data/meta-data
touch /opt/cloud-init-data/vendor-data
```
Here is a sample user-data file, you will need to modify it to reflect your needs
```
>cat /opt/cloud-init-data/user-data
#cloud-config
runcmd:
  - [eval, 'echo $(cat /proc/cmdline) "autoinstall" > /root/cmdline']
  - [eval, 'mount -n --bind -o ro /root/cmdline /proc/cmdline']
  - [eval, 'snap restart subiquity.subiquity-server']
  - [eval, 'snap restart subiquity.subiquity-service']
autoinstall:
  version: 1
  ssh:
    install-server: true
  identity:
    hostname: ubuntu-server
    password: "$6$exDY1mhS4KUYCE/2$zmn9ToZwTKLhCw.b4/b.ZRTIZM30JZ4QrOQ2aOXJ8yk96xpcCof0kxKwuX1kqLG/ygbJ1f8wxED22bTL4F46P0"
    username: ubuntu
  late-commands:
    # randomly generate the hostname & show the IP at boot
    - echo ubuntu-host-$(openssl rand -hex 3) > /target/etc/hostname
    # dump the IP out at login screen
    - echo "Ubuntu 22.04 LTS \nIP - $(hostname -I)\n" > /target/etc/issue
  user-data:
    disable_root: true
    timezone: America/Los_Angeles
    package_upgrade: false
    users:
      - name: john
        primary_group: users
        groups: sudo
        lock_passwd: true
        # don't need PW since using SSH, leaving this in though...
        # password is "changeme" - created with `docker run -it --rm alpine mkpasswd --method=SHA-512`
        passwd: "$5$IWwNqL9VUSDoc4Jv$DEUGR.cZQcbz/QvdCOmU13fX5ZW0rANg8LqkAtX3nBA"
        shell: /bin/bash
        # use cat ~/.ssh/id_rsa.pub or generate to get your public key
        ssh_authorized_keys:
          - "ssh-rsa AAAAB3NzaC1yc2ELDKINEINFEDqy9pJILJTzxySx7HiQ1eruJK2SgDZqZhbUrq0B3DO9Bzrhe7O//w0F2n2At+ll1TycBwizdO9m/fEc1mMA3AubM8GvwovkuUE53Rjo54mSYRtgCfF38VsTcqYTYSr/rIjIDv/7NO8ZfrMTW0N671mL6uBwcy7FlJolMbdzuM1Po6ngCPiSipSblhXz2WJMnsewbYqww1P6ovXbBQqv58/XwTWXjAfn5zdFt4V53fSOBStwPlr01gajASTa5W2LLkW9OKC14YYsyu9Av8PooXWEAIifmiA9HXxmEV+nmoJ75aAroiihJoniJ9MBC94cz4Ea7oPZ6MwFif39PgJt+5V john@john.local"
        sudo: ALL=(ALL) NOPASSWD:ALL
    # shutdown after first host initial provisioning
    power_state:
      mode: reboot
```


# Usages
Once the setup is done the device will boot and get ready with the image and cloud init automatically and will not need any login/post boot interaction. 

Simply plug in the stick into a computer that is not provisioned to get it provisioned with standard ubuntu image. 
You will need to modify the user-data file to your requirements, details on cloud init config can be found at https://cloud-init.io/

The node if not provisioned will need to be setup to boot from usb for this to work. 
Once provisioned the node may not boot from usb so this will be one time workflow only for first time provisioning. (Its possible to keep the usb boot in the loop for every boot but that requires more work)

The ISO updates are needed as the default ubuntu boot only looks for a usb mounted cloud init file and to enable a net nocloud data source we need to pass boot parameters. PI only allows for a single usb mass storage device so I had to use the net mounted cloud init. If in future there is a way to create two usb devices, it will be possible to use the vanilla iso image form ubuntu to boot. 

# Next Step
The project goal was to show how the key can be used to image a bare-metal box. 
It's possible to further build this out such that the pi can be used to interact with a cloud control plane to get both the iso and the cloud init files, this will open up the possibility for using this to provision large number of nodes with ease. 