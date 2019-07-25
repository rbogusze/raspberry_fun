# (WIP) Motivation and roadmap

192.168.1.99 kafka 
192.168.1.20 monitoring 
192.168.1.90 mysql
192.168.1.97 producer1
192.168.1.29 producer2
192.168.1.101 producer2

Gathering information like temperature is a perfect case to have a lot of fun with, use different technologies just to see if they fit the picture well and provide something that is cool to watch.

I can see the scenario as follows:
1. gather CPU temperature that is already available in Raspberry Pi
2. store it in Cassandra database that was already used in [Stock LCD display](docs/stock_lcd_display.md)
3. store it in Kafka, as messagin system like that is perfect for such job and it just a beatiful concept to decouple producers of data from consumers of data
4. store it in old good RDBMS MySQL, but this time the script that gathers the temperature will not even know about that - we will use Kafka for that, as the data is already there
5. display it on Grafana, just to have a nice view

Even more fun with data
1. configure Kafka to store the data "since the beginning" which will allow us to build external systems for analysis like Hadoop
2. play with Kafka and see how we can make it brak/fix/scale

# First things first

## Install monitoring

I intend to use mysql to store the temperature data, which would be a no-brainer choise 10 years ago. Now we have more options in that area, but let's first try to do it in relational database like mysql.

Nice links:
- Graphing MySQL performance with Prometheus and Grafana https://www.percona.com/blog/2016/02/29/graphing-mysql-performance-with-prometheus-and-grafana/
- Monitoring – How to install Prometheus/Grafana on arm – Raspberry PI/Rock64 https://www.mytinydc.com/index.php/en/2019/04/14/monitoring-how-to-install-prometheus-grafana-on-arm-raspberry-pi-rock64-client-server/

Node exporter installation
```
# cd /opt
# wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-armv7.tar.gz
# md5sum node_exporter-0.18.1.linux-armv7.tar.gz
f9b28f4bfeecc4fe17b441098117359c  node_exporter-0.18.1.linux-armv7.tar.gz
# tar xvzf node_exporter-0.18.1.linux-armv7.tar.gz
# cd node_exporter-0.18.1.linux-armv7
# nohup ./node_exporter &
# cd /opt
# ln -s node_exporter-0.18.1.linux-armv7 node_exporter

or for Raspberry Pi 1
# wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-armv6.tar.gz
# md5sum node_exporter-0.18.1.linux-armv6.tar.gz 
83611c07f3728175b81f3631df12eba8  node_exporter-0.18.1.linux-armv6.tar.gz
# tar xvzf node_exporter-0.18.1.linux-armv6.tar.gz
# cd node_exporter-0.18.1.linux-armv6
# nohup ./node_exporter &
# cd /opt
# ln -s node_exporter-0.18.1.linux-armv6 node_exporter
```
http://monitoring:9100/metrics
-> you should see lot's of metrics

Make node_exporter to autostart on system boot
```
# vi /lib/systemd/system/node_exporter.service
[Unit]
Description=Prometheus Node Exporter
After=multi-user.target

[Service]
Type=idle
WorkingDirectory=/opt/node_exporter
ExecStart=/opt/node_exporter/node_exporter
Restart=on-failure

[Install]
WantedBy=multi-user.target

# chmod 644 /lib/systemd/system/node_exporter.service
# systemctl daemon-reload
# systemctl enable node_exporter.service
# systemctl start node_exporter.service
# systemctl status node_exporter.service
```


Prometheus server installation
```
# cd /opt
# wget https://github.com/prometheus/prometheus/releases/download/v2.11.1/prometheus-2.11.1.linux-armv7.tar.gz
# md5sum prometheus-2.11.1.linux-armv7.tar.gz
24c11873082f6c6be172b25e5b365576  prometheus-2.11.1.linux-armv7.tar.gz
# tar xvzf prometheus-2.11.1.linux-armv7.tar.gz
# cd prometheus-2.11.1.linux-armv7
# cp prometheus.yml prometheus.yml_`date -I`
# vi prometheus.yml
(does it make sense?)
# nohup ./prometheus &
# cd /opt
# ln -s prometheus-2.11.1.linux-armv7 prometheus

```
http://monitoring:9090/targets

Make Prometheus server to autostart on system boot
```
# vi /lib/systemd/system/prometheus.service
[Unit]
Description=Prometheus server
After=multi-user.target

[Service]
Type=idle
WorkingDirectory=/opt/prometheus
ExecStart=/opt/prometheus/prometheus
Restart=on-failure

[Install]
WantedBy=multi-user.target

# chmod 644 /lib/systemd/system/prometheus.service
# systemctl daemon-reload
# systemctl enable prometheus.service
# systemctl start prometheus.service
# systemctl status prometheus.service
```

