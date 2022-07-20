# Weather Pi setup

Assumes:
  - sufficiently big SD card to store the Prometheus data with long retention (bigger than 32 GB)
  - Raspbian installed
    - make sure this is 64-bit as it is needed for Prometheus
  - Ethernet connected + IP connectivity outside (allow this on the gateway in the appropriate VLAN rules)
    - the outside connectivity is needed e.g. for Grafana to send alerts via PagerDuty
  - sensors as per https://github.com/vladak/weather/blob/master/README.md#sensors

## RTC

In order to keep the correct time for Grafana/Prometheus, a RTC is installed. This is the [Adafruit PCF8523 Real Time Clock Breakout Board - STEMMA QT](https://www.adafruit.com/product/5189) with CR1220 battery. It is connected to the same QT board as the environment sensors.

The setup follows the steps on https://github.com/vladak/pirig/blob/master/pihole.md#rtc

## Weather service

Contains monitoring for various environmental metrics.

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

and follow the instructions in the `README.md` file.

## Prometheus

I had to replace Telegraf+InfluxDB since the latter was really memory hungry and even on 4GB RAM it entered periods of time
when it was not possible to get/store data and it was just spinning on CPU.

Note that [Prometheus does not really support running as 32-bit program](https://github.com/prometheus/prometheus/issues/4392#issuecomment-433721793) so make sure to use arm64 variant.

Install Prometheus:

Note: do not install from APT since Raspbian contains really old version. Instead, grab the latest stable form armv7 from https://prometheus.io/download/

```
wget https://github.com/prometheus/prometheus/releases/download/v2.34.0/prometheus-2.34.0.linux-armv7.tar.gz
cd
tar xfz prometheus-2.34.0.linux-armv7.tar.gz
mv prometheus-2.34.0.linux-armv7 prometheus
```

create the user/group:
```
groupadd -g 120 prometheus
useradd -u 115 -g 120 -s /usr/sbin/nologin prometheus
```

setup the service:

```
cat << EOF | sudo tee >/etc/systemd/system/prometheus.service
[Unit]
Description=Monitoring system and time series database
Documentation=https://prometheus.io/docs/introduction/overview/

[Service]
Restart=always
User=prometheus
EnvironmentFile=/etc/default/prometheus
ExecStart=/home/pi/prometheus/prometheus $ARGS
ExecReload=/bin/kill -HUP $MAINPID
TimeoutStopSec=20s
SendSIGKILL=no
LimitNOFILE=8192

[Install]
WantedBy=multi-user.target
EOF
```

create the data directory:
```
mkdir /home/pi/prometheus-data
sudo chown prometheus:prometheus prometheus-data
```

create the initial config options (set the data size per expected available space on the file system):
```
cat << EOF | sudo tee >/etc/default/prometheus
ARGS="--storage.tsdb.retention.size=20GB --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/home/pi/prometheus-data"
EOF
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

enable the service:

```
sudo systemctl enable prometheus
```

check the state
```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

## Grafana

### Catch 1: install Grafana from the right source

Trying to install Grafana from the standard Raspbian repositories leads to blank page on port 3000 where Grafana web server listens for requests. This is due to Angular JS problems in the what seems to be abandoned package (https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=843263) so the recommended way is to install the package from the official place:
```
wget https://dl.grafana.com/oss/release/grafana_6.2.5_armhf.deb
sudo dpkg -i grafana_6.2.5_armhf.deb
```

The package will create the `grafana` user and group as well as the Grafana services.

### Provision dashboards

Use the `*dashboard.json` exports to provision Grafana if needed.

The Y axis limits for barometric pressure are set based on https://en.wikipedia.org/wiki/List_of_atmospheric_pressure_records_in_Europe

### Setup PF monitoring

use https://gist.github.com/vladak/5d2447ae774bc4886fe1ab549fff8d8b

### 502 gateway problem in Grafana

After adding data source to Grafana, it complained (502 gateway error) about it. I had to change data source configuration in Grafana to use `127.0.0.1` instead of `localhost` and then it started working.

### Setup PagerDuty alert channel and notifications

Copy the *Integration Key* from the PagerDuty web interface.

Setup alert rule for outside temperature.
