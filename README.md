# raspi-pihole




Raspberry Pi 2 model B running Pi-Hole w/ a Python daemon sending statistics to InfluxDB for Grafana monitoring



## Getting Started




### Prerequisites



The following software dependencies are required:

* pi-hole
* InfluxDB
* Python3 and `pip`
* `git`
* [RPi.GPIO](https://pypi.org/project/RPi.GPIO/) 
* [Pi-Hole-Influx](https://github.com/janw/pi-hole-influx)


This setup sends pi-hole data to an InfluxDB instance hosted on an Amazon aws ec2 instance with HTTPS authentication enabled. Refer to the [tig-stack-aws](https://github.com/jihomc/tig-stack-aws) repository for setup instructions.



## Installations 



### Pi-Hole


#### Installation

Run automated installer

```shell
curl -sSL https://install.pi-hole.net | bash
```

#### Configuration

Configure router to have DHCP clients use Pi-hole as the DNS server.

Reserve static IP address in router for raspberry pi

Change admin console password

```shell
pihole -a -p
```


#### Usage

##### View the web interface

http://pi.hole/admin


##### Pihole commmand line

https://docs.pi-hole.net/core/pihole-command/

Change password

```shell
pihole -a -p
```

Chronometer

```shell
pihole -c -e
```

#### Troubleshooting

Troubleshooting after unexpected loss of power.

```shell
fsck
pihole -r
pihole reconfigure
```


### InfluxDB

On the server hosting InfluxDB, enter the influx CLI to create a database for pi-hole-influx to send data to.

```shell
influx -ssl -unsafeSsl -host 127.0.0.1
> auth
username: username
password: password
> CREATE DATABASE raspi
> SHOW DATABASES
> exit
```


### Pi-hole-Influx



A simple daemonized script to report Pi-Hole stats to InfluxDB

https://github.com/janw/pi-hole-influx



#### Installation


Install Python3, pip, and git

```shell
sudo apt update
sudo apt install git python3-pip -y
```

Clone the repo and install Python dependences

```shell
git clone https://github.com/janw/pi-hole-influx.git ~/pi-hole-influx
cd ~/pi-hole-influx

pip3 install -r requirements.txt
```


#### Configuration


Copy config.example and modify it

```shell
cp user.toml.example user.toml
vim user.toml
```

Modify default.toml settings - InfluxDB w/ HTTPS
In this example, InfluxDB is configured with HTTPS using self-signed ssl certificates 

```shell
~/pi-hole-influx/default.toml
influxdb_ssl = "True"
influxdb_verify_ssl = "False"
```

Edit user.toml to include InfluxDB credentials

```shell
influxdb_host = "ip_address"
influxdb_port = "8086"
influxdb_database = "raspi"
influxdb_username = "username"
influxdb_password = "password"

request_timeout = "10"
reporting_interval = "30"

instances="localhost=http://127.0.0.1/admin/api.php"
```

Symlink the systemd service into place, reload, and enable

```shell
sudo ln -s ~/pi-hole-influx/piholeinflux.service /etc/systemd/system/
sudo systemctl --system daemon-reload
sudo systemctl enable piholeinflux.service
sudo systemctl start piholeinflux.service
sudo systemctl status piholeinflux.service
```

Troubleshooting service start errors

```shell
sudo systemctl status piholeinflux.service
journalctl -xe
```
 



### Pi-hole Under-voltage warnings script



#### Installation


Install RPi.GPIO

```shell
sudo apt update
sudo apt upgrade
sudo apt install python3-rpi.gpio
```


#### Configuration


Download [piPower](https://github.com/electronicsguy/piPower) script - shows LED status in the console (over SSH)

Edit piPower script to fix #! and print statement for Python3

```shell
vim ~/raspi-pihole/piPower

#!/usr/bin/python3

print("Low power for " + str(powerlow) + " seconds")
```

Execute script

```shell
sudo Python3 piPower
```


 
