# (WIP) Motivation and roadmap

Gathering information like temperature is a perfect case to have a lot of fun with, use different technologies just to see if they fit the picture well and provide something that is cool to watch.

I can see the scenario as follows:
1. gather CPU temperature that is already available in Raspberry Pi
2. store it in good old RDBMS MySQL
3. store it in Cassandra database that was already used in [Stock LCD display](docs/stock_lcd_display.md)
4. store it in Kafka, as messagin system like that is perfect for such job and it is just a beatiful concept to decouple producers of data from consumers of data
5. store it in good old RDBMS MySQL, but this time the script that gathers the temperature will not even know about that - we will use Kafka for that, as the data is already there
6. display it on Grafana, just to have a nice view

Even more fun with data
1. configure Kafka to store the data "since the beginning" which will allow us to build external systems for analysis like Hadoop
2. play with Kafka and see how we can make it break/fix/scale

# First things first

[Monitoring and benchmarking](monitoring.md)

[MySQL](mysql.md)

[Kafka](kafka.md)


# Apache Flink

## Apache Flink Installation
Nice links:
- https://ci.apache.org/projects/flink/flink-docs-release-1.9/getting-started/tutorials/local_setup.html


(testing on consumer1)

cd /opt
wget http://ftp.man.poznan.pl/apache/flink/flink-1.9.0/flink-1.9.0-bin-scala_2.12.tgz
md5sum flink-1.9.0-bin-scala_2.12.tgz 
90aa35b57f0861c0793dfb048a1c2d5e  flink-1.9.0-bin-scala_2.12.tgz
tar xvzf flink-1.9.0-bin-scala_2.12.tgz
cd /opt/flink-1.9.0


vi bin/taskmanager.sh
TM_MAX_OFFHEAP_SIZE="8388607T"
TM_MAX_OFFHEAP_SIZE="10000000"

./bin/start-cluster.sh
http://consumer1:8081 






## Troubleshooting
### Error: Could not create the Java Virtual Machine.
Invalid maximum direct memory size: -XX:MaxDirectMemorySize=8388607T
The specified size exceeds the maximum representable size.
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.

too little RAM apparently

./bin/stop-cluster.sh
vi 


?? export FLINK_HEAP_OPTS="-Xmx200M -Xms200M"

2019-09-30 08:27:56,807 ERROR org.apache.flink.runtime.taskexecutor.TaskManagerRunner       - TaskManager initial
ization failed.
java.lang.OutOfMemoryError: Could not allocate enough memory segments for NetworkBufferPool (required (Mb): 9, al
located (Mb): 9, missing (Mb): 0). Cause: Direct buffer memory

vi conf/flink-conf.yaml

taskmanager.network.memory.min: 12mb
taskmanager.network.memory.max: 64mb
java.lang.OutOfMemoryError: Could not allocate enough memory segments for NetworkBufferPool (required (Mb): 12, allocated (Mb): 9, missing (Mb): 3). Cause: Direct buffer memory

-> ok, so this is it

When increasing
jobmanager.heap.size: 300m
taskmanager.heap.size: 300m


java.lang.OutOfMemoryError: Could not allocate enough memory segments for NetworkBufferPool (required (Mb): 29, allocated (Mb): 9, missing (Mb): 20). Cause: Direct buffer memory

java.lang.OutOfMemoryError: Could not allocate enough memory segments for NetworkBufferPool (required (Mb): 64, allocated (Mb): 9, missing (Mb): 55). Cause: Direct buffer memory

https://dzone.com/articles/troubleshooting-problems-with-native-off-heap-memo

what helped is to leave #jobmanager.heap.size and #taskmanager.heap.size to theyr apparent defaults of 1G






# Cassandra (WIP)

First I will be using the same Raspberry Pi that was used for mysql just to compare apples to apples.

Shutdown mysql

```
# systemctl stop mariadb
# systemctl disable mariadb
# systemctl stop mysql
# systemctl disable mysql
# systemctl stop mysqld_exporter
# systemctl disable mysqld_exporter
```

Install cassandra as described in stock_lcd_display.md

    # systemctl start cassandra.service
	
