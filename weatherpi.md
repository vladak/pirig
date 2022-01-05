# Pi setup

Assumes:
  - Ethernet connected + IP connectivity outside (allow this on the gateway in the appropriate VLAN rules)
  - 1-wire USB dongle connected to USB port
  - BME280 sensor connected via I2C (https://learn.adafruit.com/adafruit-bme280-humidity-barometric-pressure-temperature-sensor-breakout)

## Barometric pressure monitoring

### I2C

- enable I2C via `sudo raspi-config`
- verify that the sensor is visible via `sudo i2cdetect -l`

### Python

XXX

## Temperature monitoring

### OWFS

```
sudo apt-get install owfs
```

- change `/etc/owfs.conf` to contain the following line and comment about any lines with `FAKE` sensors
```
server: usb = all
```

Initially this was not working and the `owfs` service complained about no bus being seen. `apt-get update && apt-get upgrade` pulled bunch of raspberrypi kernel updates and after reboot the sensors were available under the `/run/owfs` directory.

### Temperature monitoring

Using the guide on https://www.abelectronics.co.uk/kb/article/3/owfs-with-i2c-support-on-raspberry-pi:

```
sudo apt-get -y install python3-ow
```

Then it is possible to get sensor readings:
```python
import ow
ow.init('localhost:4304')
sensorlist = ow.Sensor('/').sensorList()
for sensor in sensorlist:
    print('Device Found')
    print('Address: ' + sensor.address)
    print('Family: ' + sensor.family)
    print('ID: ' + sensor.id)
    print('Type: ' + sensor.type)
    print(' ')
```

To get temperature readings:
```python
sensorlist[0].temperature
```

### Weather service

Install Git to get the code:
```
sudo apt-get -y install git
```

Clone it:
```
sudo mkdir /srv/weather
sudo chown pi /srv/weather
git clone https://github.com/vladak/weather /srv/weather
```

And follow the instructions in the README file.

## Prometheus

I had to replace Telegraf+InfluxDB since the latter was really memory hungry and even on 4GB RAM it entered periods of time
when it was not possible to get/store data and it was just spinning on CPU.

Install Prometheus:

```
sudo apt-get install prometheus
```

add this to the config in the `scrape_configs` section:
```yml
  - job_name: weather
    static_configs:
      - targets: ['localhost:8111']
      
  - job_name: CPU_localhost
    static_configs:
      - targets: ['localhost:8222']

  - job_name: CPU_pihole
    static_configs:
      - targets: ['pi:8222']
```

## Grafana

### Catch 1: install Grafana from the right source

Trying to install Grafana from the standard Raspbian repositories leads to blank page on port 3000 where Grafana web server listens for requests. This is due to Angular JS problems in the what seems to be abandoned package (https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=843263) so the recommended way is to install the package from the official place:
```
wget https://dl.grafana.com/oss/release/grafana_6.2.5_armhf.deb
sudo dpkg -i grafana_6.2.5_armhf.deb
```

### 502 gateway problem in Grafana

After adding data source to Grafana, it complained (502 gateway error) about it. I had to change data source configuration in Grafana to use `127.0.0.1` instead of `localhost` and then it started working.
