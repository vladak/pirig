# Common

## Install preparations

initially based on https://caffinc.github.io/2016/12/raspberry-pi-3-headless/

- use the minimal/Lite version of Raspbian
  - avoid X server, browsers, desktop packages
- flash the system to the micro SD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
  - use the Lite version
  - Ctrl+Shift+X: change the configuration withing the Imager
    - change hostname
    - enable SSH
      - allow pubkey auth only
    - change password of the `pi` user
    - disable telemetry
- create DHCP entry for the Pi on the router
- power up the Pi, wait for install to finish
- SSH into the Pi as the `pi` user
- run `sudo raspi-config` and change:
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
