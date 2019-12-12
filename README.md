# raspi-hole-nginx





Raspberry Pi 2 model B running Pi-Hole with Nginx as the webserver, plus a Python daemon sending pi-hole statistics to an InfluxDB database hosted on Amazon aws for Grafana monitoring





## Getting Started




### Prerequisites



The following dependencies are required:

* Pi-hole
* Nginx
* Python3 and `pip`
* `git`
* [RPi.GPIO](https://pypi.org/project/RPi.GPIO/) 
* [Pi-Hole-Influx](https://github.com/janw/pi-hole-influx)

This setup sends pi-hole data to an InfluxDB instance hosted on an Amazon aws ec2 instance with HTTPS authentication enabled. Refer to the [aws-tig-nginx](https://github.com/jihomc/aws-tig-nginx) repository for setup instructions.



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



### InfluxDB - aws



On the aws ec2 server hosting InfluxDB, enter the influx CLI to create a database for pi-hole-influx to send data to. InfluxDB is configured with HTTPS already. 

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


Copy the pi-hole-influx example config

```shell
cd ~/pi-hole-influx
cp user.toml.example user.toml
vim user.toml
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

Edit default.toml to include HTTPS settings for InfluxDB

In this example, InfluxDB is configured with HTTPS using self-signed ssl certificates 

```shell
vim ~/pi-hole-influx/default.toml
influxdb_ssl = "True"
influxdb_verify_ssl = "False"
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



### Nginx - raspi



#### Installation


Download packages

```shell
https://nginx.org/en/linux_packages.html
sudo apt install curl gnupg2 ca-certificates lsb-release
```

Set up the apt repository for stable packages

```shell
echo "deb http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
      | sudo tee /etc/apt/sources.list.d/nginx.list
```

Import official nginx signing key for verifcation of packages

```shell
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
sudo apt-key fingerprint ABF5BD827BD9BF62
```

The output should contain the full fingerprint 573B FD6B 3D8F BC64 1079 A6AB ABF5 BD82 7BD9 BF62

```shell
sudo apt update
sudo apt install nginx
```


#### Configuration


Configuration File

```shell
sudo vim /etc/nginx/nginx.conf
```

Starting, stopping, reloading nginx

```shell
sudo systemctl [ start | restart | status ] nginx
```


#### Use Nginx instead of Lighttpd as the webserver

##### Documentation: https://docs.pi-hole.net/guides/nginx-configuration/

Basic requirements

1. Stop default lighttpd

```shell
service lighttpd stop
```

2. Install necessary packages

```shell
apt-get -y install nginx php7.0-fpm php7.0-zip apache2-utils
```

##### Encountered issues installing php7.0 with the above command, solution below

```shell
sudo apt-add-repository ppa:ondrej/php
sudo apt-get update
sudo apt-get install php7.0
```

3. Disable lighttpd at startup

```shell
systemctl disable lighttpd
```

4. Enable php7.0-fpm at startup

##### Encountered issues with php7.0-fpm as well, solution below

```shell
sudo apt-cache search php7.0- |more
sudo apt-get install php7.0-fpm

systemctl enable php7.0-fpm
```

5. Enable nginx at startup

```shell
systemctl enable nginx
```

6. Edit /etc/nginx/nginx.conf to on the raspi to use nginx as the pi-hole webserver

See [nginx.conf](https://github.com/jihomc/raspi-hole-nginx/blob/master/nginx/nginx.conf)


##### Additional configuration

Add domain to setupVars.conf on raspi

```shell
vim /etc/pihole/setupVars.conf

IPV4_ADDRESS=www.pi.mydomain.com
```

Change ownership of html director to nginx user

```shell
chown -R www-data:www-data /var/www/html
```

Make sure html directory is writable

```shell
chmod -R 755 /var/www/html
```

Start php7.0-fpm daemon

```shell
systemctl start php7.0-fpm
``` 

Start nginx webserver

```shell
systemctl start nginx
```



#### Nginx - aws instance 



Nginx on aws will act as a reverse proxy sitting in front of the raspi webserver. Requests to pi.mydomain.com will be sent to the pi-hole admin console hosted on the raspi webserver.

See the [aws instance nginx.conf](https://github.com/jihomc/raspi-hole-nginx/blob/master/aws/nginx.conf)

 
