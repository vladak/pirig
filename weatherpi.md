# Weather Pi setup

Assumes:
  - Raspbian installed
  - Ethernet connected + IP connectivity outside (allow this on the gateway in the appropriate VLAN rules)
  - [DS9490R](https://www.maximintegrated.com/en/products/interface/universal-serial-bus/DS9490.html) 1-wire USB dongle connected to USB port
    - 1-wire temperature sensors connected to the USB dongle via RJ-11 
  - [BMP280](https://www.adafruit.com/product/2651) sensor connected via I2C
    - using the guide from https://learn.adafruit.com/adafruit-bmp280-barometric-pressure-plus-temperature-sensor-breakout/circuitpython-test


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

### Setup dashboards

Use the `*dashboard.json` exports to provision Grafana.

The Y axis limits for barometric pressure are set based on https://en.wikipedia.org/wiki/List_of_atmospheric_pressure_records_in_Europe

### 502 gateway problem in Grafana

After adding data source to Grafana, it complained (502 gateway error) about it. I had to change data source configuration in Grafana to use `127.0.0.1` instead of `localhost` and then it started working.
