Title: HAProxy with MySQL
Date: 2015-08-12 18:48
Category: Tutorial
Id: 010111

#HAProxy with SQL

In a production environment, reliability is important. We need to make sure that our MySQL server is always available to process comments for our system. This can be accomplished using a tool called HAProxy. HAProxy allows load-balancing across two or more servers. We will create a second slave that will be used to load balance. The Pelican webserver will run HAProxy.

First, we need to build a second slave. Setup a new slave the same way as explained in the “SQL Master Slave” Tutorial. Make sure to give the new slave a unique server-id. Also, if changes have since been made to the master’s database, you will need to re-run `SHOW MASTER STATUS;` to find the appropriate log position and file.

Next, appropriate permissions need to be set on both slaves. These permissions will allow the HAProxy server to connect with the slaves.

Run on both slaves in mysql configuration:

```
INSERT INTO mysql.user (Host,User) values ('haproxy-ip','haproxy_check'); FLUSH PRIVILEGES;
```

Swap haproxy-ip for the ip address of the HAProxy server. This command adds a user account to the MySQL server that is called haproxy_check. 

Add remote access credentials with the following command:

```
GRANT ALL PRIVILEGES ON *.* TO 'haproxy_root'@'haproxy-ip' IDENTIFIED BY 'password' WITH GRANT OPTION; FLUSH PRIVILEGES;
```

This allows the SQL server to be accessed by the HAProxy server.

Now we will move on to the configuration of the HAProxy server. We will start by making sure that the HAProxy service is always started whenever the server reboots.

```
sudo vim /etc/default/haproxy
```

Change `ENABLED=0` to `ENABLED=1`

To configure the server, the HAProxy configuration file needs to be edited.

```
vim /etc/haproxy/haproxy.cfg
```

Erase the contents of the current file and add the following:

```
global
    user haproxy
    group haproxy

defaults
    retries 2
    timeout connect 3000
    timeout server 5000
    timeout client 5000

listen mysql-cluster
    bind 127.0.0.1:3306
    mode tcp
    option mysql-check user haproxy_check
    balance roundrobin
    server mysql-1 slave1-ip:3306 check
    server mysql-2 slave2-ip:3306 check

listen webserver 
    bind 0.0.0.0:8888
    mode http
    stats enable
    stats uri /
    stats realm Strictly\ Private
    stats auth user:password
```
```
service start haproxy
```

The first section of code makes sure that the HAProxy user (we added this user in the first steps of this tutorial) is used to access the slaves. The “defaults” section sets the connection timeout details (how long it takes for the slave to appear down). The “listen mysql-cluster” section tells HAProxy to connect to two different servers - mysql-1 and mysql-2. Note that slave1-ip and slave2-ip need to be changed to the ip addresses of the two slaves. This section also specifies a roundrobin load-balancing method. The last section turns on web-monitoring. Specify a user and password and you can log on to a web-gui that will show HAProxy stats at haproxy-ip:8888

That’s all that is needed for HAProxy to function properly. Check that load balancing is working correctly with the following command:

```
mysql -h 127.0.0.1 -u haproxy_root -p -e "show variables like 'server_id'"
```

If you run the above command multiple times, you should see the server-id change each time. This means that HAProxy is successfully load balancing across the two slaves!

To get this setup working with the React comments system. You need to change the GET address in /usr/share/nginx/html/comments.py to 127.0.0.1 and the user to “haproxy_root”.
