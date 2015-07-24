Title: React Comments Part1
Date: 2015-07-23 12:26
Category: Tutorial

#NGINX, uWSGI, Falcon, Supervisor Tutorial

Our end goal - a dynamic React comment box in a static Pelican Blog - requires multiple components. At the front end, NGINX handles the initial browser requests. It will act as a reverse-proxy that will pass on requests to uWSGI. uWSGI will allow us to run multiple threads of Falcon in the background. Falcon is a python framework that provides a groundwork for building our own custom API. This custom API will, in turn, be used to update and query a MySQL database (which will hold the information used in our comment system, such as comments and emails).

To recap, our comment system will operate like the following:

(Comment submitted in browser) --> NGINX (Reverse proxy) --> uWSGI (Application container) --> Falcon (Our custom API) --> MySQL (Database for storing comments)

Additionally (not shown above), Supervisor will be used to make sure that uWSGI is running whenever the server is rebooted. 

Each individual component will be explained in detail in the remainder of the tutorials for this series.

##Configuring a SQL database

`sudo apt-get update python-mysqldb`

`sudo apt-get install mysql-server`

Enter password for sql database when prompted

`mysql -u root -p`

Enter password to log in
```
mysql> SHOW DATABASES;
+---------------------+
| Database            |
+---------------------+
| information_schema  |
| mysql               |
| performance_schema  |
+---------------------+
3 rows in set (0.00 sec)
```
Create database with:

`CREATE DATABASE database_name;`

Delete database with:

`DROP DATABASE database_name;`

`USE database_name;`

`SHOW tables;`
```
CREATE TABLE comments (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, comment VARCHAR(5000), name VARCHAR(50), timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
```
```
mysql> SHOW TABLES;
+------------------+
| Tables_in_falcon |
+------------------+
| comments         |
+------------------+
1 row in set (0.00 sec)
```
`DESCRIBE comments;`

`SELECT * FROM comments;`

Read more here: https://www.digitalocean.com/community/tutorials/a-basic-mysql-tutorial or here: http://dev.mysql.com/doc/

##Falcon

In a production environment, thousands of requests may be made to a web server at a time. This puts a heavy load on the server application which is trying to respond to the thousands of requests on a first-come, first-serve basis. The solution to this problem is using a threaded “application container” to manage the requests. To accomplish this threading (also known as multi-process) we will use uWSGI. We are choosing uWSGI because of it’s built-in support for integration with NGINX. Threading allows us to have potentially thousands of Falcon instances running simultaneously.
```
sudo apt-get install python-pip
sudo pip install falcon --upgrade
sudo mkdir /opt/comments/
```
vim /opt/comments/comments.py:

```python
# comments.py
# Let's get this party started


import falcon

class CommentsResource:
    def on_get(self, req, resp):
        """Handles GET requests"""
        resp.status = falcon.HTTP_200  # This is the default status
        resp.body = ('\nTwo things awe me most, the starry sky '
                     'above me and the moral law within me.\n'
                     '\n'
                     '    ~ Immanuel Kant\n\n')

# falcon.API instances are callable WSGI apps
app = falcon.API()

# Resources are represented by long-lived class instances
comments = CommentsResource()

# things will handle all requests to the '/things' URL path
app.add_route('/sqlaccess', comments)
```

##NGINX

```
sudo apt-get install nginx

comment line “include /etc/nginx/sites-enabled/*;” in /etc/nginx/nginx.conf
add to bottom of http { } section of /etc/nginx/nginx.conf:
         
        # Configuration for Nginx
        server {
                
                # Running port
                listen 80;
                        
                # Settings to by-pass for static files 
                location ^~ /static/  {
                
                        # Example:
                        # root /full/path/to/application/static/file/dir;
                        root /app/static/; 

                }
                        
                # Serve a static file (ex. favico) outside static dir.
                location = /favico.ico  {
                
                        root /app/favico.ico;
                
                }
                        
                # Proxying connections to application servers
                location /sqlaccess {
                        
                        include            uwsgi_params;
                        uwsgi_pass         127.0.0.1:8080;
                 
                        proxy_redirect     off;
		proxy_set_header   Host $host;
                        proxy_set_header   X-Real-IP $remote_addr;
                        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header   X-Forwarded-Host $server_name;
                        
        }
    } 
```

`sudo service nginx restart`

##uWSGI
```
apt-get install python-dev
pip install uwsgi
```
Test with:

`uwsgi --wsgi-file /opt/comments/comments.py --callable app -s 127.0.0.1:8080`

add to bottom of /etc/init/supervisord.conf:
```
[program:uwsgi]
command=uwsgi --wsgi-file /opt/comments/comments.py --callable app -s 127.0.0.1:8080
autostart=true
autorestart=true
startsecs=2
```

vim /etc/init/supervisor.conf
```
description     "supervisord"
start on runlevel [2345]
stop on runlevel [!2345]
respawn
respawn limit 10 5
umask 022
env SSH_SIGSTOP=1
expect stop
console none
pre-start script
        test -x /usr/bin/supervisord || { stop; exit 0; }
end script
exec /usr/bin/supervisord
```

Now reboot

Browse to yourip/sqlaccess
