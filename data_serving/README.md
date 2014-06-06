* Author: [Rui Jin](http://blog.jinrui.name/)
* Email: jinrui@jinrui.name

#Introduction
## Data Serving

The data serving benchmark relies on the Yahoo! Cloud Serving Benchmark (YCSB). YCSB is a framework to benchmark data store systems. This framework comes with the interfaces to populate and stress many popular data serving systems. Here we provide the instructions and pointers to download and install YCSB and use it with the Cassandra data store.

#Install
## Prerequisite Software Packages

1.[**Cassandra0.7.3**](http://archive.apache.org/dist/cassandra/0.7.3/apache-cassandra-0.7.3-bin.tar.gz)

2.YCSB 0.1.3 Clone from git hub, use: *git clone git://github.com/brianfrankcooper/YCSB.git* 

3.Apache ant and Java JDK(1.6) 4.Download [**YCSB**](http://parsa.epfl.ch/cloudsuite/software/dataserving.tar.gz) benchmark


## Preparing YCSB

**Build YCSB:** 

1.tar zxvf dataserving.tar.gz 

2.cd YCSB 

3.ant

**Build client:** 

1.tar zxvf apache-cassandra-0.7.3-bin.tar.gz 

2.cd apache-cassandra-0.7.3/lib 

3.cp *.jar /DIR_OF_YCSB/db/cassandra-0.7/lib/ 

4.cd /DIR_OF_YCSB/ 

5.ant 

6.ant dbcompile-cassandra-0.7


## Install Cassandra
**Running Cassandra on a single node:**

1.cd /DIR_OF_CASSANDRA/

2.ant

3.Check the configuration parameters: conf/cassandra.yaml contains default values for the Cassandra parameters. First, ensure that the paths for the following parameters point to the directories where you have the write permissions. 

4.data_file_directories, commitlog_directory, and saved_caches_directory.

5.In conf/log4j-server.properties, make sure that the parameter: log4.appender.R.Fileis set to the directories of your choice that you have write permissions to.

6.Set your JAVA_HOME environment variable properly.

7.Run Cassandra by invoking: sudo bin/cassandra -f

**Error you may meet:**
*The stack size specified is too small, Specify at least 160k
Could not create the Java virtual machine.*

To fixed this error,just open the file conf/cassandra-env.sh, change JVM_OPTS="$JVM_OPTS -Xss128k to a larger number , like JVM_OPTS="$JVM_OPTS -Xss228k. Then run again:sudo ./bin/cassandra -f

#Run
## Generating Dataset
1.cd /DIR_OF_CASSANDRA/

2.sudo ./bin/cassandra-cli

3.create keyspace usertable with replication_factor=1;

4.use usertable;

5.create column family data with column_type = 'Standard' and comparator = 'UTF8Type';

6.exit;

**Then:**

7.cd /DIR_OF_YCSB/

8.configure *settings_load.dat*
hosts: specifies the IP address of the machine running Cassandra.
recordcount: specifies the number of records to be loaded in the data store.

9.sudo ./run_load.command

##Running the benchmark
1.cd /DIR_OF_YCSB/

2.configure *settings.dat*:set recordcount the same as the recordcount in *settings_load.dat* And of course given the right server hosts

3.sudo ./run.command

#How TO USE
##Run the server:
1.cd /DIR_OF_CASSANDRA/ *it's in the /home/arc/setup/apache-cassandra-0.7.3*

2.sudo bin/cassandra -f

##Run the client:
**Load data:**

1.cd /DIR_OF_YCSB/ *it's in the /home/arc/setup/YCSB*

2.configure *settings_load.dat*
hosts: specifies the IP address of the machine running Cassandra.
recordcount: specifies the number of records to be loaded in the data store.

3.sudo ./run_load.command

**Run benchmark**

1.cd /DIR_OF_YCSB/ *it's in the /home/arc/setup/YCSB*

2.configure *settings.dat*:set recordcount the same as the recordcount in *settings_load.dat* And of course given the right server hosts

3.sudo ./run.command

#Run the server and client in two different machine:
If you want to run the server and client in different machine, just run the server in the server machine and configure the hosts in *settings_load.dat* and *settings.dat* in the client machine. Then load data and run benchmark in client machine.

# Reference

* [Official Installation Guideline](http://parsa.epfl.ch/cloudsuite/docs/data-serving.pdf)
