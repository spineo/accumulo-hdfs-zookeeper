# accumulo-hdfs-zookeeper

Create a storage cluster on AWS (or any other Cloud) running Apache Accumulo on Hadoop DFS (HDFS) and Zookeeper for node management.

In the first part of this project we will set up a 3-node cluster to install, configure, and run Accumulo, HDFS, and Zookeeper on each node and of course, have each application communicate with each other. In doing so, we will leverage an earlier project (https://github.com/spineo/hadoop-app) I created involving 3 AWS instances (Amazon Linux t2.micro 64-bit (x86)) which we will upgrade to t2.medium to cover the additional resource utilization.

In the second part of the project we will develop a simple Java client to interact directly with Accumulo/HDFS.

## Change the Instance Types

In the AWS console we will be selecting the _HadoopMainNode_ and then go to _Actions -> Instance Settings -> Change Instance Type_, select _t2.medium_, and click "Apply" (once we get ready to set up the applications on the remaining nodes we can similarly apply this action to the _HadoopDataNode1_ and _HadoopDataNode2_ instances). Note that the instance(s) must be stopped first before applying this action (I would generally recommend that instances be stopped when not in use as AWS charges can quickly skyrocket!)

## Install/Configure Zookeeper on the Main Node

We will start out by installing, configuring, and testing Zookeeper on the _HadoopMainNode_ and then complete the installation/configuration to include the remaining two nodes.


### Set up the User and Deploy the Application


Log into the main node and as _ec2-user_ (or any user with _sudo_ privileges) run the below commands to setup our 
_zookeeper_ user:
```
sudo useradd zookeeper -m
sudo usermod --shell /bin/bash zookeeper
sudo passwd zookeeper
```

Create the data directory by running below commands:
```
sudo mkdir -p /data/zookeeper
sudo chown zookeeper:zookeeper /data/zookeeper
```

We will now download/install the binary in the same location where we installed Hadoop (i.e., /var/applications) and will use the latest stable release from https://zookeeper.apache.org/releases.html#download by running below commands:
```
cd /var/applications
sudo su
wget https://downloads.apache.org/zookeeper/zookeeper-3.6.0/apache-zookeeper-3.6.0-bin.tar.gz
tar -xvf apache-zookeeper-3.6.0-bin.tar.gz
chown -R zookeeper:zookeeper apache-zookeeper-3.6.0-bin
ln -s apache-zookeeper-3.6.0-bin zookeeper
chown -h zookeeper:zookeeper zookeeper
```

Finally, we will confirm that our _java_ dependency being used is the correct one:
```
[zookeeper@ip-xxx-xxx-xxx-xxx zookeeper]$ java -version
openjdk version "1.8.0_222"
OpenJDK Runtime Environment (build 1.8.0_222-b10)
OpenJDK 64-Bit Server VM (build 25.222-b10, mixed mode)
```

### Configure the Application

Run the below commands as user _zookeeper_ (you can first run _export ZOOKEEPER_HOME=/var/applications/zookeeper_ or add this statement to your ~/.bashrc)
```
cd $ZOOKEEPER_HOME/conf
cp zoo_sample.cfg zoo.cfg
```
and edit the _zoo.cfg_ file to include the below parameters (all except _dataDir_ are likely default):
```
# The number of milliseconds of each tick
tickTime=2000

# The number of ticks that the initial 
# synchronization phase can take
initLimit=10

# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5

# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/data/zookeeper

# the port at which the clients will connect
clientPort=2181

# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
```

### Start the Application

Run the below commands:
```
cd $ZOOKEEPER_HOME
./bin/zkServer.sh start
```
and you should see output similar to below (in this case no need to specify our config file):
```
/bin/java
ZooKeeper JMX enabled by default
Using config: /var/applications/zookeeper/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```

We can now test connecting to our server running locally through the CLI (we can ignore most of the verbose output but if successful should see at the end a line containing _WatchedEvent state:SyncConnected_):
```
./bin/zkCli.sh -server 127.0.0.1:2181
```
If we type any command (i.e., help) it should list the available CLI commands and we can type _quit_ when done:
```
[zk: 127.0.0.1:2181(CONNECTED) 0] help
ZooKeeper -server host:port cmd args
	addWatch [-m mode] path # optional mode is one of [PERSISTENT, PERSISTENT_RECURSIVE] - default is PERSISTENT_RECURSIVE
	addauth scheme auth
	close 
	config [-c] [-w] [-s]
	connect host:port
	create [-s] [-e] [-c] [-t ttl] path [data] [acl]
	delete [-v version] path
	deleteall path [-b batch size]
	delquota [-n|-b] path
	get [-s] [-w] path
	getAcl [-s] path
	getAllChildrenNumber path
	getEphemerals path
	history 
	listquota path
	ls [-s] [-w] [-R] path
	printwatches on|off
	quit 
	reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
	redo cmdno
	removewatches path [-c|-d|-a] [-l]
	set [-s] [-v version] path data
	setAcl [-s] [-v version] [-R] path acl
	setquota -n|-b val path
	stat [-w] path
	sync path
	version 
```

Likewise, we can stop the server as we get ready to set up the _systemd_ configuration:
```
[zookeeper@ip-xxx-xxx-xxx-xxx zookeeper]$ ./bin/zkServer.sh stop
/bin/java
ZooKeeper JMX enabled by default
Using config: /var/applications/zookeeper/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
```

## Setting up _systemd_

Create a new file _/etc/systemd/system/zookeeper.service_ with the below contents:
```
[Unit]
Description=Zookeeper Daemon
Documentation=http://zookeeper.apache.org
Requires=network.target
After=network.target

[Service]    
Type=forking
WorkingDirectory=/opt/zookeeper
User=zookeeper
Group=zookeeper
ExecStart=/opt/zookeeper/bin/zkServer.sh start /opt/zookeeper/conf/zoo.cfg
ExecStop=/opt/zookeeper/bin/zkServer.sh stop /opt/zookeeper/conf/zoo.cfg
ExecReload=/opt/zookeeper/bin/zkServer.sh restart /opt/zookeeper/conf/zoo.cfg
TimeoutSec=30
Restart=on-failure

[Install]
WantedBy=default.target
```

Start the daemon by running _systemctl start zookeeper_ and then _systemctl status zookeeper_ prefixed with _sudo_ if not root (you can verify that daemon is running from status output):
```
● zookeeper.service - Zookeeper Daemon
   Loaded: loaded (/etc/systemd/system/zookeeper.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2020-04-30 16:20:15 UTC; 8s ago
     Docs: http://zookeeper.apache.org
  Process: 3884 ExecStart=/var/applications/zookeeper/bin/zkServer.sh start /var/applications/zookeeper/conf/zoo.cfg (code=exited, status=0/SUCCESS)
 Main PID: 3900 (java)
   CGroup: /system.slice/zookeeper.service
           └─3900 java -Dzookeeper.log.dir=/var/applications/zookeeper/bin/../logs -Dzookeeper.log.file=zookeeper-zookeeper-server-ip-xxx-xxx-xxx-xxx.ec2.internal.log -Dzookeeper.root.logger=IN...

Apr 30 16:20:14 ip-xxx-xxx-xxx-xxx.ec2.internal systemd[1]: Starting Zookeeper Daemon...
Apr 30 16:20:14 ip-xxx-xxx-xxx-xxx.ec2.internal zkServer.sh[3884]: /usr/bin/java
Apr 30 16:20:14 ip-xxx-xxx-xxx-xxx.ec2.internal zkServer.sh[3884]: ZooKeeper JMX enabled by default
Apr 30 16:20:14 ip-xxx-xxx-xxx-xxx.ec2.internal zkServer.sh[3884]: Using config: /var/applications/zookeeper/conf/zoo.cfg
Apr 30 16:20:15 ip-xxx-xxx-xxx-xxx.ec2.internal systemd[1]: Started Zookeeper Daemon.
```

Finally, run command to enable startup on boot:
```
[root@ip-xxx-xxx-xxx-xxx apache-zookeeper-3.6.0-bin]# systemctl enable zookeeper
Created symlink from /etc/systemd/system/default.target.wants/zookeeper.service to /etc/systemd/system/zookeeper.service.
```

## Set up the Zookeeper Cluster

## Install/Configure Accumulo

## Run the Test Application


## References
