* Author: Xinjie YU ([chetui](http://blog.chetui.org))
* Email: yuxinjiect@gmail.com

## Packages

[libevent-2.0.21-stable.tar.gz](https://github.com/downloads/libevent/libevent/libevent-2.0.21-stable.tar.gz)  
[memcached-1.4.15.tar.gz](http://memcached.googlecode.com/files/memcached-1.4.15.tar.gz)  
[memcached.tar.gz](http://parsa.epfl.ch/cloudsuite/software/memcached.tar.gz)  

## Building on Server

Build and install libevent:

```
$ tar zxvf libevent-2.0.21-stable.tar.gz 
$ cd libevent-2.0.21-stable 
$ ./configure 
$ make
$ sudo make install
```
Build and install Memcached:

```
$ tar zxvf memcached-1.4.15.tar.gz 
$ cd memcached-1.4.15 
$ ./configure
$ make
$ sudo make install
```
## Building on Client

```
$ tar zxvf memcached.tar.gz
$ cd memcached/memcached_client
```
Fix the Makefile Bugs:   
Replace following rule

```
	gcc -O3  -Wall -levent  -pthread -lm -D_GNU_SOURCE  *.c -o loader
```
by following fixed rule

```
	gcc -O3  -Wall -pthread -D_GNU_SOURCE  *.c -o loader -lm -levent
```
Then build it:

```
$ make
```

## Runing Server

```
$ memcached -t 4 -m 4096 -n 550
```
-t 4: four threads  
-m 4096: 4096MB memory  
-n 550: minimal object size of 550 bytes  
-p 11211: the default port 11211 can be changed by this option  

## Runing Client

#### Configuring Target Servers

Modify the memcached/memcached_client/servers.txt according to your servers.  
e.g. If you run the client on the same machine with the only server, then you can configure your servers.txt like this:

```
127.0.0.1 11211
```

#### Scaling the Dataset & Warming up Target Servers

If it is your have not scaled the dataset to 30 times, then you need to create it according to following commands. And the server would be warmed up at the same time (the original dataset requires 300MB Memcached server memory, while the dataset scaling to 30 times requres 10GB):

```
$ cd memcached/memcached_client
$ ./loader -a ../twitter_dataset/twitter_dataset_unscaled -o ../twitter_dataset/twitter_dataset_30x -s servers.txt -w 1 -S 30 -D 4096 -j -T 1 
```

Or if the scaled file is already created, you can warming up server directly:

```
$ cd memcached/memcached_client
$ ./loader -a ../twitter_dataset/twitter_dataset_30x -s servers.txt -w 1 -S 1 -D 4096 -j -T 1
```

-w 1: number of client threads  
-S 30: scaling factor  
-D 4096: target server memory, MB  
-T 1: statistics interval  
-s servers.txt: server configuration file  
-j : an indicator that the server should be warmed up  

**The legend for the output**:

```
timediff - the measurement period T (1s in your case)
rps - requests per second during the last T
requests - total number of requests completed within last the last T  (if T=1s, equals to rps)
gets - number of completed get requests during the last T
sets - number of completed set requests during the last T
hits - number of hits in memcached during the last T
misses - number of misses in memcached during the last T
avg_lat - average latency in milliseconds during the last T
90th - 90-percentile latency in milliseconds during the last T
95th - 95-percentile latency in milliseconds during the last T
99th - 99-percentile latency in milliseconds during the last T
min - minimum latency achieved for some package during the last T
max - maximum latency achieved for some package during the last T
std - standard deviation
avg_size - average data size for get requests during the last T
```
Please wait until ./loader program finished.

#### Determining the Maximum Throughput

```
$ cd memcached/memcached_client
$ ./loader -a ../twitter_dataset/twitter_dataset_30x -s servers.txt -g 0.8 -T 1 -c 200 -w 8 
```
-g 0.8: get/set ration of 0.8  
-T 1: statistics interval  
-c 200: 200 TCP/IP connections  
-w 8: 8 client threads  

This program would not stop itself. You have to estimate the maximum rps in the output (set it as max_rps), then top the program yourself. 

#### Running Client Benchmark  

The command above will run the benchmark with the maximum throughput, however, the QoS requirements will highly likely be violated.  
The target QoS requires that 95% of the requests are executed within 10ms  
For not to violate the target QoS requirements, replace the following ***your_rps*** by your adjusted value. e.g. Let ***your_rps = 0.9 * max_rps***  
You may need to adjust ***your_rps*** for many times, until it achieve the maximum throughput without violating the target QoS requirements.

```
$ cd memcached/memcached_client
$ ./loader -a ../twitter_dataset/twitter_dataset_30x -s servers.txt -g 0.8 -T 1 -c 200 -w 8 -e -r your_rps
```

## References

* [Official Installation Guideline](http://parsa.epfl.ch/cloudsuite/docs/data-caching.pdf)
* <http://suokun.blogspot.com/2013/12/virtualization-7cloudsuitememcached.html>
* <https://www.mail-archive.com/cloudsuite@listes.epfl.ch/msg00364.html>
