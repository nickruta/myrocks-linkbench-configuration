Detailed installation instructions for setting up MyRocks to use Linkbench for benchmarking.

### I used a fresh installation of FEDORA Workstation 25 64-bit

### install required libraries for linux
sudo yum install cmake gcc-c++ bzip2-devel libaio-devel bison zlib-devel snappy-devel

sudo yum install gflags-devel readline-devel ncurses-devel openssl-devel lz4-devel gdb git



# MyROCKS SETUP - 

cd ~
mkdir git
cd git
git clone https://github.com/facebook/mysql-5.6.git

cd mysql-5.6/

git submodule init

git submodule update

cmake . -DCMAKE_BUILD_TYPE=RelWithDebInfo -DWITH_SSL=system -DWITH_ZLIB=bundled -DMYSQL_MAINTAINER_MODE=0 -DENABLED_LOCAL_INFILE=1 -DCMAKE_INSTALL_PREFIX=/home/openxs/dbs/fb56

time make

sudo make install && make clean 

cd /home/openxs

### create mysql user and set password
useradd -M mysql
sudo passwd mysql
EDIT etc/sudoers file to include mysql same as root privileges

su mysql

### create fb56.cnf file and include the minimum requirements for rocksdb
sudo vi fb56.cnf

### paste this in fb56.cnf
[mysqld]
rocksdb
default-storage-engine=rocksdb
skip-innodb
default-tmp-storage-engine=MyISAM

log-bin
binlog-format=ROW


### install MySQL
cd dbs/fb56/

sudo scripts/mysql_install_db --defaults-file=/home/openxs/fb56.cnf

### give ownership of the data folder to user mysql
chown mysql /home/openxs/dbs/fb56/data/

### start MySQL server
bin/mysqld_safe --defaults-file=/home/openxs/fb56.cnf &


#MySQL SETUP - 

### open a new terminal window
cd /home/openxs/dbs/fb56
bin/mysql -uroot test

### MAKE ROOT USER HAVE PASSWORD ROOT
SET PASSWORD FOR 'root'@'localhost' = PASSWORD(‘root’);

### LINKBENCH NEEDS MORE MAX CONNECTIONS
SET GLOBAL max_connections = 1024;


### create the three linkbench tables needed

create database linkdb;

use linkdb;


  CREATE TABLE `linktable` (
    `id1` bigint(20) unsigned NOT NULL DEFAULT '0',
    `id1_type` int(10) unsigned NOT NULL DEFAULT '0',
    `id2` bigint(20) unsigned NOT NULL DEFAULT '0',
    `id2_type` int(10) unsigned NOT NULL DEFAULT '0',
    `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
    `visibility` tinyint(3) NOT NULL DEFAULT '0',
    `data` varchar(255) NOT NULL DEFAULT '',
    `time` bigint(20) unsigned NOT NULL DEFAULT '0',
    `version` int(11) unsigned NOT NULL DEFAULT '0',
    PRIMARY KEY (link_type, `id1`,`id2`) COMMENT 'cf_link_pk',
    KEY `id1_type` (`id1`,`link_type`,`visibility`,`time`,`version`,`data`) COMMENT 'rev:cf_link_id1_type'
  ) ENGINE=RocksDB DEFAULT COLLATE=latin1_bin; 

CREATE TABLE `counttable` (
  `id` bigint(20) unsigned NOT NULL DEFAULT '0',
  `link_type` bigint(20) unsigned NOT NULL DEFAULT '0',
  `count` int(10) unsigned NOT NULL DEFAULT '0',
  `time` bigint(20) unsigned NOT NULL DEFAULT '0',
  `version` bigint(20) unsigned NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`,`link_type`)
) ENGINE=RocksDB DEFAULT CHARSET=latin1;

CREATE TABLE `nodetable` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  `type` int(10) unsigned NOT NULL,
  `version` bigint(20) unsigned NOT NULL,
  `time` int(10) unsigned NOT NULL,
  `data` mediumtext NOT NULL,
  PRIMARY KEY(`id`)
) ENGINE=RocksDB DEFAULT CHARSET=latin1;


#LINKBENCH SETUP - 

### open a new terminal window

cd~/git

### get linkbench
git clone https://github.com/facebookarchive/linkbench.git

### build linkbench
mvn clean package -P fast-test

### set up config files for linkbench
### here are some settings to make a faster first run to verify that it is all set up correctly 
###inside the FBWorkload.properties file, make the maxid1 smaller
vi config/FBWorkload.properties

	maxid1 = 500000

### in the MyConfig.properties
vi config/

	# MySQL connection information
	host = localhost 
	user = root
	password = root

	# under Load Phase Configuration 
	loaders = 10

	# under Request Phase Configuration 
	requesters = 20
	requests = 10000


### run linkbench against my rocks!

### load phase
./bin/linkbench -c config/LinkConfigMysql.properties -l

### request phase ( I set the warmup time to one second but default was 600 )
./bin/linkbench -c config/LinkConfigMysql.properties -D warmup_time=1 -r

### verify in MySQL terminal that RocksDB was the ENGINE used - 
mysql> show engine rocksdb status\G
