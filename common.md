# Common

## Install preparations

initially based on https://caffinc.github.io/2016/12/raspberry-pi-3-headless/

- use the minimal/Lite version of Raspbian
  - avoid X server, browsers, desktop packages
- flash the system to the micro SD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
  - use the Lite version (64-bit if possible - for Raspberry Pi 4)
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
  - time zone to `Europe/Prague`
  - GPLU memory split to 16 MB (in Performance options)
- add `commit=60` mount option to `/` in `/etc/fstab`
  - helps to avoid I/O done by ext4 journaling thread (`jbd2`)
- disable swap (per https://www.dzombak.com/blog/2023/12/Stop-using-the-Raspberry-Pi-s-SD-card-for-swap.html)
```
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo update-rc.d dphys-swapfile remove
```
- assumes that `systemd-timesyncd` service is installed
  - it picks up the NTP server from DHCP options
- disable unneeded features (per https://www.dzombak.com/blog/2023/12/Disable-or-remove-unneeded-services-and-software-to-help-keep-your-Raspberry-Pi-online.html)
```
# mandb rebuild
sudo rm /var/lib/man-db/auto-update
# avahi for .local/mDNS
sudo apt remove --purge avahi-daemon
sudo apt autoremove --purge
# modem manager
sudo apt remove --purge modemmanager
sudo apt autoremove --purge
# Bluetooth
sudo systemctl disable bluetooth.service
sudo systemctl disable hciuart.service
sudo apt remove --purge bluez
# unneeded software
sudo apt remove --purge wolfram-engine triggerhappy xserver-common lightdm
sudo apt autoremove --purge
```
- automatically reboot after kernel panic (https://www.dzombak.com/blog/2023/12/Mitigating-hardware-firmware-driver-instability-on-the-Raspberry-Pi.html)
```
sudo sysctl -w kernel.panic=1
echo "kernel.panic = 1" | sudo tee /etc/sysctl.d/90-kernelpanic-reboot.conf
```

## Initial install

```
# The --allow-releaseinfo-change is handy in case: "changed its 'Suite' value from 'unstable' to 'stable'"
sudo apt-get -y update --allow-releaseinfo-change
sudo apt-get -y dist-upgrade
sudo apt-get -y upgrade
sudo apt-get -y install file
sudo apt-get -y install vim
sudo apt-get -y install sysstat # for iostat/sar
sudo apt-get -y install iotop # run iotop with -ob
sudo apt-get -y install tcpdump netcat # for network debugging
sudo apt-get -y install unattended-upgrades
```

## Log2ram

Use https://github.com/azlux/log2ram to store `/var/log` in RAM and sync it periodically to disk.
This helps to save the SD card.

Bump the value of `SIZE` to 100M in `/etc/log2ram.conf`.

This will help with Pi-hole logging. Use `sudo iotop -ob` to confirm.

## Temperature monitoring

Setup https://github.com/vladak/systemp
