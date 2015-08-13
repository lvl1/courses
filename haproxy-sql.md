Title: HAProxy with MySQL
Date: 2015-08-12 18:48
Category: Tutorial
Id: 010111

#HAProxy with MySQL

Setup new slave same as previous tutorial

Run on both slaves in mysql configuration:
```
INSERT INTO mysql.user (Host,User) values ('haproxy-ip','haproxy_check'); FLUSH PRIVILEGES;
```
```
GRANT ALL PRIVILEGES ON *.* TO 'haproxy_root'@'haproxy-ip' IDENTIFIED BY 'password' WITH GRANT OPTION; FLUSH PRIVILEGES;
```

Change `ENABLED=0` to `ENABLED=1` in /etc/default/haproxy to start haproxy whenever the server reboots

```
vim /etc/haproxy/haproxy.cfg
```
```
global
    log 127.0.0.1 local0 notice
    user haproxy
    group haproxy

defaults
    log global
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

Check load balancing with following command on haproxy server:
```
mysql -h 127.0.0.1 -u haproxy_root -p -e "show variables like 'server_id'"
```
