
从自建MySQL迁移到京东云RDS

# 1 概述
本文描述如何把数据从自建MySQL数据库迁移到京东云数据库RDS-MySQL(简称京东云RDS)。整个迁移将遵循如下原则：
* 避免迁移后数据不一致;
* 缩短因迁移到京东云所导致的数据库不可修改时间(小于10分钟);
* 降低迁移成本(包括带宽成本、存储成本、计算成本);

基于上述原则，本迁移采取的思路首先把数据库全量从源MySQL导出，然后导入京东云RDS，该过程时间可能较长，但源数据库依然处于完全可读写状态；在完成全量导入后，分析源数据新增的binlog文件，增量导入到京东云RDS。增量导入过程所消耗的时长取决于在启动全量导出时间点到启动增量导入时间点之间数据库所发生的变化量。

从自建MySQL迁移到京东云RDS的整个过程会使用京东如下云服务：
* 云数据库RDS-MySQL：提供MySQL数据库PaaS服务，自带跨可用区高可用、自动备份和性能监控功能；
* 云主机：作为MySQL客户端，获得全量和增量数据文件，完成MySQL数据的导入；
* 对象存储：提供数据备份文件存储功能。自建MySQL的数据备份文件可通过Internet上传到对象存储，然后通过京东云内部的高速网络下载到云主机。

整个迁移过程包含如下阶段：
* 准备迁移环境
* 迁移全量数据
* 迁移增量数据
* 启用新数据库

# 2 准备迁移环境
本阶段主要准备迁移过程所需的环境和资源。

## 2.1 检查是否启用源数据库binlog
首先检查源MySQL的版本信息。在本文中，源MySQL版本是“5.7.24-log MySQL Community Server”
```
[root@srv011 ~]# mysql -usbtest -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 82
Server version: 5.7.24-log MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```

要实现增量导入，需确保源数据库开启了binlog。访问源数据库，执行如下命令,确保log_bin变量的值是“ON"。
```
mysql> show variables like 'log_%';
+----------------------------------------+--------------------------------+
| Variable_name                          | Value                          |
+----------------------------------------+--------------------------------+
| log_bin                                | ON                             |
| log_bin_basename                       | /var/lib/mysql/mysql-bin       |
| log_bin_index                          | /var/lib/mysql/mysql-bin.index |
```
如果发现log_bin的值是OFF，则需在MySQL Server配置文件/etc/my.cnf中增加如下内容，然后重新启动MySQL数据库服务。
 ```
server-id=1
log-bin=mysql-bin
```
## 2.2 检查源数据库大小
为了准确规划目标MySQL数据库配置，并对迁移时间有个初步估算，建议查看源数据库大小，可执行如下命令。
```
mysql> select concat(round(sum(DATA_LENGTH/1024/1024),2),'MB') as data from information_schema.TABLES where table_schema='sbtest';
+-----------+
| data      |
+-----------+
| 8384.63MB |
+-----------+
1 row in set (0.07 sec)
```
通过上述命令得知所需迁移的数据大小为8384.63MB。