To reload the new configuration, if we ever want to change it do it with
```
# systemctl restart prometheus.service
```

Grafana server installation

```
# cd /opt
# wget https://dl.grafana.com/oss/release/grafana_6.2.5_armhf.deb 
# dpkg -i grafana_6.2.5_armhf.deb 
(that will most likely fail because of some missing prerequisites, install them - and finish grafana installation - with the next command)
# apt -f install
# /bin/systemctl daemon-reload
# /bin/systemctl enable grafana-server
# /bin/systemctl start grafana-server
```
(wait 2min)
http://monitoring:3000
login as admin/admin

Go to 'Add data source'
-> Prometheus
-> http://192.168.1.20:9090
(Save & Test)

Import dashboard
(big plus sign) -> Import 
Grafana.com Dashboard: https://grafana.com/dashboards/1860
Prometheus -> Prometheus
-> Import

Add a client to the monitoring system

Now, we are monitoring the monitoring host with our monitoring solution, which is nice, but we would like to monitor other hosts as well. 

On every node that needs to be monitored we need to:
- install 'Node exporter' as above
- add new host to Prometheus configuration and restart it

## Benchmark

Nice links:
- https://askubuntu.com/questions/87035/how-to-check-hard-disk-performance
- 

First let's establish some hardware limits on the Raspberry Pis I will use. That will let us to see the bottlenecks.

```
# apt install -y sysbench nmon fio
# cd /opt
```

CPU test
```
# sysbench --test=cpu --num-threads=4 --cpu-max-prime=20000 run
```

IO Test, following advise from https://askubuntu.com/questions/87035/how-to-check-hard-disk-performance
```
Sequential READ 
# fio --name TEST --eta-newline=5s --filename=fio-tempfile.dat --rw=read --size=1500m --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=180 --group_reporting

Screenshot from 2019-07-22 13-51-14.png

Sequential WRITE
# fio --name TEST --eta-newline=5s --filename=fio-tempfile.dat --rw=write --size=1500m --io_size=10g --blocksize=1024k --ioengine=libaio --fsync=10000 --iodepth=32 --direct=1 --numjobs=1 --runtime=180 --group_reporting

Screenshot from 2019-07-22 13-53-44.png (to compare it with previous)


Random 4K read QD1 
# fio --name TEST --eta-newline=5s --filename=fio-tempfile.dat --rw=randread --size=500m --io_size=10g --blocksize=4k --ioengine=libaio --fsync=1 --iodepth=1 --direct=1 --numjobs=1 --runtime=180 --group_reporting

(must be from cache)
Screenshot from 2019-07-22 13-59-20.png

Mixed random 4K read and write QD1 with sync
# fio --name TEST --eta-newline=5s --filename=fio-tempfile.dat --rw=randrw --size=1500m --io_size=10g --blocksize=4k --ioengine=libaio --fsync=1 --iodepth=1 --direct=1 --numjobs=1 --runtime=180 --group_reporting

'Screenshot from 2019-07-22 13-46-11.png'

Screenshot from 2019-07-22 14-02-07.png (maybe this Disk R/W time is good indicator or IO bottleneck?)

Screenshot from 2019-07-22 14-04-24.png

```


|                    | RPi 2 Model B V1.1     |
| cpu total time:    | 148.6189s              |
| Sequential READ    | 22.7MB/s , 40IOPS      |
| Sequential WRITE   | 9608kB/s , 20IOPS      |
| Random 4K read QD1 | 5MB , 1200IOPS (must be cache) |
| Mixed random 4K    | R/W: 20.0kB/s , 10IOPS |

What is important from this test is that we can max out our SD card performance in two different way. First is a sequential throuput, where we can read around 20MB/s or write 10MB/s. Second is the number of random IO that can happen, and it is actually very poor with this SD card and can go up to around 30IOPS.

? The point is to be able to say that our SD card is the bottleneck if we experience 

? Important fact to take out of that is that with currect hardware (not so great SD card, will test it again on some solid one) we can either have sequential reads with some

? How to know if I have random or sequential IO right now

Network test

$ scp oko4.img pi@mysql:

Screenshot from 2019-07-22 14-18-26.png

I know that there some CPU overhead associated with scp, but in general it should show the expected performance.

|         | RPi 2 Model B V1.1 |
| ingress | up to 9MB/s       |

