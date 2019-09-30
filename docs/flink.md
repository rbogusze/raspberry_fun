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






