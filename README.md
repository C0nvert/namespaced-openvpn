# Set Up OpenVPN Split-Tunneling with Transmission as your Torrent Client

Tested on *Raspbian Buster* and Ubuntu Server 18.04

*For non-root user, you should prefix  `sudo`  with all commands.*

### 1.  **Installing Dependencies**
```
  apt update
  apt install openvpn iproute2 python dnsutils dnsmasq curl transmission-daemon socat

```
### 2. **Download namespaced-openvpn Script**:
```
    cd /usr/local/sbin
    curl -sLSO https://raw.githubusercontent.com/C0nvert/namespaced-openvpn/master/namespaced-openvpn
    chmod +x namespaced-openvpn
```
### 3. **Setting UP provided OpenVPN config files and create a `auth.txt` wich will contain your VPN Login Data:**
  ```
   mkdir -p ~/.config/openvpn
   cp *.ovpn ~/.config/openvpn/
   sudoedit ~/.config/openvpn/auth.txt
   chmod 400 ~/.config/openvpn/*.ovpn
   chmod 400 ~/.config/openvpn/auth.txt
```
### 4. **Test your Connection:**
```
sudo /usr/local/sbin/namespaced-openvpn --config /home/"$USER"/.config/openvpn/foo.ovpn --writepid /var/run/openvpn-protected-foo-"$USER".pid --log /var/log/openvpn-protected-foo-"$USER".log --daemon
```
Now you should successful created a Namespace with OpenVPN running in it.

**Important: change `foo` to to match your ovpn file.
If you want to name your Namespace different that the default `"protected"` name add `"--namespace <nameOfNamespace>"` as parameter above.**

Check with `sudo ip netns exec protected sudo -u "$USER" --curl ifconfig.me` if your freshly created Namespace successfully connected itself to your VPN.
You should see now a different Public IP Adress from the one you see if you just enter `curl ifconfig.me`

If thats the case continue

### 5. **Create socat script:**
This script will later allow you to access Transmission's WebUI from your local Network. Without this it wouldn't be possible, cause the Namespace is kind of a isolated Container.

 ```
  mkdir /etc/socat
  sudoedit /etc/socat/9091_socat
```
*Insert following Lines:*

    #!/bin/bash
    # /etc/socat/9091_socat
    # Socat-Script Port 9091
    #Don't forget to make this Script executable!!! Use chmod 755

    # TCP Port 9091
    sudo /usr/bin/socat tcp-listen:9091,fork,reuseaddr exec:'ip netns exec protected socat STDIO tcp-connect\:127.0.0.1\:9091',nofork

Now we make this script executable
`chmod 755 /etc/socat/9091_socat`

### 6. **Create Systemd Service to start all Services at boot.**

**Socat Systemd:**

`sudoedit /etc/systemd/system/socat-tcp9091.service`

    #Socat Systemd:
    #/etc/systemd/system/socat-tcp9091.service


    [Unit]
    Description=socat Service 9091
    After=transmission-daemon.service
    BindsTo=transmission-daemon.service

    [Service]
    Type=simple
    User=root
    ExecStart=/etc/socat/9091_socat
    Restart=on-abort

    [Install]
    WantedBy=multi-user.target

**Transmission Systemd:**

```sudoedit /lib/systemd/system/transmission-daemon.service```

    #Transmission Systemd:
    #/lib/systemd/system/transmission-daemon.service

    [Unit]
    Description=Transmission BitTorrent Daemon
    After=navpn.service
    Requires=navpn.service


    [Service]
    User=root
    Type=simple
    ExecStart=/bin/sh -c 'exec /sbin/ip netns exec protected /usr/bin/sudo -u debian-transmission /usr/bin/transmission-daemon -f --log-error --config-dir /var/lib/transmission-daemon/info'
    ExecReload=/bin/kill -s HUP $MAINPID

    [Install]
    WantedBy=multi-user.target

**OpenVPN Namespaced Systemd**

