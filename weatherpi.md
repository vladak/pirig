# Weather Pi setup

Assumes:
  - sufficiently big SD card to store the Prometheus data with long retention (bigger than 32 GB)
  - Raspbian installed
    - make sure this is 64-bit (_Raspberry Pi OS Lite (64-bit)_ in the Raspberry Pi Imager app) as it is needed for Prometheus to store large amounts of data (more than 4 GB)
  - Ethernet connected + IP connectivity outside (allow this on the gateway in the appropriate VLAN rules)
    - the outside connectivity is needed e.g. for Grafana to send alerts via PagerDuty
  - sensors as per https://github.com/vladak/weather/blob/master/README.md#sensors

## RTC

In order to keep the correct time for Grafana/Prometheus, a RTC is installed. This is the [Adafruit PCF8523 Real Time Clock Breakout Board - STEMMA QT](https://www.adafruit.com/product/5189) with CR1220 battery. It is connected to the same QT board as the environment sensors.

The setup follows the steps on https://github.com/vladak/pirig/blob/master/pihole.md#rtc

## Disable unneeded services

```
sudo systemctl stop bluetooth
sudo systemctl disable bluetooth
```

## MQTT broker

- Install Mosquitto MQTT broker:
```
sudo apt install -y mosquitto mosquitto-clients
cat << EOF | sudo tee /etc/mosquitto/conf.d/local.conf
allow_anonymous true
listener 1883
EOF
sudo systemctl restart mosquitto
systemctl status mosquitto
```
- Test basic MQTT functionality:
```
mosquitto_sub -d -t test
mosquitto_pub -d -t test -m "Hello, World!"
```

## mq2anno service

This is for displaying Grafana annotations from MQTT messages.

Follow the instructions on https://github.com/vladak/mq2anno/

## Plug to MQTT service

This is for publishing state of Tapo smart plugs to MQTT.

Follow the instructions on https://github.com/vladak/plug2mqtt/

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

Note: do not install from APT since Raspbian contains really old version. Instead, grab the latest stable arm64 release from https://prometheus.io/download/

```
wget https://github.com/prometheus/prometheus/releases/download/v2.37.0/prometheus-2.37.0.linux-arm64.tar.gz
cd
tar xfz prometheus-2.37.0.linux-arm64.tar.gz
mv prometheus-2.37.0.linux-arm64 prometheus
```

create the user/group:
```
sudo groupadd -g 120 prometheus
sudo useradd -u 115 -g 120 -s /usr/sbin/nologin prometheus
```

setup the service:

```
cat << EOF | sudo tee /etc/systemd/system/prometheus.service
[Unit]
Description=Monitoring system and time series database
Documentation=https://prometheus.io/docs/introduction/overview/

[Service]
Restart=always
User=prometheus
EnvironmentFile=/etc/default/prometheus
ExecStart=/home/pi/prometheus/prometheus \$ARGS
ExecReload=/bin/kill -HUP \$MAINPID
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
cat << EOF | sudo tee /etc/default/prometheus
ARGS="--storage.tsdb.retention.size=200GB --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/home/pi/prometheus-data"
EOF
```

create initial configuration in `/etc/prometheus/prometheus.yml`

add this to the config in the `scrape_configs` section in the configuration file:
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

### node exporter

To see system information install Prometheus node exporter:
```
sudo apt-get -y install prometheus-node-exporter
```
and add this snippet to the `scrape_configs` section of `/etc/prometheus/prometheus.yml`: 
```
  - job_name: node
    # If prometheus-node-exporter is installed, grab stats about the local
    # machine by default.
    static_configs:
      - targets: ['localhost:9100']
```

### MQTT exporter

To be able to receive metrics from external sensors, install the [MQTT exporter](https://github.com/hikhvar/mqtt2prometheus):
```
sudo apt-get -y install prometheus-mqtt-exporter
```
Setup configuration in `/etc/prometheus/mqtt-exporter.yaml`
and add this snippet to the `scrape_configs` section of `/etc/prometheus/prometheus.yml`: 
```
  - job_name: mqtt
    # If prometheus-mqtt-exporter is installed, grab metrics from external sensors.
    static_configs:
      - targets: ['localhost:9641']
```
and restart both services:
```
sudo systemctl restart prometheus-mqtt-exporter
sudo systemctl restart prometheus
```

## Grafana

### Catch 1: install Grafana from the right source

Trying to install Grafana from the standard Raspbian repositories leads to blank page on port 3000 where Grafana web server listens for requests. This is due to Angular JS problems in the what seems to be abandoned package (https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=843263) so the recommended way is to install the package from the official place:
```
sudo apt-get install -y adduser libfontconfig1
wget https://dl.grafana.com/enterprise/release/grafana-enterprise_9.0.3_arm64.deb
sudo dpkg -i grafana-enterprise_9.0.3_arm64.deb
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
