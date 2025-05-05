# Setup - MyHomePi

Here I explain how I setup my own server for my private usage at home. I use the RaspberryPi 4B with 8GB memory and a external hard driver as a storage. As an operating system I use raspian because it's used to be the stablest ones. To be exact here I use `Raspberry Pi OS Lite (64-bit)`.

Also [here](./Resources/SERVICES.md) a list of services that I host right now. These services, it's usage and it's installation will be expained later on.

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