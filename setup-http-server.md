# Setup a http server to start at boot and serve the cloud init file 
This server should be accessed on the usb0 interface 

## create a script to run cloud init http server
```
cat <<'EOF' | sudo tee /usr/bin/cloud_init_server.sh
sushil@piusb1:~/work/cinit $ cat cloud_init_server.sh
DATE=`date '+%Y-%m-%d %H:%M:%S'`
echo "cloud-init-server service started at ${DATE}" | systemd-cat -p info

python -m http.server --directory /opt/cloud-init-data 80 | systemd-cat -p info
EOF
chmod +x /usr/bin/cloud_init_server.sh
```

## cretate a systemd service to start http server for cloud init at boot
```
cat <<'EOF' | sudo tee /etc/systemd/system/cloud_init_server.service
[Unit]
Description=cloud init server systemd service.

[Service]
Type=simple
ExecStart=/bin/bash /usr/bin/cloud_init_server.sh

[Install]
WantedBy=multi-user.target
EOF
sudo chmod 644 /etc/systemd/system/cloud_init_server.service
```

## activate the cloud service to run at boot
```
sudo systemctl enable cloud_init_server
```

