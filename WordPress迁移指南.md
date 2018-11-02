

https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html

For large-scale backup and restore, a physical backup is more appropriate, to copy the data files in their original format that can be restored quickly: 


physical backup

    A backup that copies the actual data files. For example, the mysqlbackup command of the MySQL Enterprise Backup product produces a physical backup, because its output contains data files that can be used directly by the mysqld server, resulting in a faster restore operation. Contrast with logical backup.

    See Also backup, logical backup, MySQL Enterprise Backup, restore.

logical backup

    A backup that reproduces table structure and data, without copying the actual data files. For example, the mysqldump command produces a logical backup, because its output contains statements such as CREATE TABLE and INSERT that can re-create the data. Contrast with physical backup. A logical backup offers flexibility (for example, you could edit table definitions or insert statements before restoring), but can take substantially longer to restore than a physical backup.

    See Also backup, mysqldump, physical backup, restore.

incremental backup

    A type of hot backup, performed by the MySQL Enterprise Backup product, that only saves data changed since some point in time. Having a full backup and a succession of incremental backups lets you reconstruct backup data over a long period, without the storage overhead of keeping several full backups on hand. You can restore the full backup and then apply each of the incremental backups in succession, or you can keep the full backup up-to-date by applying each incremental backup to it, then perform a single restore operation.

    The granularity of changed data is at the page level. A page might actually cover more than one row. Each changed page is included in the backup.

    See Also hot backup, MySQL Enterprise Backup, page.

hot backup

    A backup taken while the database is running and applications are reading and writing to it. The backup involves more than simply copying data files: it must include any data that was inserted or updated while the backup was in process; it must exclude any data that was deleted while the backup was in process; and it must ignore any changes that were not committed.

    The Oracle product that performs hot backups, of InnoDB tables especially but also tables from MyISAM and other storage engines, is known as MySQL Enterprise Backup.

    The hot backup process consists of two stages. The initial copying of the data files produces a raw backup. The apply step incorporates any changes to the database that happened while the backup was running. Applying the changes produces a prepared backup; these files are ready to be restored whenever necessary.

    See Also apply, MySQL Enterprise Backup, prepared backup, raw backup.


    mysqldump

    binglog

使用 mysqldump 迁移 MySQL 数据

https://help.aliyun.com/document_detail/26133.html?spm=a2c4g.11174283.4.9.40794c227am003
使用 mysqldump 工具的优点是简单易用、容易上手，缺点是停机时间较长，因此它适用于数据量不大，或者允许停机的时间较长的情况。



迁移任务一直卡在增量数据迁移阶段，什么时候结束？

更新时间：2017-06-07 13:26:11
★ 我的收藏

    新手学堂
    学习路径

 

增量数据迁移阶段，会进行源实例跟目标实例增量数据实时同步，不会自动结束。建议增量数据迁移无延迟时，业务在目标实例验证通过后，将业务切换到目标实例，并手动结束迁移任务。



如果先做一个全量迁移任务，然后再做增量数据迁移任务，迁移数据是否会出现不一致

更新时间：2017-06-07 13:26:11
★ 我的收藏

    新手学堂
    学习路径

 

数据会不一致，迁移任务单独做增量数据迁移时，增量迁移开始同步的增量数据为启动任务的时间。所以在增量迁移任务启动之前，源数据库产生的增量数据都不会被同步到目标实例。
如果需要进行不停机迁移，建议配置任务时，迁移类型选择 结构迁移、全量数据迁移及增量数据迁移


https://www.aliyun.com/service/datamigration?spm=5176.2020520165.121.d427.3bf77029XgFSkE
网站上云数据迁移服务
阿里云认证区域服务提供商，将通过远程方式协助客户将网站数据、应用程序迁移至阿里云服务器，并进行网站基础环境配置，让客户网站的整体上云过程更加简单。


网站上云数据迁移服务



DTS 数据增量迁移的基本原理是什么？是否能够保证两边数据库实时的？使用增量迁移时，是否会锁表？

更新时间：2018-01-30 10:12:37
★ 我的收藏

    新手学堂
    学习路径

 