## 2.3 在京东云上创建RDS实例、账户和数据库
登录京东云控制，创建MySQL实例，配置的规格(CPU、内存和存储空间)可根据源数据库的规格确定，同时创建账户(比如user001)和目标数据库(比如sbtest)。京东云除了提供图形化的控制台操作外，还支持命令行。比如可通过如下命令获得京东云RDS的相关信息。
```
MacBook:~ user$ jdc rds  describe-instance-attributes --instance-id mysql-fqiawnyind
{
    "error": null, 
    "result": {
        "dbInstanceAttributes": {
            "engine": "MySQL", 
            "instancePort": "3306", 
            "vpcId": "vpc-jr80yvv7s5", 
            "azId": [
                "cn-north-1a", 
                "cn-north-1b"
            ], 
            "instanceCPU": 2, 
            "connectionMode": "standard", 
            "instanceId": "mysql-fqiawnyind", 
            "publicDomainName": null, 
            "regionId": "cn-north-1", 
            "charge": {
                "chargeRetireTime": null, 
                "chargeStartTime": "2018-11-01T06:25:43Z", 
                "chargeExpiredTime": null, 
                "chargeStatus": "normal", 
                "chargeMode": "postpaid_by_duration"
            }, 
            "createTime": "2018-11-01T14:25:46", 
            "internalDomainName": "jddb-cn-north-1-e7501da9ca874bd0.jcloud.com", 
            "instanceClass": "db.mysql.s1.large", 
            "instanceStatus": "RUNNING", 
            "engineVersion": "5.7", 
            "subnetId": "subnet-u9xo0bo7ru", 
            "instanceMemoryMB": 8192, 
            "instanceName": "ymq_instance003", 
            "instanceType": "master", 
            "instanceStorageGB": 200, 
            "auditStatus": "off"
        }
    }, 
    "request_id": "bfds7mbod67md018s69iechwnjdt0ajc"
}
```
通过如下命令获得该实例的数据库信息。
```
MacBook:~ user$ jdc rds  describe-databases  --instance-id mysql-fqiawnyind
{
    "error": null, 
    "result": {
        "databases": [
            {
                "accessPrivilege": [
                    {
                        "privilege": "rw", 
                        "accountName": "user001"
                    }
                ], 
                "characterSetName": "utf8", 
                "dbStatus": null, 
                "dbName": "sbtest"
            }
        ]
    }, 
    "request_id": "bfds91238ju3m48j1t7kg9ok6vp7p9pw"
}
```
## 2.3 准备对象存储环境
对象存储的价格便宜[(京东云参考价格：.00427元/GB/天)](https://docs.jdcloud.com/cn/object-storage-service/price-overview)，而且把数据上传到对象存储是免收流量费，通过内网从对象存储下载到云主机也免收流量费，因此，当数据库全量数据较大时，采用对象存储完成现有MySQL环境到京东云的数据中转。


假设100G数据，在京东云对象存储上存储1天，只需花费0.427元，而且无需在京东云上产生额外的带宽成本。

操作京东云对象存储可采用京东云控制台，但对传输大数据文件(>1GB)，建议采用京东云提供的s3cmd(参考：[使用S3cmd管理京东云OSS](https://docs.jdcloud.com/cn/object-storage-service/s3cmd))。在本地上传带宽不限制的情况下，京东云对象存储的上传带宽能达到10MB/秒。下一命令展示如何上传文件到京东云对象存储，以及传输网络速度。
```
[root@srv001 ~]# time ./s3cmd/s3cmd put share.tar.1 s3://solution
WARNING: Module python-magic is not available. Guessing MIME types based on file extensions.
upload: 'share.tar.1' -> 's3://solution/share.tar.1'  [part 1 of 20, 15MB] [1 of 1]
 15728640 of 15728640   100% in    1s    10.93 MB/s  done
upload: 'share.tar.1' -> 's3://solution/share.tar.1'  [part 2 of 20, 15MB] [1 of 1]
 15728640 of 15728640   100% in    1s    11.90 MB/s  done
upload: 'share.tar.1' -> 's3://solution/share.tar.1'  [part 3 of 20, 15MB] [1 of 1]
 15728640 of 15728640   100% in    1s    11.45 MB/s  done
```
说明：在京东云上，如果云主机和对象存储是跨region访问，需要在云主机host上配置对象存储服务的endpoint对应的机器名和公网IP映射。比如，下面是在上海地域的云主机访问北京地域的对象存储时的hosts文件配置，其中solution是bucket名字。
```ini
101.124.27.193  s3.cn-north-1.jcloudcs.com
101.124.27.129  solutions.s3.cn-north-1.jcloudcs.com
```
## 2.3 准备京东云RDS操作客户端
在京东云上创建一台云主机，该云主机和前面创建的京东云RD实例位于同一个VPC。此外，安装MySQL客户端。执行如下命令，验证京东云RDS实例能正常访问。
```
[root@srv011 ~]# mysql -uuser001 -p  -h jddb-cn-north-1-e7501da9ca874bd0.jcloud.com  -Dsbtest
Enter password: 
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 81187
Server version: 5.7.21-log Source distribution

Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
```
# 2 迁移全量数据
## 2.1 导出全量数据
为了保证导出全量数据的过程中源数据库依然可以读写，需要在启动dump时，重新生成新的binlog文件。下面首先查看当前的binlog。
```
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       928 |
| mysql-bin.000002 |      1406 |
| mysql-bin.000003 |      1690 |
| mysql-bin.000004 |       341 |
+------------------+-----------+
4 rows in set (0.00 sec)
```
然后执行如下命令dump数据库全量。
```
mysqldump -u sbtest -pPassw0rd --databases sbtest --single-transaction --flush-logs>sbtest.sql
```
在启动dump命令后，将形成新的binlog文件mysql-bin.000005。
```
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000001 |       928 |
| mysql-bin.000002 |      1406 |
| mysql-bin.000003 |      1690 |
| mysql-bin.000004 |       341 |
| mysql-bin.000005 |       694 |
+------------------+-----------+
5 rows in set (0.00 sec)
```
在执行dump过程中，对数据库执行的任何修改操作，将被记录在mysql-bin.000005文件中。在dump完成后，将形成SQL文件。
```
[root@srv011 ~]# ls -al sbtest.sql
-rw-r--r--   1 root root 8043029672 Nov  2 15:30 sbtest.sql
```
为了减少文件传输，可执行gzip命令先压缩。
```
[root@srv011 ~]# gzip sbtest.sql
[root@srv011 ~]# ls -al *.gz
-rw-r--r-- 1 root root 3851542486 Nov  2 15:30 sbtest.sql.gz
```
## 2.2 传输全量数据
执行s3cmd命令，把全量数据上传到京东云对象存储。从下面命令可看出，通过公网上传数据到京东云对象存储的网速约10MB/秒。

```
[root@srv011 ~]# ls -al *.gz
-rw-r--r-- 1 root root 3851542486 Nov  2 15:30 sbtest.sql.gz
[root@srv011 ~]# time s3cmd/s3cmd put sbtest.sql.gz   s3://solution
upload: 'sbtest.sql.gz' -> 's3://solution/sbtest.sql.gz'  [part 245 of 245, 13MB] [1 of 1]
 13754326 of 13754326   100% in    0s    14.65 MB/s  done
...
real	5m21.783s
user	0m24.538s
sys	0m5.214s
```
通过s3cmd命令显示上传后的文件信息。
```
root@srv011 ~]# s3cmd/s3cmd info s3://solution/sbtest.sql.gz
s3://solution/sbtest.sql.gz (object):
   File size: 3851542486
   Last mod:  Fri, 02 Nov 2018 08:19:02 GMT
   MIME type: application/octet-stream
   Storage:   STANDARD
   MD5 sum:   13307a85cb7eabc2e22479e72e637149-1
   SSE:       none
   Policy:    none
   CORS:      none
   ACL:       none
```
## 2.3 导入全量数据
在京东云服务器，执行s3cmd命令完成全量数据文件下载，并解压文件。由于云主机和对象存储之间是高速内网，所以下载速度很快。
```
root@srv011 target]# ~/s3cmd/s3cmd get s3://solution/sbtest.sql.gz 
download: 's3://solution/sbtest.sql.gz' -> './sbtest.sql.gz'  [1 of 1]
 3851542486 of 3851542486   100% in   32s   114.37 MB/s  done

[root@srv011 target]# <b>gzip</b> -d sbtest.sql.gz
[root@srv011 target]# ls -al *.sql
-rw-r--r-- 1 root root 8043029672 Nov  2 08:19 sbtest.sql
```
执行全量数据导入，利用time命令统计数据导入所消耗的时长。为了缩短导入时长，可在执行mysqldump命令导出全量数据时，可增加--net-buffer-length=5000000选项，把单条insert语句的最大值从1M修改为5M，从而减少insert命令的个数。
```
root@srv011 target]#  time  mysql -uuser001 -pPassw0rd  -h jddb-cn-north-1-e7501da9ca874bd0.jcloud.com -Dsbtest<sbtest.sql
mysql: [Warning] Using a password on the command line interface can be insecure.
real	14m52.248s
user	0m55.703s
sys	0m23.169s
```
# 3 迁移增量数据
<font color="#dd00dd">**浅红色文字**</font>
迁移增量数据是指导入在开始导出全量数据时间点后形成的新的binlog文件，该操作需要数据库账号具有超级权限。在正常情况下，京东云RDS账户不具有超级权限。
## 3.1 提交工单，申请超级用户权限
<font color=">
正常情况下，用户不具备超级权限，执行show master logs命令将报错。
```
mysql> show master logs;
ERROR 1227 (42000): Access denied; you need (at least one of) the SUPER, REPLICATION CLIENT privilege(s) for this operation
```
用户通过控制台提交工单，并在工单中提供RDS实例ID和账号。京东云后台运维人员完成设置后，将能成功执行show master logs命令。
```
mysql> show master logs;
+------------------+-----------+
| Log_name         | File_size |
+------------------+-----------+
| mysql-bin.000034 |       384 |
| mysql-bin.000035 |       241 |
| mysql-bin.000036 |       384 |
```
## 3.2 设置源数据库只读
为了避免在导入增量数据的过程中产生新数据，可把源数据库设置为只读，这样包括具有超级权限的账户也不能修改数据库。
```
mysql> show global variables like "%read_only%";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_read_only      | OFF   |
| read_only             | OFF   |
| super_read_only       | OFF   |
| transaction_read_only | OFF   |
| tx_read_only          | OFF   |
+-----------------------+-------+
5 rows in set (0.05 sec)

mysql> set global read_only=1;
Query OK, 0 rows affected (0.00 sec)

mysql> show global variables like "%read_only%";
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_read_only      | OFF   |
| read_only             | ON    |
| super_read_only       | OFF   |
| transaction_read_only | OFF   |
| tx_read_only          | OFF   |
+-----------------------+-------+
5 rows in set (0.01 sec)

mysql> flush tables with read lock;
Query OK, 0 rows affected (0.00 sec)

mysql> update sbtest1 set pad="001" where id = 1;
ERROR 1223 (HY000): Can't execute the query because you have a conflicting read lock
```
## 3.3 导入binlog增量
从源数据库的/var/lib/mysql目录获得新增的binlog文件（本文新增的binlog文件是mysql-bin.000005)
通过执行如下命令:
```
mysqlbinlog /var/lib/mysql/mysql-bin.000005
```
将能看到如下信息：
```
# at 1298
#181102 17:48:45 server id 1  end_log_pos 1366 CRC32 0xf0bca784         Query   thread_id=118   exec_time=0     error_code=0
SET TIMESTAMP=1541152125/*!*/;
BEGIN
/*!*/;
# at 1366
#181102 17:48:45 server id 1  end_log_pos 1425 CRC32 0x19442e87         Table_map: `sbtest`.`sbtest1` mapped to number 214
# at 1425
#181102 17:48:45 server id 1  end_log_pos 1733 CRC32 0x9337c7e5         Update_rows: table id 214 flags: STMT_END_F

BINLOG '
fR3cWxMBAAAAOwAAAJEFAAAAANYAAAAAAAEABnNidGVzdAAHc2J0ZXN0MQAEAwP+/gT+eP48AIcu
RBk=
fR3cWx8BAAAANAEAAMUGAAAAANYAAAAAAAEAAgAE///wAQAAANs3DwB3MzE0NTEzNzM1ODYtMTU2
ODgxNTM3MzQtNzk3Mjk1OTM2OTQtOTY1MDkyOTk4MzktODM3MjQ4OTgyNzUtODY3MTE4MzM1Mzkt
Nzg5ODEzMzc0MjItMzUwNDk2OTA1NzMtNTE3MjQxNzM5NjEtODc0NzQ2OTYyNTMDMDAx8AEAAADb
Nw8AdzMxNDUxMzczNTg2LTE1Njg4MTUzNzM0LTc5NzI5NTkzNjk0LTk2NTA5Mjk5ODM5LTgzNzI0
ODk4Mjc1LTg2NzExODMzNTM5LTc4OTgxMzM3NDIyLTM1MDQ5NjkwNTczLTUxNzI0MTczOTYxLTg3
NDc0Njk2MjUzCW5ldyB2YWx1ZeXHN5M=
'/*!*/;
# at 1733
#181102 17:48:45 server id 1  end_log_pos 1764 CRC32 0xd1afeacf         Xid = 1338
COMMIT/*!*/;

```
在上述内容本质是对数据库进行了一次update操作，其中1366是起始点，1764是结束点。基于上述信息，执行如下命令形成增量SQL文件。

```
root@srv011 target]# mysqlbinlog --skip-gtids --start-position=1366 --stop-position=1764  /var/lib/mysql/mysql-bin.000005 > `date +%F-%T`-add.sql
```
形成的增量SQL文件内容如下：
```
[root@srv011 target]# cat 2018-11-02-17\:51\:24-add.sql
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=1*/;
/*!50003 SET @OLD_COMPLETION_TYPE=@@COMPLETION_TYPE,COMPLETION_TYPE=0*/;
DELIMITER /*!*/;
# at 4
#181102 15:28:17 server id 1  end_log_pos 123 CRC32 0x27c44c40 	Start: binlog v 4, server v 5.7.24-log created 181102 15:28:17
# Warning: this binlog is either in use or was not closed properly.
BINLOG '
kfzbWw8BAAAAdwAAAHsAAAABAAQANS43LjI0LWxvZwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAEzgNAAgAEgAEBAQEEgAAXwAEGggAAAAICAgCAAAACgoKKioAEjQA
AUBMxCc=
'/*!*/;
/*!50616 SET @@SESSION.GTID_NEXT='AUTOMATIC'*//*!*/;
# at 1366
#181102 17:48:45 server id 1  end_log_pos 1425 CRC32 0x19442e87 	Table_map: `sbtest`.`sbtest1` mapped to number 214
# at 1425
#181102 17:48:45 server id 1  end_log_pos 1733 CRC32 0x9337c7e5 	Update_rows: table id 214 flags: STMT_END_F

BINLOG '
fR3cWxMBAAAAOwAAAJEFAAAAANYAAAAAAAEABnNidGVzdAAHc2J0ZXN0MQAEAwP+/gT+eP48AIcu
RBk=
fR3cWx8BAAAANAEAAMUGAAAAANYAAAAAAAEAAgAE///wAQAAANs3DwB3MzE0NTEzNzM1ODYtMTU2
ODgxNTM3MzQtNzk3Mjk1OTM2OTQtOTY1MDkyOTk4MzktODM3MjQ4OTgyNzUtODY3MTE4MzM1Mzkt
Nzg5ODEzMzc0MjItMzUwNDk2OTA1NzMtNTE3MjQxNzM5NjEtODc0NzQ2OTYyNTMDMDAx8AEAAADb
Nw8AdzMxNDUxMzczNTg2LTE1Njg4MTUzNzM0LTc5NzI5NTkzNjk0LTk2NTA5Mjk5ODM5LTgzNzI0
ODk4Mjc1LTg2NzExODMzNTM5LTc4OTgxMzM3NDIyLTM1MDQ5NjkwNTczLTUxNzI0MTczOTYxLTg3
NDc0Njk2MjUzCW5ldyB2YWx1ZeXHN5M=
'/*!*/;
# at 1733
#181102 17:48:45 server id 1  end_log_pos 1764 CRC32 0xd1afeacf 	Xid = 1338
COMMIT/*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```
执行数据库增量SQL文件，然后查询数据库确认所进行的增量修改已同步在京东云RDS目标库中。
```
[root@srv011 target]#  mysql -uuser001 -pPassw0rd  -h jddb-cn-north-1-e7501da9ca874bd0.jcloud.com -e "select * from sbtest.sbtest1 where id = 1"
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+--------+-------------------------------------------------------------------------------------------------------------------------+-----------+
| id | k      | c                                                                                                                       | pad       |
+----+--------+-------------------------------------------------------------------------------------------------------------------------+-----------+
|  1 | 997339 | 31451373586-15688153734-79729593694-96509299839-83724898275-86711833539-78981337422-35049690573-51724173961-87474696253 | new value |
+----+--------+-------------------------------------------------------------------------------------------------------------------------+-----------+
```
在完成增量更新后，可及时通过工单系统联系京东云运维，及时释放账户的超级权限。
# 4 启用新数据库
通常再完成迁移后，尽快去验证源数据库和RDS数据库是否一致，并根据性能需求升级RDS配置。
# 5 小结
本文介绍了自建MySQL数据迁移到京东云RDS的全过程，同时通过先导入全量，然后再导入增量的方法，大大缩短迁移过程中数据库不可更改的时长。当然在源数据库数据量不大，同时允许数据库不可更改时间较长时，在进行全量导出时就对源数据进行锁定，这样一次全量导入就能把源数据库中的数据完全迁移到京东云RDS。
# 6 参考文献
* 京东云云数据库RDS: [https://docs.jdcloud.com/cn/rds/product-overview](https://docs.jdcloud.com/cn/rds/product-overview)
* 京东云对象存储：[https://docs.jdcloud.com/cn/object-storage-service/product-overview](https://docs.jdcloud.com/cn/object-storage-service/product-overview)
* 京东云对象存储S3CMD: [https://docs.jdcloud.com/cn/object-storage-service/s3cmd](https://docs.jdcloud.com/cn/object-storage-service/s3cmd)
* 京东云命令行：[https://docs.jdcloud.com/cn/cli/introduction](https://docs.jdcloud.com/cn/cli/introduction)