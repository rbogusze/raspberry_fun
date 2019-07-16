# Motivation

This project started when my kid (8 years old) started to getting some "serious" money at various occasions, like 100$ or more on birthday. It was too much to let him spend it all on Pepsi and Milky-way's and he already had a need to save it to make some large purchases like big Lego set. First I just said I will invest it for him but then he was asking everyday how much it grew today and was upset when it was falling down. I did not want to be the one telling bad news and was just tired of checking. I needed something that will tell him how much he owns and what was the gain/loss today. 

Raspberry pi with small LCD display seemed perfect for the job.

Technology choises.

1. For display I picked LCD 2x16 HD44780 with I2C

This is cheep, well known arduino display but with I2C interface to make it super easy to connect it to Raspberry Pi

2. Code will be written in python

It is just a pleasure to write in python. I have done too many bash scripts, time to go the python way.

3. Database to store the data will be Cassandra

This is not an obvious choise, but actually fits well into the requirements. I have been working for a long time as an Oracle DBA but realized there it more in world to explore if you like data and right now I am trying new things. MySQL / MariaDB would be a fine choise here as well as a light general purpose relational database, but Cassandra as a NoSQL database fits the picture nicely as well. Especially if you have couple of Raspberry Pis already running in the house - it is very easy to turn them into Cassandra cluster with data replication happening magically between them :)

