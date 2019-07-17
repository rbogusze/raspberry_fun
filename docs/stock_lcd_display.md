
![RaspPi](../images/lcd10.jpg?raw=true "Title") ![RaspPi](../images/lcd11.jpg?raw=true "Title")

![RaspPi](../images/lcd12.jpg?raw=true "Title") ![RaspPi](../images/lcd13.jpg?raw=true "Title")

# Motivation

This project started when my kid (8 years old) started to getting some "serious" money at various occasions, like 100$ or more on birthday. It was too much to let him spend it all on Pepsi and Milky-way's and he already had a need to save it to make some large purchases like big Lego set. First I just said I will invest it for him but then he was asking everyday how much it grew today and was upset when it was falling down. I did not want to be the one telling bad news and was just tired of checking. I needed something that will tell him how much he owns and what was the gain/loss today. 

Raspberry pi with small LCD display seemed perfect for the job.

Technology choices.

1. For display I picked LCD 2x16 HD44780 with I2C

This is cheep, well known arduino display but with I2C interface to make it super easy to connect it to Raspberry Pi

2. Code will be written in python

It is just a pleasure to write in python. I have done too many bash scripts, time to go the python way.

3. Database to store the data will be Cassandra

This is not an obvious choice, but actually fits well into the requirements. I have been working for a long time as an Oracle DBA but realized there it more in world to explore if you like data and right now I am trying new things. MySQL / MariaDB would be a fine choice here as well as a light general purpose relational database, but Cassandra as a NoSQL database fits the picture nicely as well. Especially if you have couple of Raspberry Pis already running in the house - it is very easy to turn them into Cassandra cluster with data replication happening magically between them :)

Ok, let's start the fun.

# Raspberry Pi setup

![RaspPi](../images/lcd5.jpg?raw=true "Title")

Downloaded Raspbian Buster Lite from https://www.raspberrypi.org/downloads/raspbian/ burn on SD card and run RaspberryPi with it.

```
# dd if=2019-07-10-raspbian-buster-lite.img | pv -s 8G | dd of=/dev/mmcblk0 bs=4M
```

First time login as pi:raspberry (after you connect monitor and keyboard to raspberry)

Enable ssh server

Login as pi
```
$ sudo raspi-config
-> 5 Interfacing Options
-> P2 SSH -> Yes
```

Know your IP
```
$ ip addr show
(...)
inet 192.168.1.20/24
```
Now you can login remotely to your raspberry
```
$ ssh -l pi 192.168.1.20
```

## Connect LCD display

I followed the https://www.youtube.com/watch?v=fR5XhHYzUK0

```
$ cd
$ sudo apt install -y git
$ git clone https://github.com/the-raspberry-pi-guy/lcd.git
$ cd lcd/
$ sudo sh install.sh
(at the end RPi will reboot, just relogin after 1min)
$ cd lcd/
$ python demo_lcd.py
Writing to display
(and you should see the display working)
```
![RaspPi](../images/lcd7.jpg?raw=true "Title")


In case you just see an error like

```
$ python demo_lcd.py
Traceback (most recent call last):
  File "demo_lcd.py", line 11, in <module>
    display = lcddriver.lcd()
  File "/home/pi/lcd/lcddriver.py", line 74, in __init__
    self.lcd_write(0x03)
  File "/home/pi/lcd/lcddriver.py", line 98, in lcd_write
    self.lcd_write_four_bits(mode | (cmd & 0xF0))
  File "/home/pi/lcd/lcddriver.py", line 93, in lcd_write_four_bits
    self.lcd_device.write_cmd(data | LCD_BACKLIGHT)
  File "/home/pi/lcd/i2c_lib.py", line 11, in write_cmd
    self.bus.write_byte(self.addr, cmd)
IOError: [Errno 121] Remote I/O error
```

Change the ADDRESS variable in
```
$ vi lcddriver.py
#ADDRESS = 0x27
# This is another common LCD address.
ADDRESS = 0x3f
```

That depends on the RPi model which one is correct.

## Install Cassandra

Following http://ubeda.edu.umh.es/2014/12/31/installing-cassandra-2-1-on-raspberry-pi/

Cassandra is a very interesting NoSQL database to use. Simple yet very fast for storing key-value data with data replication written into the core of the solution. It is written in java, so we first need to have JDK installed.

Download jdk-8u191-linux-arm32-vfp-hflt.tar.gz from https://www.oracle.com/

```
$ md5sum jdk-8u191-linux-arm32-vfp-hflt.tar.gz 
7aa06029bc92016163bf07fcd62f0b57  jdk-8u191-linux-arm32-vfp-hflt.tar.gz
$ sudo su
# tar xvzf jdk-8u191-linux-arm32-vfp-hflt.tar.gz -C /opt
# update-alternatives --install /usr/bin/java java /opt/jdk1.8.0_191/bin/java 1
# update-alternatives --install /usr/bin/javac javac /opt/jdk1.8.0_191/bin/javac 1
# update-alternatives --set java /opt/jdk1.8.0_191/bin/java
# update-alternatives --display java

```

