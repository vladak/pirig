# Pi 3 setup

## OWFS

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

Then create the log file and make it readable to everyone so that the Telegraf service can read it (it runs under the `telegraf` service):
```
touch /var/log/temperature.log
chmod o+r /var/log/temperature.log
```

And follow the instructions in the README file.

## Telegraf

Get the package from https://github.com/influxdata/telegraf/releases
```
wget https://dl.influxdata.com/telegraf/releases/telegraf_1.11.3-1_armhf.deb
sudo dpkg -i telegraf_1.11.3-1_armhf.deb
```

In case InfluxDB database is dropped, it will not be recreated until the `telegraf` service is restarted.

## InfluxDB

First install InfluxDB:
```
sudo apt-get -y install influxdb influxdb-client
```

Then configure the HTTP endpoint in `/etc/influxdb/influxdb.conf` so that the section looks like this:
```
[http]
  # Determines whether HTTP endpoint is enabled.
   enabled = true

  # The bind address used by the HTTP service.
   bind-address = "localhost:8086"

  # Determines whether HTTP request logging is enabled.
  log-enabled = false
```
Also set the logging level to `error` so that the `[logging]` section looks like this:
```
[logging]
  # Determines which log encoder to use for logs. Available options
  # are auto, logfmt, and json. auto will use a more a more user-friendly
  # output format if the output terminal is a TTY, but the format is not as
  # easily machine-readable. When the output is a non-TTY, auto will use
  # logfmt.
  # format = "auto"

  # Determines which level of logs will be emitted. The available levels
  # are error, warn, info, and debug. Logs that are equal to or above the
  # specified level will be emitted.
  level = "error"

  # Suppresses the logo output that is printed when the program is started.
  # The logo is always suppressed if STDOUT is not a TTY.
  # suppress-logo = false
```
The `log-enabled` and log level settings will ensure `daemon.log` and `syslog.log` in the `/var/log` directory do not grow too much.

Tried to reduce the memory used by InfluxDB so that the `[data]` section contains:
```
  # CacheMaxMemorySize is the maximum size a shard's cache can
  # reach before it starts rejecting writes.
  # Valid size suffixes are k, m, or g (case insensitive, 1024 = 1k).
  # Values without a size suffix are in bytes.
  cache-max-memory-size = "300m"

  # CacheSnapshotMemorySize is the size at which the engine will
  # snapshot the cache and write it to a TSM file, freeing up memory
  # Valid size suffixes are k, m, or g (case insensitive, 1024 = 1k).
  # Values without a size suffix are in bytes.
  cache-snapshot-memory-size = "1m"
```
however this leads to strange behavior of the `influxd` (high CPU utilization) and no memory saving
ref: https://github.com/influxdata/influxdb/issues/5440

## Grafana

### Catch 1: install Grafana from the right source

Trying to install Grafana from the standard Raspbian repositories leads to blank page on port 3000 where Grafana web server listens for requests. This is due to Angular JS problems in the what seems to be abandoned package (https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=843263) so the recommended way is to install the package from the official place:
```
wget https://dl.grafana.com/oss/release/grafana_6.2.5_armhf.deb
sudo dpkg -i grafana_6.2.5_armhf.deb
```

### 502 gateway problem in Grafana

After adding InfluxDB data source to Grafana, it complained (502 gateway error) about it. I had to change `bind-address = "localhost:8086"` in `/etc/influxdb/influxdb.conf`, restart the `influxdb` service. This was not enough so I tried to change data source configuration in Grafana to `127.0.0.1` and it started working.
