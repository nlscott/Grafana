# Setup Grafana, Influxdb, and Telegraf on Ubuntu 18.04

### Overview:
Assuming you already have an Ubuntu VM/box setup and configured how you want.

- Install & Configure Influxdb
- Install & Configure Telegraf
- Install & Configure Grafana
- Configure SSL With Self-Signed Certs For Grafana
- Setup Influxdb With Self-Signed Certs

----------------------------------------------------------------------------------------------------
**Install & Configure Influxdb**
See [Influxdb documentation](https://docs.influxdata.com/influxdb/v1.7/introduction/installation/) for changes or newer versions.

Default config file: `/etc/influxdb/influxdb.conf`

Install influxdb 1.7.3 (current version at the time), and set as a service to start at boot:
```bash
wget https://dl.influxdata.com/influxdb/releases/influxdb_1.7.3_amd64.deb
sudo dpkg -i influxdb_1.7.3_amd64.deb
sudo systemctl start influxdb
sudo systemctl enable influxdb
```

create the default influxdb database and user:
```sql
create database telegraf
create user telegraf with password 'password'
GRANT ALL ON telegraf TO telegraf
```

Set a rentention policy named "Two_Weeks" for db telegraf, set it to 14 days and make it the default policy:
```sql
CREATE RETENTION POLICY Two_Weeks ON telegraf DURATION 14d REPLICATION 1 DEFAULT
```

Sanity checks to show that the db, user, and rentention ploicy were created:
```sql
show databases
show users
SHOW RETENTION POLICIES ON telegraf
```

----------------------------------------------------------------------------------------------------
### **Install & Configure Telegraf**
See [Telegraf documentation](https://docs.influxdata.com/telegraf/v1.9/introduction/installation/) for changes or newer versions.


Default config file: `/etc/telegraf/telegraf.conf`

Install Telegraf 1.9.2 (current version at the time), and start as a network service at boot:

```bash
wget https://dl.influxdata.com/telegraf/releases/telegraf_1.9.2-1_amd64.deb
sudo dpkg -i telegraf_1.9.2-1_amd64.deb
sudo systemctl start telegraf
sudo systemctl enable telegraf
```

##### Create Telegraf Configuration File
If `/etc/telegraf/telegraf.conf` already exist make sure at least these options are uncommented and updated to your appropriate settings. Other wise create the file and paste these settings in.

```
[agent]
  hostname = "nameofyourgrafanaserver"
  flush_interval = "15s"
  interval = "15s"

  [[inputs.cpu]]

  [[inputs.mem]]

  [[inputs.system]]

  [[inputs.disk]]
    mount_points = ["/"]

  [[inputs.processes]]

  [[inputs.net]]
    fieldpass = [ "bytes_*" ]

[[outputs.influxdb]]
  database = "telegraf"
  urls = [ "http://127.0.0.1:8086" ]
  username = "telegraf"
  password = "password"
```

After updating the config file, always restart Telegraf:
```bash
sudo systemctl restart telegraf
```

To test that Telegraf is setup correct:
```bash
#path_to_telegraf -test -config /path_to_telegraf.conf
telegraf -test -config /etc/telegraf/telegraf.conf
```


----------------------------------------------------------------------------------------------------
### **Install & Configure Grafana**
See [Grafan documentation](http://docs.grafana.org/installation/) for changes or newer versions.

Default config file: `/etc/grafana/grafana.ini`

Default log file: `/var/log/grafana`

Install Grafana 5.4.3 (current version at the time), and set as a network serivce at boot.
```bash
wget https://dl.grafana.com/oss/release/grafana_5.4.3_amd64.deb
sudo apt-get install -y adduser libfontconfig
sudo dpkg -i grafana_5.4.3_amd64.deb
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service
```

**Logging in for the first time:**

To run Grafana open your browser and go to http://localhost:3000/. 3000 is the default http port that Grafana listens to if you haven’t configured a different port. The defaults login ia admin/admin


----------------------------------------------------------------------------------------------------
### **Configure SSL With Self-Signed Certs For Grafana**

Steps to enable SSL for Grafana. Change to the grafana config directory and create certs:

```bash
cd /etc/grafana
sudo openssl req -x509 -newkey rsa:2048 -keyout grafana-key.pem -out grafana-cert.pem -days 3650 -nodes
```

After creating the .pem files. Change the mode and owner:
```bash
sudo chmod 644 grafana-key.pem
sudo chmod 644 grafana-cert.pem
sudo chown root grafana-key.pem
sudo chown root grafana-cert.pem
```

Update /etc/grafana/grafana.ini with these options:
```bash
# Protocol (http, https, socket)
protocol = https

# https certs & key file
cert_file =/etc/grafana/grafana-cert.pem
cert_key =/etc/grafana/grafana-key.pem
```


Restart Grafana:
```bash
systemctl restart grafana-server
```

You should be able to access Grafana on https://localhost:3000/


----------------------------------------------------------------------------------------------------
### Setup Influxdb With Self-Signed Certs:

Create the certs:
```bash
sudo openssl req -x509 -nodes -newkey rsa:2048 -keyout /etc/ssl/influxdb-selfsigned.key -out /etc/ssl/influxdb-selfsigned.crt -days 3650
```

After creating the .pem files. Change the mode and owner:
```bash
sudo chmod 644 /etc/ssl/influxdb-selfsigned.crt
sudo chmod 644 /etc/ssl/influxdb-selfsigned.key
sudo chown root /etc/ssl/influxdb-selfsigned.crt
sudo chown root /etc/ssl/influxdb-selfsigned.key
```

Update the influxdb.conf file:
```bash
[http]

  # Determines whether HTTPS is enabled.
  https-enabled = true

  # The SSL certificate to use when HTTPS is enabled.
  https-certificate = "/etc/ssl/influxdb-selfsigned.crt"

  # Use a separate private key location.
  https-private-key = "/etc/ssl/influxdb-selfsigned.key"
```

Restart influxdb:
```bash
sudo systemctl restart influxdb
```

You also need to update Telegraf so it knows to use ssl when sending to influxdb. Update the telegraf.config file.
```bash
sudo nano /etc/telegraf/telegraf.conf
```

Update these settings:
```bash
[[outputs.influxdb]]
  urls = ["https://127.0.0.1:8086"]

  ## Optional TLS Config for use on HTTP connections.
  insecure_skip_verify = true

  ## HTTP Basic Auth
  username = "telegraf"
  password = "password"
```


Log into Grafana web and update the influxdb datasoure:
- change url to https://localhost:8086
- check box for `Skip TLS Verify`
