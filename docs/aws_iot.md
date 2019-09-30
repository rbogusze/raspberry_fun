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

