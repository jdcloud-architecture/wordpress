```
```bash
[root@srv011 ~]# wget http://repo.mysql.com/mysql57-community-release-el7.rpm
--2018-10-31 14:13:56--  http://repo.mysql.com/mysql57-community-release-el7.rpm
Resolving repo.mysql.com (repo.mysql.com)... 104.118.86.179
Connecting to repo.mysql.com (repo.mysql.com)|104.118.86.179|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 25680 (25K) [application/x-redhat-package-manager]
Saving to: ‘mysql57-community-release-el7.rpm’

100%[========================================================>] 25,680       137KB/s   in 0.2s   

2018-10-31 14:13:56 (137 KB/s) - ‘mysql57-community-release-el7.rpm’ saved [25680/25680]

[root@srv011 ~]# rpm -ivh mysql57-community-release-el7.rpm
warning: mysql57-community-release-el7.rpm: Header V3 DSA/SHA1 Signature, key ID 5072e1f5: NOKEY
Preparing...                          ################################# [100%]
Updating / installing...
   1:mysql57-community-release-el7-11 ################################# [100%]

[root@srv011 ~]# yum install -y mysql-server

[root@izm5ebuyhgwtrtioto5ziyz ~]# chkconfig mysqld on
Note: Forwarding request to 'systemctl enable mysqld.service'.
[root@izm5ebuyhgwtrtioto5ziyz ~]# service mysqld restart
Redirecting to /bin/systemctl restart mysqld.service
[root@izm5ebuyhgwtrtioto5ziyz ~]# service mysqld status
Redirecting to /bin/systemctl status mysqld.service
● mysqld.service - MySQL Server
   Loaded: loaded (/usr/lib/systemd/system/mysqld.service; enabled; vendor preset: disabled)
   Active: active (running) since Sun 2018-11-04 16:30:09 CST; 4s ago
     Docs: man:mysqld(8)
           http://dev.mysql.com/doc/refman/en/using-systemd.html
  Process: 2753 ExecStart=/usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid $MYSQLD_OPTS (code=exited, status=0/SUCCESS)
  Process: 2679 ExecStartPre=/usr/bin/mysqld_pre_systemd (code=exited, status=0/SUCCESS)
 Main PID: 2757 (mysqld)
   CGroup: /system.slice/mysqld.service
           └─2757 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

Nov 04 16:30:04 izm5ebuyhgwtrtioto5ziyz systemd[1]: Starting MySQL Server...
Nov 04 16:30:09 izm5ebuyhgwtrtioto5ziyz systemd[1]: Started MySQL Server.


root@izm5ebuyhgwtrtioto5ziyz ~]# grep "temporary password" /var/log/mysqld.log
2018-11-04T08:30:05.475184Z 1 [Note] A temporary password is generated for root@localhost: 4wy=eh:fI#,F

[root@izm5ebuyhgwtrtioto5ziyz ~]# mysql -uroot -p

mysql> set password=password('Passw0rd@123');
Query OK, 0 rows affected, 1 warning (0.00 sec)

```

阿里云测试结果

```
[root@izm5ebuyhgwtrtioto5ziyz ~]#  sysbench cpu run --threads=1
sysbench 1.0.9 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 1
Initializing random number generator from current time


Prime numbers limit: 10000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:   894.22

General statistics:
    total time:                          10.0001s
    total number of events:              8944

Latency (ms):
         min:                                  1.10
         avg:                                  1.12
         max:                                  1.46
         95th percentile:                      1.14
         sum:                               9995.19

Threads fairness:
    events (avg/stddev):           8944.0000/0.00
    execution time (avg/stddev):   9.9952/0.00

[root@izm5ebuyhgwtrtioto5ziyz ~]#  sysbench cpu run --threads=2
sysbench 1.0.9 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 2
Initializing random number generator from current time


Prime numbers limit: 10000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:  1555.83

General statistics:
    total time:                          10.0012s
    total number of events:              15563

Latency (ms):
         min:                                  1.11
         avg:                                  1.28
         max:                                  3.16
         95th percentile:                      1.32
         sum:                              19993.74

Threads fairness:
    events (avg/stddev):           7781.5000/29.50
    execution time (avg/stddev):   9.9969/0.00

[root@izm5ebuyhgwtrtioto5ziyz ~]#  sysbench cpu run --threads=4
sysbench 1.0.9 (using system LuaJIT 2.0.4)

Running the test with following options:
Number of threads: 4
Initializing random number generator from current time


Prime numbers limit: 10000

Initializing worker threads...

Threads started!

CPU speed:
    events per second:  1549.72

General statistics:
    total time:                          10.0015s
    total number of events:              15502

Latency (ms):
         min:                                  1.26
         avg:                                  2.58
         max:                                 36.30
         95th percentile:                     12.30
         sum:                              39937.67

Threads fairness:
    events (avg/stddev):           3875.5000/38.81
    execution time (avg/stddev):   9.9844/0.01

[root@izm5ebuyhgwtrtioto5ziyz ~]# 
```