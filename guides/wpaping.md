Problem: on Raspberry pi, carrier lost every so often. Domain .local is lost or reconnect, hostname gets a suffix like -2 or -3.

Workaround: Ping WPA every two minutes (source: https://www.raspberrypi.org/forums/viewtopic.php?p=1432916#p1432916)

### File `README`

```
chmod +x wpaping
sudo cp wpaping /usr/local/bin
sudo cp wpaping.service /etc/systemd/system
sudo chmod 644 /etc/systemd/system/wpaping.service
sudo systemctl daemon-reload
sudo systemctl enable wpaping
sudo systemctl start wpaping.service
```

### File `wpaconfig`

```
#!/bin/bash
#
# Loop forever doing wpa_cli SCAN commands
#

sleeptime=120  # number of seconds to sleep. 2 minutes (120 seconds) is a good value

while [ 1 ];
do
    wpa_cli -i wlan0 scan
    sleep $sleeptime
done
```

### File `wpaping.service`

```
[Unit]
Description=WPA Supplicant pinger
Requires=network-online.target

[Service]
ExecStart=/usr/local/bin/wpaping
User=root
StandardInput=null
StandardOutput=null
StandardError=null
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