Set env variables
```
# vi /root/.bashrc
export JAVA_HOME="/opt/jdk1.8.0_191"
export PATH=$PATH:$JAVA_HOME/bin
# exit
$ sudo su
# env | grep JAVA
JAVA_HOME=/opt/jdk1.8.0_191
```

Download and install Cassandra
```
# cd /opt
# wget https://archive.apache.org/dist/cassandra/2.2.13/apache-cassandra-2.2.13-bin.tar.gz
# md5sum apache-cassandra-2.2.13-bin.tar.gz 
a522356a8db4c854a8a82e9f67e308a2  apache-cassandra-2.2.13-bin.tar.gz
# tar xvzf apache-cassandra-2.2.13-bin.tar.gz
# mv apache-cassandra-2.2.13 /opt/cassandra
```

Make some configuration changes, make sure to use your IP address. 
```
# cp /opt/cassandra/conf/cassandra.yaml /opt/cassandra/conf/cassandra.yaml_`date -I`
# vi /opt/cassandra/conf/cassandra.yaml
cluster_name: 'rpi_fun'
listen_address: 192.168.1.20
rpc_address: 192.168.1.20
- seeds: "192.168.1.20"
endpoint_snitch: GossipingPropertyFileSnitch
```

Create service for running Cassandra, so that it can be automatically started on system boot.
```
# vi /lib/systemd/system/cassandra.service
[Unit]
Description=RPi's cassandra
After=multi-user.target

[Service]
Type=idle
WorkingDirectory=/opt/cassandra/bin
ExecStart=/opt/cassandra/bin/cassandra -f
Restart=on-failure
Environment="JAVA_HOME=/opt/jdk1.8.0_191"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/jdk1.8.0_191/bin"

[Install]
WantedBy=multi-user.target
```

Start Cassandra
```
# chmod 644 /lib/systemd/system/cassandra.service
# systemctl daemon-reload
# systemctl enable cassandra.service
# systemctl start cassandra.service
# systemctl status cassandra.service
```

Check if you can connect to it
```
# /opt/cassandra/bin/nodetool status
--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.1.20  96.99 KB   256          100.0%            7c54619e-483e-40e9-89db-00421059144e  rack1
# /opt/cassandra/bin/cqlsh 192.168.1.20
cqlsh> desc keyspaces
system_traces  system_auth  system  system_distributed
```

## Create data model

Create keyspace (something like a database in mysql) and tables.
```
# /opt/cassandra/bin/cqlsh 192.168.1.20
cqlsh> CREATE KEYSPACE "stock" WITH replication = {'class' : 'NetworkTopologyStrategy','dc1' : 1};
cqlsh> CREATE TABLE stock.transactions (
asset text, 
tran_date date, 
amount float, 
price float,
note text,
PRIMARY KEY (asset, tran_date)
);

cqlsh> CREATE TABLE stock.quotes (
asset text, 
quote_date date, 
price float,
PRIMARY KEY (asset, quote_date)
);

cqlsh> CREATE TABLE stock.savings (
influx_date date, 
amount float,
note text, 
PRIMARY KEY (influx_date)
);

```

Let's populate it with some data

```
# /opt/cassandra/bin/cqlsh 192.168.1.20
> use stock;
> INSERT INTO transactions (asset, tran_date, amount, price, note) values ('mbk', '2019-07-08', 2, 404, '1/3 from birthday money');
> INSERT INTO stock.quotes (asset, quote_date, price) values ('mbk', '2019-07-07', 408);
> INSERT INTO stock.savings (influx_date, amount, note) values ('2019-07-08', 477, 'birthday money in savings');

> select * from stock.transactions;
> select * from stock.quotes;
> select * from stock.savings;

```




## Glue it all together

Install required packages (this can take a while, around 1-2h so better to shutdown Cassandra as it takes a lot of memory)
```
# systemctl stop cassandra.service
# apt install -y subversion python-pip build-essential python-dev git python-rpi.gpio python-dateutil
# pip install cassandra-driver kafka-python
# systemctl start cassandra.service
```


Check out repository with scripts (that is a lot of various stuff but we will use only small part of it)
```
$ cd
$ svn checkout https://github.com/rbogusze/oracleinfrastructure/trunk/scripto
$ cd scripto/python/stock/
```

Just test if we can connect to Cassandra from python.
```
$ python test_cassandra.py 
mbk 2019-07-08 2.0
```

Configure correct Cassandra cluster IP
```
$ vi stock.py
cluster = Cluster(contact_points=['192.168.1.20'] (leave the rest of line like it is)
$ python stock.py

```

And you should see the LCD working now and computing current value.

![RaspPi](../images/lcd8.jpg?raw=true "Title") ![RaspPi](../images/lcd9.jpg?raw=true "Title")

Several notes:
- display is turned off in the night to avoid disruption in case it is installed in kids room (like in my case)
- to get the current stock readings I use stooq.pl page with very simple text parsing, I am fine with daily accuracy. It has reading for the polish stock market (where I live and invest) but it should be trivial to adjust it to other sites





