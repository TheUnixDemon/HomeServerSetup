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

For the configuration of this service you can add a onfiguration file to `/etc/network/interfaces.d/YOUR-CONF` but I'd like to use the basic config file `/etc/network/interfaces`. Here a template for that.

```bash
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

# your normal network; gateway of your other networks/interfaces
auto eth0
iface eth0 inet static
    address 192.168.0.44/24
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