# Pihole setup

This Pi serves as:
  - NTP server
    - with RTC backed system time
  - Pi-hole ad filtration endpoint

## Components

- RPi 3
- RTC clock: [PiRTC DS3231](https://www.adafruit.com/product/4282) with CR1220 battery
- 32 GiB micro SD card

![Pihole with RTC](/img/Pi_DS3231.jpg)

## NTP daemon

- install:
```
sudo apt-get install ntp/stable
```

Note: OpenNTPd would be better, however it has some issue that on the Raspberry Pi stops responding to clients after some time.

## RTC

Using the guide on https://learn.adafruit.com/adding-a-real-time-clock-to-raspberry-pi

- enable I2C in `sudo raspi-config`
  - `Interface options` menu
- install pre-requisites:
```
sudo apt-get install -y i2c-tools
```
- check that the clock is visible (should report ID `68` in one of the cells):
```
sudo i2cdetect -y 1
```
- add the system config:
  - for DS3231 (the more precise variant): `echo "dtoverlay=i2c-rtc,ds3231" | sudo tee -a /boot/config.txt`
  - or for [PCF8523](https://www.adafruit.com/product/5189): `echo "dtoverlay=i2c-rtc,pcf8523" | sudo tee -a /boot/config.txt`
- reboot:
```
sudo reboot
```
- check once again (should report `UU` instead of `68`):
```
sudo i2cdetect -y 1
```
- disable file-system based "clock":
```
    sudo apt-get -y remove fake-hwclock
    sudo update-rc.d -f fake-hwclock remove
    sudo systemctl disable fake-hwclock
```
- test (should successfully read the time from RTC eventually) 
```
sudo hwclock --test
```
- overwrite `/lib/udev/hwclock-set`:
```
cat <<EOF | sudo tee /lib/udev/hwclock-set
#!/bin/bash
dev=\$1
/sbin/hwclock --rtc=$dev --hctosys
EOF
```

## Pi-hole

Use the same style of install as described on https://blog.sleeplessbeastie.eu/2018/01/11/how-to-install-and-configure-pi-hole/

```
sudo apt-get -y install git
git clone --depth 1 https://github.com/pi-hole/pi-hole.git pi-hole
sudo bash pi-hole/automated\ install/basic-install.sh
```

Use a custom DNS server in the configuration - set it to the IP address of default GW.

Then change the admin password: `pihole -a -p`

Initially, Pi-hole was installed via Docker however later was reversed in favor of standalone install because:
  - Pi-Hole docker image does not allow to control logging sufficiently
    - `dnsmasq` and `lighttpd` logging to `/var/log` would wear the SD card out
    - https://github.com/pi-hole/docker-pi-hole/issues/366 , same thing for `lighttpd` logging
      - had to use clunky workaround (introduce system service to check settings periodically and disable logging)
  - `containerd` also contributes to the I/O - it periodically writes to disk

Once Pihole dashboard is up, disable query logging and flush the logs:
```
  pihole logging off
```

Allow non-local queries in the PiHole web admin interface - switch to *Respond only on interface eth0*

### Use custom lists

combine lists from:
  - https://github.com/vladak/fishysites
  - firebog.net

This updates the database without deleting everything:
```
sqlite3 /etc/pihole/gravity.db "SELECT Address FROM adlist" |sort >/home/pi/pihole.list
wget -qO - https://v.firebog.net/hosts/lists.php?type=tick |sort >/home/pi/firebog.list
echo https://raw.githubusercontent.com/vladak/fishysites/master/fishy_domains.txt >/home/pi/fishy.list
cat /home/pi/firebog.list /home/pi/fishy.list | sort > complete.list
comm -23 pihole.list complete.list |xargs -I{} sudo sqlite3 /etc/pihole/gravity.db "DELETE FROM adlist WHERE Address='{}';"
comm -13 pihole.list complete.list |xargs -I{} sudo sqlite3 /etc/pihole/gravity.db "INSERT INTO adlist (Address,Comment,Enabled) VALUES ('{}','firebog, added `date +%F`',1);"
pihole restartdns reload-lists
pihole -g
```

