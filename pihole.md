# Pi 3 nr. 2 setup

This Pi serves as NTP server (with RTC backed system time) and Pi-hole ad filtration endpoint.

## Components

- RPi 3
- Adafruit PiOLED: https://rpishop.cz/adafruit/874-adafruit-128x32-pioled.html
- RTC clock: https://rpishop.cz/piface/149-piface-real-time-clock.html
- 32 GiB micro SD card
- power adapter for RPi 4

## RTC

Do not use the official script linked from http://www.piface.org.uk/assets/piface_clock/PiFaceClockguide.pdf , it does not work (https://github.com/piface/PiFace-Real-Time-Clock/issues/14). Instead, use this (adapted a bit from https://www.raspberrypi.org/forums/viewtopic.php?p=1234070):

- enable I2C in `sudo raspi-config`
  - `Interface options` menu
- install pre-requisites:
```
sudo apt-get install i2c-tools
```
- install the new service:
```
sudo mkdir /srv/pifacertc
sudo chown $LOGNAME /srv/pifacertc
cat << EOF >/srv/pifacertc/pifacertc.sh
#!/bin/sh

. /lib/lsb/init-functions

log_success_msg "Probe the i2c-dev"
modprobe i2c-dev
# Calibrate the clock (default: 0x47). See datasheet for MCP7940N
log_success_msg "Calibrate the clock"
i2cset -y 1 0x6f 0x08 0x47
log_success_msg "Probe the mcp7941x driver"
modprobe i2c:mcp7941x
log_success_msg "Add the mcp7941x device in the sys filesystem"
# https://www.kernel.org/doc/Documentation/i2c/instantiating-devices
echo mcp7941x 0x6f > /sys/class/i2c-dev/i2c-1/device/new_device
log_success_msg "Synchronise the system clock and hardware RTC"
hwclock --hctosys
EOF
sudo chmod +x /srv/pifacertc/pifacertc.sh
sudo touch /etc/systemd/system/pifacertc.service
sudo chown $LOGNAME /etc/systemd/system/pifacertc.service
cat << EOF >/etc/systemd/system/pifacertc.service
[Unit]
Description=PiFace real-time clock

[Service]
ExecStart=/srv/pifacertc/pifacertc.sh

[Install]
WantedBy=multi-user.target
EOF
sudo chown root /etc/systemd/system/pifacertc.service
sudo systemctl daemon-reload
sudo systemctl enable pifacertc.service
sudo systemctl start pifacertc.service
```
- check:
```
sudo systemctl status pifacertc.service
sudo hwclock --test
```

## OpenNTPd

- enable as server: uncomment line `listen on *` in `/etc/openntpd/ntpd.conf` and restart the service:
```
sudo service openntpd restart
```

## Pi-hole

Use the same style of install as described on https://blog.sleeplessbeastie.eu/2018/01/11/how-to-install-and-configure-pi-hole/

```
git clone --depth 1 https://github.com/pi-hole/pi-hole.git pi-hole
sudo bash pi-hole/automated\ install/basic-install.sh
```

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

### Custom blacklist for fishy domains

Follow the instructions on https://github.com/vladak/fishysites

## PiOLED

use https://github.com/vladak/PiOLED