DTS 的增量迁移是实时获取在迁移过程中，源数据库产生的增量数据，然后在全量迁移完成后，开始同步到目标 RDS 实例中。
当增量迁移第一次追平源库的写入时，增量迁移的状态为无延迟，此后增量迁移会一直同步源数据库的业务写入。
DTS 在进行全量数据迁移和增量数据迁移的过程中，均不会对源端数据库进行锁表，因此在全量数据迁移和增量数据迁移的过程中，迁移源端的数据表均可以正常读写访问。



迁移任务一直卡在增量数据迁移阶段，什么时候结束？

 

增量数据迁移阶段，会进行源实例跟目标实例增量数据实时同步，不会自动结束。建议增量数据迁移无延迟时，业务在目标实例验证通过后，将业务切换到目标实例，并手动结束迁移任务。


MySQL的binlog日志

```bash

MichaeldeMacBook-Pro-2:~ myang$ mysqldump -u user001 -p -h mysql-cn-north-1-076a9fd5a44a4eff.public.jcloud.com --databases wordpress -lF
Enter password: 
-- MySQL dump 10.13  Distrib 5.7.23, for macos10.13 (x86_64)
--
-- Host: mysql-cn-north-1-076a9fd5a44a4eff.public.jcloud.com    Database: wordpress
-- ------------------------------------------------------
-- Server version	5.7.21-log

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events. 
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;

--
-- GTID state at the beginning of the backup 
--

SET @@GLOBAL.GTID_PURGED='84fc2b48-c3cf-11e8-b5f4-fa163e6f191b:1-948';

--
-- Current Database: `wordpress`
--

CREATE DATABASE /*!32312 IF NOT EXISTS*/ `wordpress` /*!40100 DEFAULT CHARACTER SET utf8 */;

USE `wordpress`;
mysqldump: Got error: 1227: Access denied; you need (at least one of) the RELOAD privilege(s) for this operation when doing refresh
```


http://www.mamicode.com/info-detail-2092497.html
数据库的备份与恢复 mysqldump+binlog方式


 MySQL三种备份

一）备份分类

https://www.cnblogs.com/frankielf0921/p/5924365.html

	
冷备：cold backup数据必须下线后备份
温备：warm backup全局施加共享锁，只能读，不能写
热备：hot backup数据不离线，读写都能正常进行
备份的数据集
完全备份：full backup
部分备份：partial backup
备份时的接口（是直接备份数据文件还是通过mysql服务器导出数据）
物理备份：直接复制（归档）数据文件的备份方式：physical backup
逻辑备份：把数据从库中提出来保存为文本文件：logical backup
完全备份：full backup
增量备份：incrementl backup
差异备份：fidderential backup


MichaeldeMacBook-Pro-2:~ myang$ mysqldump -u user001 -p -h mysql-cn-north-1-076a9fd5a44a4eff.public.jcloud.com --databases wordpress --single-transaction --flush-logs 
Enter password: 
-- MySQL dump 10.13  Distrib 5.7.23, for macos10.13 (x86_64)
--
-- Host: mysql-cn-north-1-076a9fd5a44a4eff.public.jcloud.com    Database: wordpress
-- ------------------------------------------------------
-- Server version	5.7.21-log

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
mysqldump: Couldn't execute 'FLUSH TABLES': Access denied; you need (at least one of) the RELOAD privilege(s) for this operation (1227)


CentOs安装Mysql和配置初始密码
https://www.cnblogs.com/FlyingPuPu/p/7783735.html


初始化数据。20张表，每张表200万数据。
```
# sysbench /usr/share/sysbench/oltp_read_write.lua prepare --db-driver=mysql --mysql-password=Passw0rd --tables=20 --table-size=2000000 --threads=32 
```
查看生成的数据大小以及具体的内容。
```
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from information_schema.TABLES where table_schema='sbtest';
+-----------+
| data      |
+-----------+
| 8384.63MB |
+-----------+
1 row in set (0.07 sec)

mysql> use sbtest;
Database changed
mysql> desc sbtest1;
+-------+-----------+------+-----+---------+----------------+
| Field | Type      | Null | Key | Default | Extra          |
+-------+-----------+------+-----+---------+----------------+
| id    | int(11)   | NO   | PRI | NULL    | auto_increment |
| k     | int(11)   | NO   | MUL | 0       |                |
| c     | char(120) | NO   |     |         |                |
| pad   | char(60)  | NO   |     |         |                |
+-------+-----------+------+-----+---------+----------------+
4 rows in set (0.18 sec)

mysql> select count(*) from sbtest1;
+----------+
| count(*) |
+----------+
|  2000000 |
+----------+
1 row in set (0.28 sec)

mysql> select pad from sbtest1 where id = 1;
+-------------------------------------------------------------+
| pad                                                         |
+-------------------------------------------------------------+
| 98996621624-36689827414-04092488557-09587706818-65008859162 |
+-------------------------------------------------------------+
1 row in set (0.01 sec)
```

