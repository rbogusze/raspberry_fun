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

[Flink](flink.md)

[Cassandra](cassandra.md)

[AWS IoT](aws_iot.md)

