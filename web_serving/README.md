* Author: [Xian Chen](http://doctorsnail.net/)

#Introduction

Web serving is a fundamental application in any Internet-based service. In web serving benchmark CloudStone is used to benchmark Web2.0 applications. CloudStone includes a Web2.0 social-events application(Olio) and a client implemented using the Faban workload generator. CloudStone can be used to benchmark various web technologies and web server software stacks.

Because the source code of Faban provided by the official benchmark package has some compatibility issues on Ubuntu 12.04, and the official installation manual does not provide enough details, so I write this blog to record the procedure of deploying CloudStone on 64-bit Ubuntu12.04. This blog is base on the [official installation manual](http://parsa.epfl.ch/cloudsuite/downloads.html) and [Nikolay’s blog](https://github.com/nikolayg/CloudStoneSetupOnUbuntu). If you want to deploy CloudStone without typing so many commands, [Nikolay’s installation script](https://github.com/nikolayg/CloudStoneSetupOnUbuntu/blob/master/as-setup.sh) is the best choice. The following steps are a little time consuming, please be patient :).

#Prepare the disk imgs and setup the environment

1. Create disk imgs (we named it webserving.img). For convenience, I set the img size 50GB. When the deployment is complete successfully I find the client VM occupies around 5GB, server VM occupies about 32GB and the backend VM occupies 5.2GB.
```
qemu-img create –f qcow2 webserving.img 50G
```

2. Install OS for webserver.img, here we use 64-bit ubuntu12.04
```
sudo qemu-system-x86_64 –had webserving-server.img –cdrom ubuntu12.04.iso -boot d -m 1024 -no-acpi -vnc :1
```

3. If the OS is ready, then we have to configure IP address and update the software source, also some softwares need to be installed.
```
sudo apt-get update
sudo apt-get install openssh-server
sudo apt-get install lrzsz
sudo apt-get install vim
sudo apt-get install ant
sudo apt-get install patch
sudo apt-get install git
```

4. Install openjdk6 and setup the environment
```
sudo apt-get install openjdk-6-jdk
```
Edit /etc/profile, add the following line to the end of the file
```
export JAVA_HOME="/usr/lib/jvm/java-1.6.0-openjdk-amd64"
```
Then execute the following command to put put the new configuration into effect
```
source /etc/profile
```

5. duplicate two VMs based on this img
```
cp webserving.img webserving-client.img
cp webserving.img webserving-backend.img
mv webserving.img webserving-server.img
```

6. Reconfigure the IP and MAC addresses of the three VMs. In my environment, client VM uses: 10.214.50.164, server VM uses 10.214.50.163 and the backend VM uses 10.214.50.165. Then use ssh-keygen and ssh-copy-id commands to make sure each VM can access the other two VMs without account and authentication requirements.

7. Edit /etc/hosts in client VM, change 127.0.1.1 to the IP of client VM.

#Setting up the client machine

1. Untar web benchmark package and setup the environment

```
tar xzvf web.tar.gz
git clone https://github.com/nikolayg/faban.git
cp -ar faban/stage  web-release/
mv web-release/stage  web-release/faban
```
edit /etc/profile, add the following command to the end: 
```
export FABAN_HOME="/home/arc/workspace/web-release/faban"
```
```
source /etc/profile
cd web-release
```
2.setup Olio

```
tar xzvf apache-olio-php-src-0.2.tar.gz
```
edit /etc/profile, add the following command to the end:  
```
export OLIO_HOME="/home/arc/workspace/web-release/apache-olio-php-src-0.2"
```
```
source /etc/profile
tar xzvf mysql-connector-java-5.0.8.tar.gz
cp mysql-connector-java-5.0.8/mysql-connector-java-5.0.8-bin.jar $OLIO_HOME/workload/php/trunk/lib
cd $FABAN_HOME
cp samples/services/ApacheHttpdService/build/ApacheHttpdService.jar services/
cp samples/services/MysqlService/build/MySQLService.jar services/
cp samples/services/MemcachedService/build/MemcachedService.jar services/
cd $OLIO_HOME/workload/php/trunk
cp build.properties.template build.properties
```
edit build.properties as following
```
faban.home= /home/arc/workspace/web-release/faban
faban.url=http://10.214.50.164:9980
ant deploy.jar
cp $OLIO_HOME/workload/php/trunk/build/OlioDriver.jar $FABAN_HOME/benchmarks
```
3. run faban master

```
$FABAN_HOME/master/bin/startup.sh
```
Now you can point your browser to :http://10.214.50.164:9980. You should see the OlioWorkload welcome note. Then copy the faban directory ($FABAN_HOME) to the server and backend machines. Faban directories must be in the same path on every machine.

#Setting up the backend machine

1. sudo apt-get install libaio1
2. groupadd mysql
3. sudo useradd -r -g mysql mysql
4. copy the benchmark file (web.tar.gz) to the backend machine, decompress it
    * tar xzvf web.tar.gz
    * cd web-release
    * tar xzvf mysql-5.5.20-linux2.6-x86_64.tar.gz
5. sudo chown -R mysql mysql-5.5.20-linux2.6-x86_64
6. sudo chgrp -R mysql mysql-5.5.20-linux2.6-x86_64
7. cd mysql-5.5.20-linux2.6-x86_64/
8. sudo cp support-files/my-medium.cnf /etc/my.cnf
9. sudo ./scripts/mysql_install_db –user=mysql
10. sudo bin/mysqld_safe –defaults-file=/etc/my.cnf –user=mysql &
11. install mysql
    * sudo bin/mysql -uroot
    * create user 'olio'@'%' identified by 'olio';
    * grant all privileges on *.* to 'olio'@'localhost' identified by 'olio' with grant option;
    * grant all privileges on *.* to 'olio'@'10.214.50.163' identified by 'olio' with grant option;
12. create database and table
    * create database olio;
    * use olio;
    * \. $FABAN_HOME/benchmarks/OlioDriver/bin/schema.sql
    * exit
13. cd $FABAN_HOME/benchmarks/OlioDriver/bin
14. sudo chmod +x dbloader.sh
15. fill the database. ./dbloader.sh <load_scale> ;note that we use the local dbserver,so we use localhost, and we select 25 as the load scale, then the loader will load 100*25 users. This operation may take more time, then take a coffe to wait patiently.
16. set up the tomcat
    * tar xzvf apache-tomcat-6.0.35.tar.gz
    * append the following line to /etc/profile : export CATALINA_HOME="/home/arc/workspace/web-release/apache-tomcat-6.0.35" . then execute the command: source /etc/profile
    * cd $CATALINA_HOME/bin
    * tar zxvf commons-daemon-native.tar.gz
    * cd commons-daemon-1.0.7-native-src/unix
    * ./configure
    * make
    * cp jsvc ../..
17. set upgeocoder emulator
    * copy $OLIO_HOME/geocoder directory from the client machine to the backend machine
    * append /etc/profile with export GEOCODER_HOME="/home/arc/workspace/web-release/apache-olio-php-src-0.2/geocoder"
    * source /etc/profile
    * cd $GEOCODER_HOME/
18. cp build.properties.template build.properties
19. change the servlet.lib.path in build.properties to $CATALINA_HOME/lib
20. ant all
21. cp dist/geocoder.war $CATALINA_HOME/webapps
22. $CATALINA_HOME/bin/startup.sh

#Setting up the frontend machine

1. decompress apache-olio-php-src-0.2.tar.gz and set up the enviroment variables.
    * tar xzvf apache-olio-php-src-0.2.tar.gz
    * append /etc/profile with export OLIO_HOME="/home/arc/workspace/web-release/apache-olio-php-src-0.2"
    * source /etc/profile
2. create working directory
    * append command "export APP_DIR="/var/www"" to /etc/profile
    * source /etc/profile
    * sudo mkdir -p $APP_DIR
3. sudo cp -r $OLIO_HOME/webapp/php/trunk/* $APP_DIR
4. sudo cp cloudstone.patch $APP_DIR
5. cd $APP_DIR
6. sudo patch -p1 < cloudstone.patch
7. change $APP_DIR/etc/config.php
    * $olioconfig['dbTarget'] = 'mysql:host=10.214.50.165;dbname=olio';
    * comment out $olioconfig['cacheSystem'] = 'MemCached';
    * remove the comment $olioconfig['cacheSystem'] = 'NoCache';
    * $olioconfig['geocoderURL'] = 'http://10.214.50.165:8080/geocoder/geocode';

###install Nginx

1. install the dependent packages
    * sudo apt-get install libpcre3 libpcre3-dev libpcrecpp0 libssl-dev zlib1g-dev
    * tar zxvf nginx-1.0.11.tar.gz
    * cd nginx-1.0.11/
    * ./configure
    * make
    * sudo make install
2. check whether the installation is success
    * sudo /usr/local/nginx/sbin/nginx
    * point your borwser to http://10.214.50.163 and you will get a welcom page
    * sudo /usr/local/nginx/sbin/nginx -s stop
3. edit the configure file of nginx (/usr/local/ngnix/conf/nginx.conf)

    * location / {   
    root /var/www/public_html;   
    index index.html index.htm index.php;   
    * append the following text in server segment   
    location ~\.php${   
    root /var/www/public_html;   
    fastcgi_pass 127.0.0.1:9000;   
    fastcgi_index index.php;   
    fastcgi_param SCRIPT_FILENAME/var/www/public_html/$fastcgi_script_name;   
    include fastcgi_params;   
    }   
    * add access_log off in server segment   
4. sudo /usr/local/nginx/sbin/nginx

###install PHP

1. sudo apt-get install libxml2-dev curl libcurl3 libcurl3-dev libjpeg-dev libpng-dev
2. tar zxvf php-5.3.9.tar.gz
3. cd php-5.3.9/
4. /configure –enable-fpm –with-curl –with-pdo-mysql=/home/arc/workspace/web-release/mysql-5.5.20-linux2.6-x86_64 –with-gd –with-jpeg-dir –with-png-dir –with-config-file-path=/var/www/etc/
5. make
6. sudo make install
7. append export PHPRC="/var/www/etc/" to /etc/profile and execute the command source /etc/profile
8. mkdir -p /tmp/http_sessions
9. edit /var/www/etc/php.ini
    * append the following text  
    extension_dir=/usr/local/lib/php/extensions/no-debug-non-zts-20090626/  
    date.timezone="Europe/Zurich"  
    * error_reporting = E_NONE  
10. run nginx with support of php
    * sudo cp /usr/local/etc/php-fpm.conf.default /usr/local/etc/php-fpm.conf
    * sudo addgroup nobody
    * sudo /usr/local/sbin/php-fpm

###install APC

1. sudo apt-get install autoconf
2. tar xzvf APC-3.1.9.tgz
3. cd APC-3.1.9/
4. phpize
5. /configure –enable-apc –enable-apc-mmap –with-php-config=/usr/local/bin/php-config
6. make
7. sudo make install
8. check the usability of APC. Execute the command php-fpm -m and the result should contain the apc module
9. restart php-fpm. sudo /usr/local/sbin/php-fpm

###configure filestore

1. copy cloudsuite.patch to /var/www
2. cd /var/www
3. sudo patch -p1< cloudsuite.patch
4. set up the environment and change the permission
    * append export FILESTORE="/var/filestore" to /etc/profile
    * source /etc/.profile
    * sudo mkdir -p $FILESTORE
    * sudo chmod a+rwx $FILESTORE
5. copy the faban directory from client machine to frontend machine
6. set $FABAN_HOME to “/home/arc/workspace/web-release/faban”
7. sudo chmod +x $FABAN_HOME/benchmarks/OlioDriver/bin/fileloader.sh
8. $FABAN_HOME/benchmarks/OlioDriver/bin/fileloader.sh 102 /var/filestore.
9. edit $APP_DIR/etc/config.php.
    $olioconfig['localfsRoot'] = '/var/filestore';
10. restart php-fpm.
    sudo /usr/local/sbin/php-fpm

#Start to conduct experiments

1. make sure faban master, mysql server, nginx, php-fpm are all running

2. point your browser to the client machine, here I use http://10.214.50.164:9980

3. configure the experiment environmnt, and point the Ok menue
