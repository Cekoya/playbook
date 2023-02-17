PiVPN
=====

Instructions for configuring a Raspberry Pi (rPi) as a VPN. It is assumed that you are on Mac OS X.

Preparation
-----

- Download [Raspberry Pi Imager](https://www.raspberrypi.org/downloads/) for easy installation of the Raspbian OS on the rPi.
- Make sure you have a network cable (RJ45) and a free port on your network router.
- If you do not have a fixed IP with your internet provider:
  - Check your router to see if it has a feature for automatically update a DNS service when the external IP changes (some D-Link routers have that option)
  - Or go to `duckdns.org` and set up a domain identifier for your VPN. This will be used to access the VPN from outside your local network. We will later set-up the rPi to automatically update duckdns.org on a regular basis if the IP were to change.

SD Card
-----

1. Start `Raspberry Pi Imager`.
1. Put the SD card in the USB adapter and connect it to your Mac.
1. Select `Raspbian Lite` for the OS. You can also select another version of Raspbian such as `Raspbian` or `Raspbian Full` if you want a graphical user interface (using a mouse/keyboard/monitor or VNC). With Raspbian Lite, you will have to use SSH in the Max OS X Terminal to manage the VPN.
1. Select the SD card.
1. Click on the `Write` button.
1. Disconnect the SD card adapter and reconnect.
1. Enable SSH on the rPi by running this command in a Terminal window:
    ```
    touch /Volumes/boot/ssh
    ```
1. Eject the SD card.

Raspberry Pi
-----

1. Put the SD card in the rPi.
1. Plug the rPi to the network using a RJ45 cable.
1. Connect the power adapter to the rPi to boot it.
1. On your router, look at the list of devices connected and look for a `raspberrypi` entry. Note the IP address that was assigned to the rPi.
1. Ask the router to remember to always assign the same IP to the rPi (reserve). This is kind of required so the routing works after a reboot or power outage.
1. Using SSH, log in as `pi` (default password i `raspberry`):
    ```
    ssh pi@192.168.???.???
    ```
1. Now let's do some configuration. Run `sudo raspi-config` and then:
    1. Change User Password: provide a new password and note it somewhere.
    1. Network Options > Hostname: set the hostname to `pivpn` (or something else, it's up to you).
    1. Localisation Options > Change Timezone: select America > Montreal (or something else more relevant).
1. Select `Finish` and reboot the rPi.
1. Now configure the rPi as a VPN device by following [these instructions](https://pivpn.io/), using the default values and the following when prompted:
    - Use `pi` user to store configuration.
    - Select `WireGuard` instead of `OpenVPN`
    - Accept the default VPN port.
    - Select a `Custom` DNS.
    - Custom DNS: `192.168.?.1, 8.8.8.8`. Replace the question mark with the third number of the rPi IP address you identified earlier. FYI, `8.8.8.8` is the public DNS server from Google.
    - Select `DNS Entry` as the way clients will connect.
    - DNS entry: use the DNS url from Duck DNS (i.e. myvpn.duckdns.org) or the one your router manages if it supports handling a dynamic IP address (see Preparation section).
    - Accept to enable unattended upgrades of security patches.
1. Reboot the rPi when asked.

Note: if the rPi has a kiosk screen, you can [activate the HDMI blanking](https://www.raspberrypi.org/documentation/configuration/screensaver.md).


VPN User profile
----

1. Use this PiVPN command to create profile(s):
    ```
    pivpn add
    ```
1. Share the profile file(s) with your user(s).

IMPORTANT: a profile **should not be shared** by multiple devices. If a user will be using an iPad and a computer to access the VPN, you should create 2 profiles, one for each.

Samba
----

We will now configure the rPi with Samba, a file sharing protocol compatible with Mac OS X and Windows.

SSH to the rPi and run this command:

```
sudo apt-get install samba samba-common-bin
```

If the installer prompts you for a WINS settings modification, choose `Yes`.

Then edit the Samba configuration:

```
sudo nano /etc/samba/smb.conf
```

Add the following lines at the end of the file:

```
[vpnconfigs]
path = /home/pi/configs
writeable=No
create mask=0777
directory mask=0777
public=no
```

Press `Ctrl-X` and `Y` and `Enter`.

Now let's create a Samba user for connecting the the shared folder:

```
sudo smbpasswd -a pi
```

If you want to change the password later, it's the same command but without the `-a` option.

Let's restart the Samba service:

```
sudo systemctl restart smbd
```

Now the shared folder should be visible to Mac OS X and Windows computers. Since the shared folder is not public, you need to connect using the Samba user we just created. On Mac OS X, in the finder, press `Ctrl-K` and type this URL: `smb://pivpn/vpnconfigs`. Then use the `pi` user and the password you typed in earlier.

Internet router
-----

The VPN packets must be routed to the rPi from the router exposed to the internet.

1. Go on your router and configure a new `Port Forwarding Rule`.
1. The rule should route all incoming traffic (TCP, UDP) on port 51821 to the IP of the rPi identified earlier.

DNS Entry Update
-----

To easily connect to the VPN using a domain name instead of an IP address, we need to configure the rPi to monitor the public IP address and update theee DNS entry automatically. This is not required if you have a fixed IP address with your internet provider or your router has a feature for updating the DNS entry and you enabled it.

`DuckDNS.org` will supply the necessary code to install:

1. Go to [duckdns.org](http://duckdns.org) and sign-in.
1. Then go to the [install page](https://www.duckdns.org/install.jsp).
1. Click on the `Linux Cron` button.
1. Select your domain prefix in the list.
1. SSH to the rPi and run these commands:
    ```
    mkdir duckdns
    cd duckdns
    nano duck.sh
    ```
1. You are now in the text editor. Go on the DuckDNS install page from earlier step and locate the code snippet that starts with `echo` and copy the snipper to your clipboard. For example, `echo url="https://www.duckdns.org/update?domains=foobar&token=485857-f7a4-4ab6-9f5b-d49d0916528d&ip=" | curl -k -o ~/duckdns/duck.log -K -`
1. Go back to the nano editor and paste the snippet. Then save and exit by pressing `Ctrl-X` and `Y` and `Enter`.
1. Run this command:
    ```
    chmod 700 duck.sh
    ```
1. Test the update by running the command. You should see some statistics and not an error.
    ```
    ./duck.sh
    ```
1. Now its time to schedule the command to run regularly:
    ```
    crontab -e
    ```
1. If prompted, choose `nano` as the editor.
1. You are now in the nano editor. Add the following line at the end of the file. This tells Cron to run the duck.sh script every 15 minutes:
    ```
    */15 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
    ```
1. Then save and exit by pressing `Ctrl-X` and `Y` and `Enter`.


Client
-----

Client devices must install Wireguard. [Click here](https://www.wireguard.com/install/) for installation instructions.

Once the client is installed, you need to install a VPN profile. If this is on a mobile device, you can use the camera to capture the profile using a qr code. To display the QR core, run the following command on the rPi (connect through SSH first):

```
pivpn -qr profile-name
```

The other way is to share the profile file to the vpn user. Ideally, you should **not send the profile by email** as it is not very secure. A USB key or a shared drive should be used, make sure to delete the file after the profile has been installed in Wireguard.

You can connect to the shared config folder using Samba. Refer the to Samba section above.

Troubleshooting
-----

### PiVPN fails to start when running `pivpn -d`

```
Nov 12 01:37:14 pivpn systemd[1]: Starting WireGuard via wg-quick(8) for wg0...
Nov 12 01:37:15 pivpn wg-quick[558]: [#] ip link add wg0 type wireguard
Nov 12 01:37:15 pivpn wg-quick[558]: Error: Unknown device type.
Nov 12 01:37:15 pivpn wg-quick[558]: Unable to access interface: Protocol not supported
Nov 12 01:37:15 pivpn wg-quick[558]: [#] ip link delete dev wg0
Nov 12 01:37:15 pivpn wg-quick[558]: Cannot find device "wg0"
Nov 12 01:37:15 pivpn systemd[1]: wg-quick@wg0.service: Main process exited, code=exited, status=1/FAILURE
Nov 12 01:37:15 pivpn systemd[1]: wg-quick@wg0.service: Failed with result 'exit-code'.
Nov 12 01:37:15 pivpn systemd[1]: Failed to start WireGuard via wg-quick(8) for wg0.
pi@pivpn:~ $ sudo apt-get update
```

Remove and install again:

```
sudo apt-get remove wireguard-dkms
sudo apt-get install wireguard-dkms
```
