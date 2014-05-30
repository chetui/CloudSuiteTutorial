* Author: Xinjie YU ([chetui](http://blog.chetui.org))
* Email: yuxinjiect@gmail.com

If you want to install all these into a single node, make sure your node have enough disk capacity.   
Since the downloaded testing datasetat will increase over time, On 2014.05.27 my node needs 120GB disk capacity totally.  
    
If you have an deployed image, you can jump to [Runing Benchmark Section](https://github.com/chetui/CloudSuiteTutorial/tree/master/data_analytics#running-benchmark) directly. 

## Packages

[Analytics.tar.gz](http://parsa.epfl.ch/cloudsuite/software/analytics.tar.gz) (including Hadoop0.20.2, Mahout and Apache Maven.)  
[Wikipedia Training Dataset](http://parsa.epfl.ch/cloudsuite/software/enwiki-20100904-pages-articles1.xml.bz2) (Around 5.4GB after decompression)  
[Wikipedia Test Dataset](http://download.wikimedia.org/enwiki/latest/enwiki-latest-pages-articles.xml.bz2) (The size will increase over time. On 2014.05.27 its size is around 45GB after decompression)

## Installing Hadoop

#### Account Setting

It is suggested to create a isolated account to run this benchmark.  
Hence, we are going to create a account and a group all named ***hadoop***.

```
$ sudo groupadd hadoop
$ sudo useradd -md /home/hadoop -g hadoop hadoop
$ sudo passwd hadoop
```

**[Option]** You can change the default shell from /bin/sh to /bin/bash.

```
$ su - hadoop
$ chsh -s /bin/bash
$ exit
```

Set up the public key of SSH. 

```
$ su - hadoop
$ ssh-keygen -t rsa -P ""   #Please type [Enter] when it ask you to input something.
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ ssh localhost        #verify whether you set up public key successfully. You may need to type [yes] if it is your first time to ssh localhost.
$ exit
$ exit
```

Now you have created a hadoop account.   
**Notice that all the following steps in this tutorial should be executed under this hadoop account.**  

#### Uncompressing Packages

```
$ cp analytics.tar.gz /home/hadoop
$ cd /home/hadoop
$ tar jxvf analytics.tar.gz
$ mv analytics-release/* .
$ tar jxvf hadoop.tar.gz
$ tar jxvf mahout.tar.gz
$ tar jxvf apache-maven-3.0.3-bin.tar.gz
$ sudo chown -R hadoop:hadoop ./hadoop-0.20.2/
$ sudo chown -R hadoop:hadoop ./mahout-distribution-0.6/
$ sudo chown -R hadoop:hadoop ./apache-maven-3.0.3/
$ sudo chown hadoop:hadoop ./categories.txt
```

#### Installing Java JDk

Hadoop needs Java JDK 1.6 or later.  

Make sure your machine can access to the software source.  
Different Linux Distribution may be various. I use Ubuntu 12.04.

```
$ sudo apt-get install openjdk-6-jdk
```

Add following two lines into ***~/.bashrc***:

```
export HADOOP_HOME=/home/hadoop/hadoop-0.20.2
export JAVA_HOME=/usr/lib/jvm/java-6-openjdk-amd64/
```

Modify the ***line 10 in /home/hadoop/hadoop-0.20.2/conf/hadoop-env.sh***:

```
export JAVA_HOME=/usr/lib/jvm/java-6-openjdk-amd64/
```

#### Setting up Hadoop Environment

Make directories needed for Hadoop.

```
$ mkdir /home/hadoop/hadoop-0.20.2/fs
$ mkdir /home/hadoop/hadoop-0.20.2/namespace
$ mkdir /home/hadoop/hadoop-0.20.2/tmp
```

Configure path of directories.  
Modify following part in ***./hadoop-0.20.2/conf/hdfs-site.xml***:

```
  <property>
  <name>hadoop.tmp.dir</name>
  <value>/home/hadoop/hadoop-0.20.2/tmp</value>
  </property>
  <property>
  <name>dfs.name.dir</name>
  <value>/home/hadoop/hadoop-0.20.2/namespace</value>
  </property>
  <property>
  <name>dfs.data.dir</name>
  <value>/home/hadoop/hadoop-0.20.2/fs</value>
  </property>
```

Format the HDFS filesystem:

```
$ $HADOOP_HOME/bin/hadoop namenode -format
```

Start Hadoop Service:

```
$ $HADOOP_HOME/bin/start-all.sh     
```

Refer [How to use Hadoop Section](https://github.com/chetui/CloudSuiteTutorial/tree/master/data_analytics#how-to-use-hadoop) to know more about Hadoop command.

## Installing Mahout

You need to connect to the Internet to do the following steps.

```
$ cd mahout-distribution-0.6
$ ../apache-maven-3.0.3/bin/mvn install -DskipTests
```

## Preparing & Training Dataset

#### Uncompressing Dataset

```
$ cd ~
$ bzip2 -d enwiki-20100904-pages-articles1.xml.bz2
$ bzip2 -d enwiki-latest-pages-articles.xml.bz2

$ mkdir ./mahout-distribution-0.6/examples/temp
$ mv enwiki-20100904-pages-articles1.xml ./mahout-distribution-0.6/examples/temp
$ mv enwiki-latest-pages-articles.xml ./mahout-distribution-0.6/examples/temp
```

enwiki-20100904-pages-articles1.xml is the training dataset.  
enwiki-latest-pages-articles.xml is the testing dataset.


#### Chunking Dataset

You need to chunk both datasets into 64MB pieces.  
The output data would be stored in HDFS.  

1) Chunk training dataset.

```
$ ./mahout-distribution-0.6/bin/mahout wikipediaXMLSplitter -d /home/hadoop/mahout-distribution-0.6/examples/temp/enwiki-20100904-pages-articles1.xml -o wikipedia-training/chunks -c 64
```
The output data would be stored in wikipedia-training of HDFS.  
You can check chunk-*.xml files by following command.

```
$ $HADOOP_HOME/bin/hadoop dfs -ls wikipedia-training/chunks
```

2) Chunk testing dataset.

```
$ ./mahout-distribution-0.6/bin/mahout wikipediaXMLSplitter -d /home/hadoop/mahout-distribution-0.6/examples/temp/enwiki-latest-pages-articles.xml -o wikipedia/chunks -c 64
```

The output data would be stored in wikipedia of HDFS.  
You can check chunk-*.xml files by following command.

```
$ $HADOOP_HOME/bin/hadoop dfs -ls wikipedia/chunks
```

#### Creating category-based splits

```
$ cp categories.txt ./mahout-distribution-0.6/examples/temp
```

Your following action on Hadoop may need at lease 12G memory under the default configuration.  
If you want to change the configuration of Hadoop, refer to [Parameter Tuning Section](https://github.com/chetui/CloudSuiteTutorial/tree/master/data_analytics#hadoop-parameter-tuning).  

1) On training dataset.

```
$ ./mahout-distribution-0.6/bin/mahout wikipediaDataSetCreator -i wikipedia-training/chunks -o traininginput -c /home/hadoop/mahout-distribution-0.6/examples/temp/categories.txt
```

Check file ***part-r-00000 in traininginput***.

```
$ $HADOOP_HOME/bin/hadoop dfs -ls traininginput
```

2) On testing dataset.

```
$ ./mahout-distribution-0.6/bin/mahout wikipediaDataSetCreator -i wikipedia/chunks -o wikipediainput -c /home/hadoop/mahout-distribution-0.6/examples/temp/categories.txt
```

Check file ***part-r-00000 in wikipediainput***.

```
$ $HADOOP_HOME/bin/hadoop dfs -ls wikipediainput
```

#### Training Model

```
./mahout-distribution-0.6/bin/mahout trainclassifier -i traininginput -o wikipediamodel -mf 4 -ms 4
```

Check directory ***wikipediamodel***.

```
$ $HADOOP_HOME/bin/hadoop dfs -ls
```

## Running Benchmark

Check directory ***wikipediainput-output***.

```
$ $HADOOP_HOME/bin/hadoop dfs -ls
```

If there exist a directory named ***wikipediainput-output***, it means the benchmark already be runned.  
If you want to run this benchmark again, you need to remove ***wikipediainput-output*** directory first.

```
$ $HADOOP_HOME/bin/hadoop dfs -rmr wikipediainput-output
```

Before running this benchmark, you may need to tune parameter of Hadoop according your hardware configuration. Please refer to [Parameter Tuning Section](https://github.com/chetui/CloudSuiteTutorial/tree/master/data_analytics#hadoop-parameter-tuning).

Now you can run this benchmark:

```
./mahout-distribution-0.6/bin/mahout testclassifier -m wikipediamodel -d wikipediainput --method mapreduce
```

You can refer help info of Mahout to explore different ways of running benchmark:

```
./mahout-distribution-0.6/bin/mahout testclassifier --help
```

## Hadoop Parameter Tuning

* Change the number of map or reduce tasks per job according to [HowManyMapsAndReduces](http://wiki.apache.org/hadoop/HowManyMapsAndReduces)

Modify following parameters in file ***$HADOOP_HOME/conf/mapred-site.xml***:

``` 
mapred.map.tasks            # specifies the number of map tasks per job
mapred.reduce.tasks         # the number of reduce tasks per job
```

* Change the heap size taken by a java process

Modify following parameter in file ***$HADOOP_HOME/conf/mapred-site.xml***:

```
mapred.child.java.opts 
```


## How to use Hadoop

* Start Hadoop Service

```
$ $HADOOP_HOME/bin/start-all.sh
```

* Stop Hadoop Service

```
$ $HADOOP_HOME/bin/stop-all.sh
```

* List File/Dir in HDFS

```
$ $HADOOP_HOME/bin/hadoop dfs -ls [your_path]
```

* Remove Dir in HDFS

```
$ $HADOOP_HOME/bin/hadoop dfs -rmr path_to_your_dir
```

* Remove File in HDFS

```
$ $HADOOP_HOME/bin/hadoop dfs -rm path_to_your_file
```


## Troubleshoot

If you encounter following error message:

```
Exception in thread "main" ... org.apache.hadoop.hdfs.server.namenode.SafeModeException: ... Name node is in safe mode.
```

**Solution**

When the HDFS is just startup, it will fall into safe mode to check metadata & data block.   
You just need to wait for a few minutes and retry your command.

## Reference

* [Official Installation Guideline](http://parsa.epfl.ch/cloudsuite/analytics.html)
