# mysql-master-salve

### 搭建集群

参考链接：https://github.com/Mountains-and-rivers/mongo-replica-set

### 制作镜像

```
cd docker/master
docker build -t mysql-master:0.1 .

cd docker/slave
docker build -t mysql-slave:0.1 .

镜像已分片images 目录
cd docker/master
cat xa* > master.tar
docker load -i master.tar

cd docker/slave
cat xa* > slave.tar
docker load -i slave.tar
```

### 部署

```
#master
kubectl create -f mysql-master-rc.yaml
kubectl create -f mysql-master-service.yaml

#salve
kubectl create -f mysql-slave-rc.yaml
kubectl create -f mysql-slave-service.yaml
```

### 验证

```
kubectl get pods -o wide|grep mysql
mysql-master-f8kvw         1/1     Running   0          4m19s   10.244.1.30   node01   <none>           <none>
mysql-slave-qjjtn          1/1     Running   0          4m19s   10.244.1.31   node01   <none>           <none>


kubectl get svc|grep mysql
mysql-master             ClusterIP   10.110.242.217   <none>        3306/TCP    4m36s
mysql-slave              ClusterIP   10.107.23.217    <none>        3306/TCP    4m36s
```

### 登录

```

[root@master ~]#  kubectl exec -it mysql-master-ssx8k bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mysql-master-ssx8k:/# mysql -u root
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
root@mysql-master-ssx8k:/# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 5.7.34-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 

```

### master 验证

```
[root@master mysql-replica]# kubectl exec -ti mysql-master-f8kvw bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mysql-master-f8kvw:/#  mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 5.7.34-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> create database demo;
Query OK, 1 row affected (0.04 sec)

mysql> use demo;
Database changed
mysql>  create table user(id int(10), name char(20));
Query OK, 0 rows affected (0.03 sec)

mysql>  insert into user values(9999, 'usertest');
Query OK, 1 row affected (0.01 sec)

mysql> select * from user;
+------+----------+
| id   | name     |
+------+----------+
| 9999 | usertest |
+------+----------+
1 row in set (0.00 sec)
```

### slave 验证

```
[root@master mysql-replica]# kubectl exec -ti mysql-slave-qjjtn bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@mysql-slave-qjjtn:/#  mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.34-log MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.110.242.217
                  Master_User: demo
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-master-f8kvw-bin.000003
          Read_Master_Log_Pos: 763
               Relay_Log_File: mysql-slave-qjjtn-relay-bin.000005
                Relay_Log_Pos: 1002
        Relay_Master_Log_File: mysql-master-f8kvw-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 763
              Relay_Log_Space: 3087305
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 69aa5dbc-ada8-11eb-bfbd-1a3bfcda7167
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql>  show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| demo               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql>  use demo;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql>  select * from user;
+------+----------+
| id   | name     |
+------+----------+
| 9999 | usertest |
+------+----------+
1 row in set (0.00 sec)

mysql> 
```

### 持久化[TODO]

```
nfs

hostPath
```

### 参考：

1，https://kublr.com/blog/setting-up-mysql-replication-clusters-in-kubernetes-2/

2，https://github.com/docker-library/mysql/tree/master/5.7