Wait few minutes and check if cassandra is up

    # /opt/cassandra/bin/nodetool status 

On producer hosts install cassandra-driver

    # pip install cassandra-driver


Create keyspace (something like a database in mysql) and tables.
```
# /opt/cassandra/bin/cqlsh 192.168.1.90
CREATE KEYSPACE "temperature" WITH replication = {'class' : 'NetworkTopologyStrategy','dc1' : 1};

CREATE TABLE temperature.reading (
    reading_location text,
    reading_date timestamp,
    reading_note text,
    reading_value float,
    PRIMARY KEY (reading_location, reading_date)
);
```
Configure correct Cassandra cluster IP

	$ cd ~/scripto/python/temperature
	$ vi collect.py
	cluster = Cluster(contact_points=['192.168.1.90'] (leave the rest of line like it is)

## Testing two producers -> Cassandra

First let's just what can we get from Cassandra.

One thread on RPi2mB and see how it compares to other 

	$ python collect.py --backend cassandra
	
| backend   | TPS | consumer          | producer4 |
|-----------|-----|-------------------|-----------|
| Cassandra | 128 | CPU 30%, IO 13ms  | CPU 18%   | 
| MySql     | 238 | CPU 21%, IO 320ms | CPU 11%   |
| Kafka     | 147 | CPU 9%, IO 4ms    | CPU 25%   | 

Ok, let's check how far we can go with the score this time.

	$ ./multi_collect.sh --backend cassandra
	
| TPS                    | consumer          | producer2,4 | bottleneck   |
|------------------------|-------------------|-------------|--------------|
| 8 * 70 + 8 * 60 = 1040 | CPU 100%, IO 22ms | CPU 42%,62% | consumer CPU |

Actually it just takes two powerful producers, producer2 which is RPi3 and producer4 which is RPi2 to make the Cassandra reach 100% CPU utilisation. Comparing it with MySQL (2840TPS) it scores rather poorly which leads to maybe a surprising conclusion - if we were to have just one server then running it with MySQL gives much better results that NoSQL Cassandra database. However, one node Cassandra cluster is not really where it shines, as it is born to be a part of a multi node cluster, contrary to RDBMS MySQL which is usually a singleton.

RDBMS database do not scale well horisontally, contrary to NoSQL databases which were designed that way. In theory adding second node to the Cassandra cluster should double the performance. Can't wait to see this happening.

## Adding second Cassandra node

The confuguration is the same as it was with one node, only seeds parameter stays the same.

Start the service

	# systemctl start cassandra.service
	
Wait few minutes and check if cassandra is up

    # /opt/cassandra/bin/nodetool status 

We should see both nodes up

	Status=Up/Down
	--  Address       Load       Tokens       Owns (effective)  Host ID Rack
	UN  192.168.1.90  237.8 KB   256          49.1%             40d5e7  rack1
	UN  192.168.1.20  195.7 KB   256          50.9%             bf7b25  rack1

Ok, let's first test exactly how we tested last time.

| producer   | TPS avg | total | CPU | bottleneck                 |
|------------|---------|-------|-----|----------------------------|
| 2. RPi3mB+ | 8 * 81  | 648   | 48% | consumer, as I can do more |
| 3. RPi2mB  | 8 * 63  | 504   | 67% | consumer, as I can do more |
|            |         | 1152  |     |                            |


| consumer                      | CPU, IO   | bottleneck       |
| mysql(just host name), RPi2mB | 89%, 25ms | clearly that one |
| cassandra2, RPi3mB2           | 50%, 1ms  | not this one     |

Hm, this is not what I was expecting, let's have a closer looks why this is happening.

First, think for a moment why we did not have to add additional IP for the second node in client configuration? That is actually how Cassandra works, client can connect only to some nodes and receive the information about all nodes in the cluster. We could add it of course, but this is not necessary to be able to work with other nodes in the cluster.

Actually the oposite happened. We can see second node doing something, but that does not bring us any performance gains.

Looking at the space consumed by the temperature table gives us some hints.

First node

	root@mysql:/opt/cassandra/data/data# du -sh temperature/
	120K	temperature/

Second node

	root@cassandra2:/opt/cassandra/data/data# du -sh temperature/
	12K	temperature/

That is not spread equally, what is even more interesting is that on second node there is no data at all. Checking how the keyspaces is configured:

	CREATE KEYSPACE "temperature" WITH replication = {'class' : 'NetworkTopologyStrategy','dc1' : 1};
	
The last `1` indicates the replication factor, meaning in our case that data should be spread through all nodes in the cluster without additional replicas. We have run the test after the keyspace was created, so now data should arrive at all the cluster nodes. Even forcing a repair operation that should re-balance already present data does not change anything (run on all nodes one after another)

	# /opt/cassandra/bin/nodetool repair -full

Something stops the data to spread evenly through all cluster nodes. 

There are nice articles about that:
- https://www.datastax.com/dev/blog/basic-rules-of-cassandra-data-modeling
- https://www.datastax.com/dev/blog/the-most-important-thing-to-know-in-cassandra-data-modeling-the-primary-key
- http://intellidzine.blogspot.com/2014/01/cassandra-data-modelling-primary-keys.html
- http://wiki.apache.org/cassandra/Operations

The main points:
- Rows are spread around the cluster based on a hash of the partition key

Looking at how the table syntax

```
CREATE TABLE temperature.reading (
    reading_location text,
    reading_date timestamp,
    reading_note text,
    reading_value float,
    PRIMARY KEY (reading_location, reading_date)
);
```

The trick is that only `reading_location` is a partition key, `reading_date` is a cluster key. So far we were generating data only from `producer2` and `producer4` so with just a little bit of luck we end up in the same partition token that belongs to the same node. 

Let's include the date as part of partition key that should make it spread more evenly between the nodes.

```
# /opt/cassandra/bin/cqlsh mysql.home
> drop table temperature.reading ;
> CREATE TABLE temperature.reading (
    reading_location text,
    reading_date timestamp,
    reading_note text,
    reading_value float,
    PRIMARY KEY ((reading_location, reading_date))
);
```

Running the test again and to my surprise that did not change much. I can see still uneven CPU load on Cassandra nodes. What is good is that I finally see the data spread evenly between the nodes, but still the load is not even. Similarily as before, node1 92% CPU, node2 56%.

Clearly the bootleneck is the CPU on consumers, but what I overlooked initially is that the nodes that make the Cassandra cluster are of different type. Those are RPi2mB and RPi3mB2. The way Cassandra operates is to first decide to which node the insert should go to and by default assumes that every node is the same - as this is generally the good advise when creating a cluster. But there is hope - each node receives range of tokens that make it the go-to node for particular primary key values. For situation like that we can configure stronger nodes to accept more of them that will increase the load for that node.

Let's change this parameter on RPi3mB2 so that it can accept more load than the RPi2mB.


	$ /opt/cassandra/bin/nodetool decommission
	$ sudo systemctl stop cassandra.service
	$ vi /opt/cassandra/conf/cassandra.yaml
	num_tokens: 512
	$ rm -Rf /opt/cassandra/data/commitlog/*
	$ rm -Rf /opt/cassandra/data/data/*
	$ rm -Rf /opt/cassandra/data/hints/*
	$ rm -Rf /opt/cassandra/data/saved_caches/*
	$ sudo systemctl start cassandra.service


Hm, still uneven load despite:

--  Address       Load       Tokens       Owns (effective)  Host ID                               Rack
UN  192.168.1.90  29.28 MiB  256          32.6%             f087e2f4-c49e-4052-b6b2-db50eac54925  rack1
UN  192.168.1.20  74.6 KiB   512          67.4%             68ae8d45-ad41-4de4-a7d7-555719970ef7  rack1


Ok, removed data from both nodes
-> still the same, 41TPCx20, 95% node1, 65% cassandra2

Let's try to use one node only - cassandra2

CPU 95% producer2: 47TPS * 20 = 940
CPU 88% cassandra2


CPU 69% producer2: 40TPS * 20 = 800
CPU 46% producer3: 40TPS * 8 = 320, total: 1120
CPU 100% cassandra2


CPU 70% producer2: 37TPS * 20 = 740
CPU 47% producer3: 37TPS * 8 = 296, total 1036
CPU 94% mysql 128
CPU 81% cassandra2 512
- after joining the cluster, I can't make first node to take some of the data from the second node (because bootstraping does not happen for seed nodes, but why repair does not work?)
UN  192.168.1.90  68.02 KiB  128          20.8%             6ff9db27-dc46-454d-a9b0-fc3e88bec51c  rack1
UN  192.168.1.20  32.47 MiB  512          79.2%             9c68cb48-e868-4aa8-b700-26422d3551ca  rack1
- a little bit lower than with 1 node



Nice links:
- http://abiasforaction.net/unbalanced-cassandra-cluster/






What we learned so far:
1. One should pay attention to primary key, as this is how rows are spread around the cluster
2. Having different hardware on nodes is an anti-pattern and something that can make one week node to hinder the performance of the whole cluster






# AWS IoT

To setup the AWS part:

Following: IoT (Internet Of Things) Getting Started - Part 1 https://www.youtube.com/watch?v=sq_l2J4oyLU

Basically this is what needs to be done:

Create a thing: producer1
Create a certificate, download three files + root CA to `~/_poufne/tasks/aws_iot/producer1`
  https://www.amazontrust.com/repository/AmazonRootCA1.pem
  https://www.amazontrust.com/repository/AmazonRootCA3.pem

  Actions -> Attach thing
Create a policy
  Name: ProducersPolicy, Action: iot:* , Resource ARN: *
Certificate, (recently created one) Actions -> Attach policy

Where to find REST API endpoint? Things -> producer1 -> Interact
Rest API Endpoint: xxxxxxxxxxx-ats.iot.us-west-2.amazonaws.com


Copy the certificates to producers, like:

	$ scp * pi@producer2:~/certs

Change a bit the name so that I can easily reference them in the script

	$ cd ~/certs
	$ mv *-certificate.pem.crt certificate.pem.crt
	$ mv *-private.pem.key private.pem.key
	$ mv *-public.pem.key public.pem.key
	$ wget https://www.amazontrust.com/repository/AmazonRootCA1.pem
	$ mv AmazonRootCA1.pem rootCA.pem

Install libraries for python

	# pip install paho-mqtt
	
Run it first with one thread providing as broker the address that you see in AWS as Rest API Endpoint.

	$ python collect.py --backend=awsiot --broker=xxxxxxxxxxxxxx-ats.iot.eu-west-1.amazonaws.com

One thread on RPi2mB and see how it compares to other. 

| backend   | TPS | consumer          | producer4 |
|-----------|-----|-------------------|-----------|
| AWS IoT   | 740 | ?                 | CPU 25%   |
| Cassandra | 128 | CPU 30%, IO 13ms  | CPU 18%   |
| MySql     | 238 | CPU 21%, IO 320ms | CPU 11%   |
| Kafka     | 147 | CPU 9%, IO 4ms    | CPU 25%   |

WOW, that is something. Let's see how far we can go with all threads and all producers.

	$ ./multi_collect.sh --backend=awsiot --broker=xxxxxxxxxxxxxx-ats.iot.eu-west-1.amazonaws.com 

| producer   | threads | TPS avg / thread | total | CPU  | bottleneck |
|------------|---------|------------------|-------|------|------------|
| 1. RPi1    | 2       | 182              | 364   | 100% | producer   |
| 2. RPi3mB+ | 8       | 1670             | 13360 | 100% | producer   |
| 3. RPi2mB  | 8       | 712              | 5696  | 100% | producer   |
| 4. RPi2mB  | 8       | 707              | 5656  | 100% | producer   |
|            |         |                  | 25076 |      |            |

Wow again. I was able to produce 25076 messages a second, so with default 3min of this test running I produced 7,5mil messaged. Nice.









# Infrastructure

## Producers setup



## Consumers setup

### RPi custom image

Nice links:
- ? https://medium.com/platformer-blog/creating-a-custom-raspbian-os-image-for-production-3fcb43ff3630
- ? https://learnaddict.com/2016/02/23/modifying-a-raspberry-pi-raspbian-image-on-linux/
