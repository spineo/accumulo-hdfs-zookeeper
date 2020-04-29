# accumulo-hdfs-zookeeper
Create a storage cluster on AWS running Accumulo on Hadoop DFS (HDFS) and Zookeeper for node management.

In the first part of this project we will set up a 3-node cluster to install, configure, and run Accumulo, HDFS, and Zookeeper on each node and of course, have each application communicate with each other. In doing so, we will leverage an earlier project (https://github.com/spineo/hadoop-app) I created involving 3 AWS instances (Amazon Linux t2.micro 64-bit (x86)) which we will upgrade to t2.large to cover the additional resource utilization.

In the second part of the project we will develop a simple Java client to interact directly with Accumulo/HDFS.
