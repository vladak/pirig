# Common

## Install preparations

use https://caffinc.github.io/2016/12/raspberry-pi-3-headless/

- use the minimal/Lite version of Raspbian
  - avoid X server, browsers, desktop packages
- flash the system to the micro SD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
- create DHCP entry for the Pi on the router
- SSH into the pi
- power up the Pi, wait for install to finish
- run `raspi-config` and change:
  - default system locale to `C.UTF-8`
  - time zone to `Europe/Prague`
  - GPLU memory split to 16 (in Advanced options)
- add `commit=60` mount option to `/` in `/etc/fstab`
  - helps to avoid I/O done by ext4 journaling thread (`jbd2`)

## Initial install

```
sudo apt-get update
sudo apt-get dist-upgrade
sudo apt-get upgrade
sudo apt-get -y install vim
sudo apt-get -y install sysstat # for iostat/sar
sudo apt-get -y install iotop # run iotop with -ob
sudo apt-get -y install tcpdump netcat # for network debugging
```

- setup SSH key to login
- change the password of the `pi` account

## OpenNTPd

- install (as client):
```
sudo apt-get -y install openntpd
service openntpd status
```
- add `-s` to `DAEMON_OPTS` in `/etc/default/openntpd` to synchronize time right away at startup

## Log2ram

Use https://github.com/azlux/log2ram to store `/var/log` in RAM and sync it periodically to disk.
This helps to save the SD card.

Bump the value of `SIZE` to 100M in `/etc/log2ram.conf`.

This will help with Pi-hole logging. Use `sudo iotop -ob` to confirm.