Now, as our SD card write performance is similar, it may be that I am seeing disk performance.

$ scp oko4.img pi@mysql:/dev/null

Ok, this one does not cause any IO activity, so we are just measuring network throughput and a bit of CPU.

|         | RPi 2 Model B V1.1 |
| ingress | up to 11MB/s       |

Screenshot from 2019-07-22 14-23-09.png

According to https://www.raspberrypi.org/products/raspberry-pi-2-model-b/
tested Raspberry Pi 2 Model B has 100 Base Ethernet, which is around what we see



## Install mysql + datamodel

```
# apt-get install -y mariadb-server
# mysql -u root 
> create database temperature;
CREATE TABLE temperature.reading (
    reading_location VARCHAR(20),
    reading_date date,
    reading_note VARCHAR(20),
    reading_value float
);

```

Setting up mysql metrics for Prometheus. 

Nice links:
- https://www.percona.com/blog/2016/02/29/graphing-mysql-performance-with-prometheus-and-grafana/

```
# cd /opt
# wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.12.0/mysqld_exporter-0.12.0.linux-armv7.tar.gz
# md5sum mysqld_exporter-0.12.0.linux-armv7.tar.gz 
36190ef34fe21f7c0bb4266567f3e19d  mysqld_exporter-0.12.0.linux-armv7.tar.gz
# tar xvzf mysqld_exporter-0.12.0.linux-armv7.tar.gz
# ln -s mysqld_exporter-0.12.0.linux-armv7 mysqld_exporter
```

mysqld_exporter wants MySQL credentials.
```
# mysql -u root 
> GRANT REPLICATION CLIENT, PROCESS ON *.* TO 'prom'@'localhost' identified by 'secret';
> GRANT SELECT ON performance_schema.* TO 'prom'@'localhost';
# cd /opt/mysqld_exporter
# cat << EOF > .my.cnf
[client]
user=prom
password=secret
EOF
# ./mysqld_exporter --config.my-cnf=".my.cnf"
```

http://192.168.1.90:9104

Make mysqld_exporter to autostart on system boot
```
# vi /lib/systemd/system/mysqld_exporter.service
[Unit]
Description=Prometheus Mysqld Exporter
After=multi-user.target

[Service]
Type=idle
WorkingDirectory=/opt/mysqld_exporter
ExecStart=/opt/mysqld_exporter/mysqld_exporter --config.my-cnf=".my.cnf"
Restart=on-failure

[Install]
WantedBy=multi-user.target

# chmod 644 /lib/systemd/system/mysqld_exporter.service
# systemctl daemon-reload
# systemctl enable mysqld_exporter.service
# systemctl start mysqld_exporter.service
# systemctl status mysqld_exporter.service
```

Add another target and job to Prometheus
```
# cd /opt/prometheus
# vi prometheus.yml
- job_name: mysql
    static_configs:
      - targets: ['192.168.1.90:9104']
        labels:
          alias: mysql1

```


## Install Kafkac

Download Kafka from https://kafka.apache.org/downloads
```
# cd /opt
# wget http://ftp.man.poznan.pl/apache/kafka/2.3.0/kafka_2.12-2.3.0.tgz
# sha512sum kafka_2.12-2.3.0.tgz 
a5ed591ab304a1f16f7fd64183871e38aabf814a2c1ca86bb3d064c83e85a6463d3c55f4d707a29fc3d7994dd7ba1f790b5a6219c6dffdf472afd99cee37892e  kafka_2.12-2.3.0.tgz
# tar xvzf kafka_2.12-2.3.0.tgz
# export PATH=$PATH:/opt/kafka_2.12-2.3.0/bin

# cd /opt/kafka_2.12-2.3.0
# bin/zookeeper-server-start.sh config/zookeeper.properties &

# cd /opt/kafka_2.12-2.3.0
# bin/kafka-server-start.sh config/server.properties &
```

Let's produce some messages




## Data model
We are still using Cassandra as the primary data backend.

Create keyspace (something like a database in mysql) and tables.
```
# /opt/cassandra/bin/cqlsh 192.168.1.20
CREATE KEYSPACE "temperature" WITH replication = {'class' : 'NetworkTopologyStrategy','dc1' : 1};

CREATE TABLE temperature.reading (
    reading_location text,
    reading_date date,
    reading_note text,
    reading_value float,
    PRIMARY KEY (reading_location, reading_date)
);
```
Configure correct Cassandra cluster IP
$ cd ~/scripto/python/temperature
$ vi collect.py
cluster = Cluster(contact_points=['192.168.1.20'] (leave the rest of line like it is)
