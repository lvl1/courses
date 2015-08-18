Title: SQL Master Slave
Date: 2015-08-11 12:02
Category: Tutorial
Id: 010110


#SQL Master Slave

##Configure Master:

```
sudo apt-get install mysql-server python-mysqldb
```

```
sudo vim /etc/mysql/my.cnf:
```
comment out `bind-address = 127.0.0.1`
uncomment `#server-id = 1`
uncomment `#log_bin = /var/log/mysql/mysql-bin.log`
change `#binlog_do_db = include_database_name` to `binlog_do_db = comments_db`

Exit and restart MySQL with:
```
sudo service mysql restart
```
```
mysql -u root -p
```
```
GRANT REPLICATION SLAVE ON *.* TO 'sql-slave'@'%' IDENTIFIED BY 'password';
```
```
FLUSH PRIVILEGES;
```
In NEW terminal (this part is a little finnicky. It has to be done in a new terminal than the one used previous):
```
USE comments_db
```
```
FLUSH TABLES WITH READ LOCK;
```

Write down output of following command for later reference:
```
SHOW MASTER STATUS;
```

In second window at linux prompt:
```
mysqldump -u root -p --opt comments_db > /opt/comments_db.sql
```

Back in first window:
```
UNLOCK TABLES;
```
```
QUIT;
```

##Configure Slave

```
sudo apt-get install mysql-server python-mysqldb
```

Copy .sql file from master:
```
scp root@master-ip:/opt/comments_db.sql .
```
```
mysql -u root -p
```
```
CREATE DATABASE comments_db;
```
At linux prompt:
```
mysql -u root -p comments_db < comments_db.sql
```
```
sudo vim /etc/mysql/my.cnf
```
Make sure lines are like the following:
```
server-id = 2
```
```
#bind-address = 127.0.0.1
```
```
log_bin = /var/log/mysql/mysql-bin.log
```
```
binlog_do_db = comments_db
```

Add to bottom of file:
```
relay-log = /var/log/mysql/mysql-relay-bin.log
```
```
sudo service mysql restart
```
```
mysql -u root -p
```

Make sure to put your own values in for master-ip, password, MASTER_LOG_FILE, and MASTER_LOG_POS:
```
CHANGE MASTER TO MASTER_HOST='master-ip',MASTER_USER='sql-slave', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=  330;
```
```
START SLAVE;
```

Check status with:
```
SHOW SLAVE STATUS\G
```

Run following sql command on both master and slave to allow remote root access:
```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'password' WITH GRANT OPTION;
```

Change the ip’s that are accessed by falcon in comments.py (/usr/share/nginx/html/comments.py)