# 热备：hot backup数据不离线，读写都能正常进行

##开启binlog日志，在/etc/my.cnf中增加如下内容：
server-id=1
log-bin=mysql-bin


```ini
[root@ymq-srv011 etc]# cat my.cnf
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

server-id=1
log-bin=mysql-bin

查看MYSQL状态，表示已经启用bin-log

```sql
mysql> show variables like 'log_%';
+----------------------------------------+--------------------------------+
| Variable_name                          | Value                          |
+----------------------------------------+--------------------------------+
| log_bin                                | ON                             |
| log_bin_basename                       | /var/lib/mysql/mysql-bin       |
| log_bin_index                          | /var/lib/mysql/mysql-bin.index |
| log_bin_trust_function_creators        | OFF                            |
| log_bin_use_v1_row_events              | OFF                            |
| log_builtin_as_identified_by_password  | OFF                            |
| log_error                              | /var/log/mysqld.log            |
| log_error_verbosity                    | 3                              |
| log_output                             | FILE                           |
| log_queries_not_using_indexes          | OFF                            |
| log_slave_updates                      | OFF                            |
| log_slow_admin_statements              | OFF                            |
| log_slow_slave_statements              | OFF                            |
| log_statements_unsafe_for_binlog       | ON                             |
| log_syslog                             | OFF                            |
| log_syslog_facility                    | daemon                         |
| log_syslog_include_pid                 | ON                             |
| log_syslog_tag                         |                                |
| log_throttle_queries_not_using_indexes | 0                              |
| log_timestamps                         | UTC                            |
| log_warnings                           | 2                              |
+----------------------------------------+--------------------------------+
21 rows in set (0.00 sec)

mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       154 |
+------------------+-----------+
1 row in set (0.00 sec)

[root@ymq-srv011 ~]# time mysqldump -u sbtest -pPassw0rd --databases sbtest --single-transaction --flush-logs>sbtest.sql
mysqldump: [Warning] Using a password on the command line interface can be insecure.

real	2m3.736s
user	1m6.830s
sys	0m7.362s
[root@ymq-srv011 ~]# ls -al *.sql
-rw-r--r-- 1 root root 8043029640 Nov  1 13:04 sbtest.sql
```



```
[root@ymq-srv011 mysql]# ls -al mysql-bin.*
-rw-r----- 1 mysql mysql  928 Nov  1 12:59 mysql-bin.000001
-rw-r----- 1 mysql mysql 1406 Nov  1 13:02 mysql-bin.000002
-rw-r----- 1 mysql mysql 1220 Nov  1 13:03 mysql-bin.000003
-rw-r----- 1 mysql mysql   57 Nov  1 13:02 mysql-bin.index
[root@ymq-srv011 mysql]# pwd
/var/lib/mysql
```
数据不离线

```
mysqlbinlog --start-position=123 --stop-position=1189  /var/lib/mysql/mysql-bin.000003 > `date +%F-%T`-add.sql


http://www.mamicode.com/info-detail-2092497.html
```


# 把备份文件上传到对象存储



导入圈梁数据
mysql –uuser001 -pPassword -Dsbtest<sbtest.sql -h jddb-cn-north-1-fd9a1be6a9e34f0c.jcloud.com





On macOS, install via Homebrew:

$ brew cask install osxfuse
$ brew install s3fs

```执行数据库全量导入

time  mysql -uuser001 -pPassw0rd  -h jddb-cn-north-1-e7501da9ca874bd0.jcloud.com -Dsbtest<sbtest.sql



导入全量数据，话费14分钟
[root@ymq-srv011 ~]# time  mysql -uuser001 -pPassw0rd  -h jddb-cn-north-1-e7501da9ca874bd0.jcloud.com -Dsbtest<sbtest.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
real	14m43.311s
user	0m54.393s
sys	0m20.569s