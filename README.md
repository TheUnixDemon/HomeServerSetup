# Setup - MyHomePi

Here I explain how I setup my own server for my private usage at home. I use the RaspberryPi 4B with 8GB memory and a external hard driver as a storage. As an operating system I use raspian because it's used to be the stablest ones. To be exact here I use `Raspberry Pi OS Lite (64-bit)`.

The reason for that kind of *project* is to have a own seperated router where everything can be hosted and you have 100% control about anything like content filtering, hosting a own VPN, configuring a additional firewall or basicly anything that doesn't require a ton of processor power. But basicly everything here can be used on similar Debian based x86 systems if needed. But I like to refer to my setup. Also I like to keep my 

# Installation Guide

## Flashing Storage & SSH PUB-Key

Install via [`Raspberry Pi Imager`](https://www.raspberrypi.com/software/) or other flashing software to install `Raspberry Pi OS Lite (64-bit)` on the storage of your choice. Also keep in mind to enable SSH and maybe install already the PUB-Key for the authentication process.

If you want to generate the keys for the PUB-Key authentication. Here a command for that. I would recommend to checkout [ubuntuusers](https://wiki.ubuntuusers.de/SSH/) for a better understanding how SSH and `ssh-keygen` works.

```bash
#!/bin/bash
ssh-keygen -t rsa -b 4096
```

The keys will be saved in the folger `~/.ssh`. You need to put the content of the `*.pub` in the `authorized_keys`. Also you can put that public key manually there. For that copie in the `/home/PIUSER/.ssh/authorized_keys`. Maybe you need to create the `.ssh` folder beforewards.

## After Installation

After installation of Raspian with SSH access the first step would be to update the system and reboot afterwards.

```bash
#!/bin/bash
# full-upgrade is newer than dist-upgrade but does the same
sudo apt update && sudo apt full-upgrade
sudo reboot
```

## Static IP-Adress

For that we don't use `dhcpcd` or `NetworkManager`. You could use `dhcpcd` if you don't want to use your RaspberryRouter for LAN connections but only as AccessPoint. For that you need to install `dhcpcd5`. Basicly I had trouble to setup virtual interfaces with `dhcpcd5` but maybe there is some work around for that.

But I use for configure a static IP-Adresses the service called `networking`. For that install `ifupdown` than you are good to go with the installation part.

```bash
#!/bin/bash
sudo apt-get install ifupdown
sudo rfkill unblock wifi # remove soft lock of wlan interface
```

For the configuration of this service you can add a onfiguration file to `/etc/network/interfaces.d/YOUR-CONF` but I'd like to use the basic config file `/etc/network/interfaces`. Here a template for that. And for more information I like to refer to [ubuntuusers](https://wiki.ubuntuusers.de/interfaces/).

```bash
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# your normal network; gateway of your other networks/interfaces
auto eth0
iface eth0 inet static
    address 192.168.0.40/24
    gateway 192.168.0.1
    network 192.168.0.0
    broadcast 192.168.0.255
#    dns-nameservers 127.0.0.1 # own DNS; for later

# gateway of manually configured clients; virtual ethernet interface
auto eth0:1
iface eth0:1 inet static
    address 10.140.25.1/24
    network 10.140.25.0
    broadcast 10.140.25.255
#    dns-nameservers 127.0.0.1 ::1

# accesspoint interface
auto wlan0
iface wlan0 inet static
    address 10.140.20.1/24
    network 10.140.20.0
    broatcast 10.140.20.255
#    dns-nameservers 127.0.0.1 ::1
```

Now the config is complete you should **enable** the service and **disable** the other ones that want to configure the interfaces.

```bash
#!/bin/bash
sudo systemctl disable NetworkManager systemd-networkd
sudo systemctl enable --now networking
sudo systemctl status networking
```

If you checked the `networking` service and it says something like that the interfaces are already configured that is caused by `NetworkManager` and `systemd-networkd`. So you need to restart to let `networking` configure the interfaces for you needs.

```bash
#!/bin/bash
sudo reboot
```

So to check if everything runs fine you can check that with followed commands.

```bash
#!/bin/bash
ifconfig # or 'ip -br a'
```

## PiHole

I would also recommend to use **PiHole** or alternatives like **AdGuard  Home**. A huge part of this guide is possibly also usable for **AdGuard Home** but I refer to **PiHole**.

To install PiHole you can use this command or go to this [https://pi-hole.net/](https://github.com/pi-hole/pi-hole/#one-step-automated-install) to get more informations.

```bash
#!/bin/bash
curl -sSL https://install.pi-hole.net | bash
```

After you followed throught the guide you should have a running website to handle PiHole.

### Blocklists

I recommend a very famous blocklist source [here](https://github.com/hagezi/dns-blocklists). I tested these here.

* [**Multi pro**](https://github.com/hagezi/dns-blocklists?tab=readme-ov-file#pro)
* [**Pop-Up Ads**](https://github.com/hagezi/dns-blocklists?tab=readme-ov-file#popupads)
* [**Threat Intelligence Feeds**](https://github.com/hagezi/dns-blocklists?tab=readme-ov-file#tif)

Afterwards you added your blocklists of your choice you have to get to `Tools -> Update Gravity` section within the web ui and click on `update`. You can do that over the terimal as alternative if you prefer that.

### Unbound

PiHole has a guide for Unbound as a recursive DNS. It's well documented and works correctly if you want to setup that. Also lookup here for checkups for your configurations.

I like to have followed features if I make a request to a DNS.
* **DNSSEC** - validate response of DNS server
* **DNS-Over-TLS (DOT)** - encrypted requests to the DNS server

The problem is PiHole supported only **DNSSEC** so I build my own DNS request forwarder with the help of **Unbound**. Unbound is basicly a service that makes the same thing as every DNS server does. It makes requests to the **root nameserver**. The problem with that is that DNS servers and Unbound not encrypt the requests that they make.

But you can also use Unbound to use Unbound normally to make DNS requests but also encrypted (DOT).

```bash
#!/bin/bash
sudo apt-get install unbound
```

Now that Unbound installed you need to configure it.

```bash
#!/bin/bash
sudo nano /etc/unbound/unbound.conf.d/NAMECONFIG.conf
```

Here an example how I make this configuration for Unbound.
```bash
server:
    # logfile: "/var/log/unbound/unbound.log"
    verbosity: 0

    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: yes

    prefer-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m

    tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"

    private-address: 192.168.0.0/24
    private-address: 10.140.20.0/24
    private-address: 10.140.25.0/24

forward-zone:
    name: .
    forward-first: no # disables fallback to recursive dns behavior
    forward-tls-upstream: yes
    # uses Quad9; can be replaced with other configs
    forward-addr: 9.9.9.9@853#dns.quad9.net
    forward-addr: 149.112.112.112@853#dns.quad9.net
    forward-addr: 2620:fe::fe@853#dns.quad9.net
    forward-addr: 2620:fe::9@853#dns.quad9.net
```

The port is changed from the default port 53 to 5335 as the PiHole documentation says to do because of the port conflict that would be the cause if you don't do that. Because PiHole runs already under port 53.

```bash
#!/bin/bash
sudo systemctl enable --now unbound # if not done already
sudo systemctl restart unbound

# checks connection with DNS
dig pi-hole.net @127.0.0.1 -p 5335 # expected status: NOERROR
dig fail01.dnssec.works @127.0.0.1 -p 5335 # expected status: SERVFAIL
dig dnssec.works @127.0.0.1 -p 5335 # expected status: NOERROR
```

And now we are finished with the configuration of the Unbound DNS request forwarder. So you can change the DNS reference within PiHole to `127.0.0.1#5335` within the section `Settings -> DNS`.

### Last Steps

Now the Unbound setup and the reference to it within PiHole is made you should change the config of `networking` so that your Rasberry calls only to the locally DNS setup (PiHole) and not to your standard router.

```bash
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# your normal network; gateway of your other networks/interfaces
auto eth0
iface eth0 inet static
    ...
    dns-nameservers 127.0.0.1 # own DNS configuration

# gateway of manually configured clients; virtual ethernet interface
auto eth0:1
iface eth0:1 inet static
    ...
    dns-nameservers 127.0.0.1 ::1

# accesspoint interface
auto wlan0
iface wlan0 inet static
    ...
    dns-nameservers 127.0.0.1 ::1
```

After that you should restart your server to load up every setting and newly made configuration correcly.

## Forwarding Subnet Packages

First you have to **enable** that IPv4 packages have the right to be forwarded across another interface. Add/Uncomment for that a line within `/etc/sysctl.conf`.

```bash
#!/bin/bash
sudo nano "/etc/sysctl.conf"
```

The line I told about. You can add or uncomment the containing one.

```bash
net.ipv4.ip_forward=1
```

Now add the following forwarding configurations with **iptables**.

```bash
#!/bin/bash
sudo iptables -t nat -A POSTROUTING -o eth0 -s 10.140.20.0/24 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -o eth0 -s 10.140.25.0/24 -j MASQUERADE
```

This configuration is only temporary so you can put your configuration into the `/etc/iptables.ipv4.nat` files. Or you install a package that makes exactly that for you. Make the temporary config persistent.

For that you can install following packages.

```bash
#!/bin/bash
sudo apt-get install iptables-persistent netfilter-persistent
sudo systemctl enable --now netfilter-persistent.service # only to be sure that it runs correctly
sudo netfilter-persistent save # makes it persistent
```

## Hostapd - AccessPoint

To make a AccessPoint via **wlan0** interface you can use the DHCP configuration within the PiHole web ui. PiHole will be reployed and installed with **dnsmasq** so you don't have to setup the DHCP for your wlan0 interface manually.

You have to install Hostapd for that.
```bash
#!/bin/bash
sudo apt-get install hostapd
# creating future hostapd config
sudo touch /etc/hostapd/hostapd.conf
sudo nano /etc/default/hostapd # needs to refer to config path
```

Now that you have the file `/etc/default/hostapd` open you should change/add this line to the configurations.

```bash
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

Let us configure the AccessPoint (hostapd). For that open the file `/etc/hostapd/hostapd.conf` with the editor your choice. Here an example for a 5GHz AccessPoint.

```bash
interface=wlan0
driver=nl80211

ssid=NAME_NETWORK
country_code=DE

hw_mode=a
channel=36
ieee80211d=1
ieee80211n=1
ieee80211ac=1
wmm_enabled=1

wpa=2
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
auth_algs=1
wpa_passphrase=PASSWORD
```

And now we have to **enable** and **start** the service. Then check if the service is running correctly.

```bash
#!/bin/bash
sudo systemctl unmask hostapd
sudo systemctl enable --now hostapd
sudo systemctl status hostapd
```

But if the current service is already running you should just restart it and checkout the status of this service using **systemctl**.

Afterwards you can add your DHCP IP-Adress section that you want to be automaticly assigned to the clients. For that use the IP-Adress section that is within the subnet of **wlan0**. Then you should restart your RaspberryPi and checkout if you can connect with your clients and get access to the internet. 

### Forwarding Error

If not maybe checkout if the forwarding (common problem) is the problem. You can test that if you make this temporary and reconnect to your AccessPoint.

```bash
#!/bin/bash
sudo iptables -t nat -A POSTROUTING -o eth0 -s 10.140.20.0/24 -j MASQUERADE
```

If that's the problem you could write down a daemon (`/etc/systemd/system/YOURDAEMON.service`) that makes this every time you restart your server. But that would be not the greatest solution for that kind of problem. Here an example for that.

Here an example for a service like that.

```bash
[Unit]
Description=assignes interfaces to forward
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash PATH.sh
RemainAfterExit=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Naturally you have to write the `PATH.sh`. Here also an example for that.

```bash
#!/bin/bash
sudo iptables -t nat -A POSTROUTING -o eth0 -s 10.140.20.0/24 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -o eth0 -s 10.140.25.0/24 -j MASQUERADE
```

Now you can **enable** your own service.

```bash
#!/bin/bash
sudo systemctl enable --now YOURDAEMON.service
sudo systemctl status YOURDAEMON.service
```

And that's it. Now it should work out fine even without the convenience of the tools that does that.

So a final restart and checkout and we are finished with the AccessPoint, static adresses and a DNS blocklist (with encryption via Unbound).

## PIVPN - WireGuard

I used PIVPN with WireGuard to make access from outside my private network possible. So I want to mention that here.

Oneshot line to install [PIVPN](https://www.pivpn.io/). I recommend to enable automatic security updates and refer to PiHole as DNS. The rest of it is pretty much easy going.

```bash
#!/bin/bash
curl -L https://install.pivpn.io | bash
```

You only have to go through everything that PIVPN mentioned. After that the VPN is ready to use. But don't forget to make a rule on your default router that the port that you've choosen is open and uses **udp**.

## DDNS Client

If you want to get access from outside you have probably a dynamic ip adress that you get from you ISP. So you have to get a DDNS that refers to your ip. Basicly your server (RaspberryPi) makes a call intervallwise to the DDNS host so that the domain gets updated and refers to your current ip adress.

Here I show how it's done with the service `ddclient` and **no-ip.com** since **no-ip.com** is free to use. `ddclient` can use every host of DDNS adresses.

```bash
#!/bin/bash
sudo apt-get install ddclient
sudo nano /etc/ddclient.conf
```

Here the configuration for **no-ip.com** with `ddclient`.

```bash
# ddclient.conf für No-IP (dyndns2)
protocol=dyndns2
use=web, web=checkip.dyndns.com/, web-skip='IP Address'
server=dynupdate.no-ip.com
login=USERNAME
password=PASSWORD
DOMAINNAME.SUFFIX
```

```bash
#!/bin/bash
sudo systemctl enable --now ddclient
sudo systemctl status ddclient
```

If `ddclient` has a error within it's status restart the service and try it again.

## Jellyfin

Also i like to run a Jellyfin server. And if you use the raspberry only for transfering the data it's no big deal because the encoding process will happen on the client devices.

Here the oneshot line to install Jellyfin.

```bash
#!/bin/bash
curl -s https://repo.jellyfin.org/install-debuntu.sh | sudo bash
```

For the storage I use a hard drive that is mounted through the configuration that I made within `/etc/fstab`.

Output every UUID and storage information of connected devices.

```bash
#!/bin/bash
sudo blkid
```

And now you can use that UUID of your choosen device to mount that device where you want to using `/etc/fstab`. I would recommend to have the device with the ext4 data systems because it's the same as evey linux systems uses. So it will probably not have any issues because of the format. 

Add this line with your mounting directory and the UUID of your device to the file `/etc/fstab`.

```bash
...
# jellyfin
UUID=YOUR_UUID /MOUNTINGDIR ext4 defaults,users,exec 0 0
```

After a reboot the device should be mounted. You can check that using `df`. If everything is fine you can refer within jellyfin to the directories of the device and storage your data on this device.

### HTTPS

It is important to know that every client software that I know can't handle self signed SSL certificates. Other users have the same problem here. So I would say if you want to use HTTPS you have no choice but to make a certified SSL certificate. For that you can use `certbot`. 

Or you make a reverse proxy. The scenario is that Jellyfin uses a self signed SSL certificate and your desktop binds that connection locally over HTTP using Apache2 or something like that. 

But that would only workout for devices that can bind the HTTPS connection locally on a local HTTP side. So mobile devices would have trouble with that.