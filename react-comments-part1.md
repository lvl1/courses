Title: React Comments Part1
Date: 2015-07-27 12:26
Category: Tutorial

#NGINX, uWSGI, Falcon, Supervisor Tutorial

Our end goal - a dynamic React comment box in a static Pelican Blog - requires multiple components. At the front end, NGINX handles the initial browser requests. It will act as a reverse-proxy that will pass on requests to uWSGI. uWSGI will allow us to run multiple threads of Falcon in the background. Falcon is a python framework that provides a groundwork for building our own custom API. This custom API will, in turn, be used to update and query a MySQL database (which will hold the information used in our comment system, such as comments and emails).

To recap, our comment system will operate like the following:
```
                           Supervisor Starts uWSGi --> uWSGI (Spins Up Falcon Instances)
                                                                       |
                                                                       V
(Comment submitted in browser) --> NGINX (Reverse proxy) --> Falcon (Our custom API) --> MySQL (Database for storing comments)
```
Each individual component will be explained in detail in the remainder of the tutorials for this series.

Configuring a SQL database

SQL (Structured Language Query) is used to communicate with a database. It is the standard language for relational database management. Simply put, a database is a structured set of data usually arranged in tables of rows and columns. To create our database, we will be using MySQL: an open-source database management system.

On your Ubuntu server, execute the following command to refresh the package database:
```
sudo apt-get update
```
Next, install mysql-server and python-mysqldb. Python-mysqldb will be used to access the SQL database via python code (which we will be doing with Falcon later in the tutorial).
```
sudo apt-get install mysql-server python-mysqldb
```
During the install, you will be asked to specify a password. The password is needed for when you access the SQL database.

That is it for the installation! Log into the database with the following command and the password you configured(mysql -p can also be used if you are already logged in as root).
```
mysql -u root -p
```
With SQL, there are two simple rules of thumb for entering commands. Firstly, all commands must end with a semicolon. Without a semicolon, the command will not be executed. Secondly, commands are usually written in uppercase and databases, tables, usernames, or text are written in lowercase. This is optional (the MySQL command line is not case sensitive) but is a common practice to make commands easier to distinguish.

MySQL organizes its information into databases. Each database can hold multiple tables which hold specific data.

You can check what databases are available by using “SHOW DATABASES;”
Your output will look similar to the following:
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
Now we are going to create our own database called comments:
```
CREATE DATABASE comments_db;
```
(Optionally) To delete a database, use the following command:
```
DROP DATABASE comments_db;
```
Now that we have our comments database created, we need to open it in order to use it.
```
USE comments_db;
```
To show tables that are in the database, use the SHOW command. Because we have not created any tables yet, an “Empty set” message will be shown.
```
SHOW tables;
```
It’s time to create our first table. Execute the following command to create a table called comments_table:
```
CREATE TABLE comments_table (id INT NOT NULL PRIMARY KEY AUTO_INCREMENT, comment VARCHAR(5000), 
name VARCHAR(50), 
timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
```
Let’s break that command down. Each phrase separated by a comma in the brackets creates an individual column. We have made four columns - id, comment, name, and timestamp.
The “id” column makes a column that automatically numbers each row.
The “comment” column will be used to store the comments. VARCHAR(5000) limits the column to 5000 characters.
The “name” column will hold the name of the user. Again, VARCHAR(50) limits the column to 50 characters.
Finally, “timestamp” creates an automatic timestamp. This timestamp is based off of the system time configured on the server.

To verify that our table was created, execute the “SHOW TABLES;” command:
Your output should be similar to the following:
```
mysql> SHOW TABLES;
+------------------+
| Tables_in_falcon |
+------------------+
| comments         |
+------------------+
1 row in set (0.00 sec)
```
You can see the format of the table with the “DESCRIBE comments;” command”
```
mysql> DESCRIBE comments_table;
+-----------+---------------+------+-----+-------------------+----------------+
| Field     | Type          | Null | Key | Default           | Extra          |
+-----------+---------------+------+-----+-------------------+----------------+
| id        | int(11)       | NO   | PRI | NULL              | auto_increment |
| comment   | varchar(5000) | YES  |     | NULL              |                |
| name      | varchar(50)   | YES  |     | NULL              |                |
| timestamp | timestamp     | NO   |     | CURRENT_TIMESTAMP |                |
+-----------+---------------+------+-----+-------------------+----------------+
4 rows in set (0.00 sec)
```
To see the individual details of your table, use "SELECT * FROM comments;”
```
mysql> SELECT * FROM comments;
Empty set (0.00 sec)
```
Because there has not been data added to the table, the output will be “Empty set”

(Additionally) 
Although it is not required for this tutorial (we will be using falcon to add information), you can add data to the table with the following command:
```
INSERT INTO `comments_table` (`id`,`comment`,`name`,`timestamp`) VALUES (NULL, "Hello!", "FalconLover", NULL);
```
Read more about MySQL here: https://www.digitalocean.com/community/tutorials/a-basic-mysql-tutorial 
Or here: http://dev.mysql.com/doc/

##Falcon

In a production environment, thousands of requests may be made to a web server at a time. This puts a heavy load on the server application which is trying to respond to the thousands of requests on a first-come, first-serve basis. The solution to this problem is using a threaded “application container” to manage the requests. To accomplish this threading (also known as multi-process) we will use uWSGI. We are choosing uWSGI because of it’s built-in support for integration with NGINX. Threading allows us to have potentially thousands of Falcon instances running simultaneously.

Run the following commands to install falcon using pip. Pip is a package manager that is an alternative to apt-get.
```
sudo apt-get install python-pip
sudo pip install falcon --upgrade
```
To do a quick test of Falcon, we are going to make a project folder in /opt. We will later be 
sudo mkdir /opt/comments/
```
vim /opt/comments/comments.py:
```
```
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
```

comment line “include /etc/nginx/sites-enabled/*;” in /etc/nginx/nginx.conf
add to bottom of http { } section of /etc/nginx/nginx.conf:
```         
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
```
sudo service nginx restart
```
##uWSGI
```
sudo apt-get install python-dev
sudo pip install uwsgi
```
Test with:
```
uwsgi --wsgi-file /opt/comments/comments.py --callable app -s 127.0.0.1:8080
```
Supervisor
```
sudo apt-get install supervisor
```
add to bottom of /etc/supervisor/supervisord.conf:
```
[program:uwsgi]
command=uwsgi --wsgi-file /opt/comments/comments.py --callable app -s 127.0.0.1:8080
autostart=true
autorestart=true
startsecs=2
```
```
vim /etc/init/supervisor.conf
```
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
```
sudo reboot
```
Browse to yourip/sqlaccess