Important: Change in the Line `ExecStart "ch2-udp.ovpn` to your ovpn `file Name` and `location`. Same goes for the `auth.txt` file wich we crated above and will contain your login and password for OpenVPN. 

`sudoedit /etc/systemd/system/navpn.service`

    #OpenVPN Namespaced Systemd
    #/etc/systemd/system/navpn.service

    [Unit]
    Description=Namespaced OpenVPN connection to protected
    Before=systemd-user-sessions.service
    After=network-online.target
    Wants=network-online.target
    Documentation=https://github.com/slingamn/namespaced-openvpn

    [Service]
    Type=notify
    WorkingDirectory=/etc/openvpn
    ExecStart=/usr/local/sbin/namespaced-openvpn --daemon --cd /etc/openvpn --config /home/pi/.config/openvpn/ch2-udp.ovpn --auth-user-pass /home/pi/.config/openvpn/auth.txt --writepid /run/namespaced-openvpn/protected.pid --log /var/log/openvpn-protected-pi.log
    PIDFile=/run/namespaced-openvpn/protected.pid
    KillMode=process
    ExecReload=/bin/kill -HUP $MAINPID
    RestartSec=5s
    Restart=on-failure
    RuntimeDirectory=namespaced-openvpn

    [Install]
    WantedBy=multi-user.target

### 7. **Enable all Systemd Services**

```
systemctl enable socat-tcp9091.service
systemctl enable transmission-daemon.service
systemctl enable navpn.service
```
### 8. **Reboot your Device**

   `reboot now`

### 9. **Test for DNS Leak**


```
apt install qt
cd ~/
wget https://raw.githubusercontent.com/macvk/dnsleaktest/master/dnsleaktest.sh
```
```
chmod +x dnsleaktest.sh
```
```
cd ~/
sudo ip netns exec protected sudo -u "$USER" ./dnsleaktest.sh
```
Add this magnet link into your torrent client: 
```
magnet:?xt=urn:btih:be716e0415044000dc8fc2383819c84525a78a59&dn=checkmyiptorrent+Tracking+Link&tr=http%3A%2F%2F34.204.227.31%2F
```
The Displayed IP should match the IP from the DNS Leak Script


### 10. **Optional: Change WebUI Interface for a Modern one**

```
wget https://github.com/C0nvert/transmission-web-control/raw/master/release/install-tr-control.sh --no-check-certificate
```

    bash install-tr-control.sh

*Choose Number 1.*
After installation finished go to  `http://server_domain_name_or_IP:9091`
If the UI is still the same , reload the Page.

### NOTES:

Custom Scripts are located under /usr/bin
dnsleaktest.sh  Test DNS Leak
tr-webui-installer.sh  Installs the new Transmission WebUI

PureVPN use UDP Port 29212 instead of 53, cause it's been used by PiHole DNS
Problem DNS Resolving not working with UDP need further digging. Possible cause 29212 don't get resolved!

### Useful Commands:

#### Permissions
```
chmod 775 -R /pathToFile
chmod 755 /pathToFile
chmod +x /pathToFile
chown user:group (-R) /pathToFile
```
#### Start OpenVPN
```
sudo openvpn --config /pathtoovpn --config /pathToConfig (e.g override.conf)
```
#### Start Namespaced-OpenVPN
 Namespace default value = protected
 If you want to change it add the parameter --namespace <DESIRED NAMEDPACE>

Example String
```
sudo /usr/local/sbin/namespaced-openvpn --config /home/"$USER"/.config/openvpn/foo.ovpn --writepid /var/run/openvpn-protected-foo-"$USER".pid --log /var/log/openvpn-protected-foo-"$USER".log --daemon
```

### **Useful Links:**

https://github.com/rakshasa/rtorrent/wiki/VPN-with-Traffic-Splitting

https://github.com/slingamn/namespaced-openvpn

https://github.com/ronggang/transmission-web-control/wiki/Linux-Installation

https://github.com/ronggang/transmission-web-control
