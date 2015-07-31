Title: Salt tutorial part 3
Date: 2015-07-08 11:45
Category: Tutorial
Id: 010103

#Deployment of Nginx Using Salt Formulas (Part 3 of 3)

Nginx (pronounced “engine x”) is a simple web server that we will set up on our Salt minion. We will accomplish this using Salt states. Salt allows you to define a top.sls file that holds the instructions for what packages (such as nginx) will be installed on the minion.

##Configure the nginx.sls File 

Make sure you are connected to your Salt master. You should see a prompt similar to the following:

```
saltuser@salt-master:~#
```

We’ll start by creating our nginx.sls file. The naming of this file is arbitrary: we only need to have it match our top.sls file later on. 

```
sudo mkdir /srv/salt/
sudo vim /srv/salt/nginx.sls
```

The nginx.sls file is what we define as a “state”. Within that state is the instructions we want it to include. We define these parameters within the file:

```
nginx:
  pkg:
    - installed
  service:
    - running
    - require:
      - pkg: nginx
/usr/share/nginx/html/index.html:
  file:
    - managed
    - source: salt://nginx/index.html
    - require:
      - pkg: nginx
```


Let’s break this down line by line. The first three lines specify that the nginx package should be installed. Simple enough. Next, we say that the service should be running. However, before the service is started, we require that the nginx package be installed. Next, we replace the file located at /usr/share/nginx/html/index.html on the minion with the file salt://nginx/index.html located on the master (we will create this file later). Again, this file replacement requires that the nginx package is installed first.

##Create Your HTML File

Now we need to make our custom html file. 

```
sudo vim /srv/salt/nginx/index.html
```

If you desire, you can change the following html to anything you would like on your webserver!

```html
<html>
<head><title>Salt rocks!</title></head>
<body>
<h1>This file brought to you by Salt</h1>
</body>
</html>
```

##Configure top.sls File

Now that we have created our state and html file, we need a way for the master to determine what states it should apply to each minion. To do this, we will create a top.sls file. Salt looks to find this file in the /srv/salt/ directory.

```
sudo vim /srv/salt/top.sls
```

Write this into the file:

```
base:
  ‘minion1':
    - nginx 
  ‘minion22’:
    - not-applied
```

For this example, we have a default base environment. When the minion is told to execute a highstate, it looks through this file to see what it should apply. Assuming we created a minion named “minion1”, the minion will apply the “nginx” state. Because the minion does not match “minion22”, it will not apply “not-applied” (this could be any other package or combination of packages).

##Apply the nginx State

Now apply this state by executing the following command on the salt master:

```
sudo salt 'minion1' state.highstate
```

This applies the top.sls file to minion1. You should see a success message such as the following:

```
Summary
------------
Succeeded: 3 (changed=2)
Failed:	0
------------
Total states run: 	3
```

Check that everything worked as expected by going to the ip address of your minion in your browser (the minion ip can be found in the DigitalOcean control panel).

##Additionally (Configure Minion with User Account and SSH Keys)

You can run a state for your minion that will configure a new user and SSH keys. We will do this in one state file called ‘user.sls’. 

Because we have already configured SSH keys on the Salt master for saltuser, we can copy the public SSH key from the master to the minion using our state file. In order for Salt to access the public key, we need to copy it to the salt directory. 

Log into the master as saltuser. Start by making a new directory to store the key in:

`sudo mkdir /srv/salt/keys`

Next, make a copy of the SSH key in the new directory:

`sudo cp /home/saltuser/.ssh/authorized_keys /srv/salt/keys/minion_key`

All that’s left is creating the user.sls file:

`sudo vim /srv/salt/user.sls`

Add the following to the file:
```
minionuser:
  user.present:
    - fullname: Minion User
    - shell: /bin/bash
    - home: /home/minionuser
    - groups:
      - sudo
/home/minionuser/.ssh:
  file.directory:
    - user: minionuser
    - group: minionuser
    - mode: 700
/home/minionuser/.ssh/authorized_keys:
  file:
    - managed
    - user: minionuser
    - group: minionuser
    - source: salt://keys/minion_key
    - mode: 600
```
The first section of this file creates a user. A fullname is specified, as well as what shell should be used (bash). The home folder was created and minionuser was added to the sudo group.

The second section makes an SSH directory. Ownership of the directory is given to minionuser and the appropriate permissions are set (mode: 700 = read, write ,and execute). 

The final section copies the SSH key from the master to the minion. We make a file in the /home/minionuser/.ssh/ directory called authorized\_keys that comes from the source of salt://keys/minion\_key on the master. Again, ownership is given to minionuser and permissions are set to 600 for read and write (note that proper file and directory permissions must be set in order for SSH to function correctly).

After saving the file, execute the following command to apply the new state:

`sudo salt 'www1' state.highstate`

Connect into your minion to confirm that it uses SSH keys to login!

##What I did for pillar:

/srv/pillar/top.sls
```
base:
  'www1':
    - users
```
/srv/pillar/users.sls
```
users:
  jdonas:
    fullname: My Name
    groups:
      - sudo
    pub_ssh_keys:
      - ssh key
```
/srv/salt/users.sls
```
{% for username, details in pillar.get('users', {}).items() %}
{{ username }}:
  user.present:
    - fullname: {{ details.get('fullname','') }}
    - name: {{ username }}
    - shell: /bin/bash
    - home: /home/{{ username }}
    {% if 'groups' in details %}
    - groups:
      {% for group in details.get('groups', []) %}
      - {{ group }}
      {% endfor %}
    {% endif %}

  {% if 'pub_ssh_keys' in details %}
  ssh_auth.present:
    - user: {{ username }}
    - names:
    {% for pub_ssh_key in details.get('pub_ssh_keys', []) %}
      - {{ pub_ssh_key }}
    {% endfor %}
    - require:
      - user: {{ username }}
  {% endif %}

{% endfor %}
```
Then ran:
```
salt '*' saltutil.refresh_pillar
salt '*' pillar.items
salt ‘*’ state.highstate
```

Resources:

https://www.digitalocean.com/community/tutorials/how-to-use-the-digitalocean-api-v2
https://developers.digitalocean.com/documentation/v2
https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04
https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-freebsd-server
https://www.digitalocean.com/community/tutorials/how-to-install-salt-on-ubuntu-12-04
https://www.digitalocean.com/community/tutorials/automated-provisioning-of-digitalocean-cloud-servers-with-salt-cloud-on-ubuntu-12-04
https://www.digitalocean.com/community/tutorials/how-to-create-your-first-salt-formula
http://bencane.com/2013/09/03/getting-started-with-saltstack-by-example-automatically-installing-nginx/
http://ingloriousdevops.com/2014/10/14/do-you-want-more-flavor-add-more-salt/
http://docs.saltstack.com/en/latest/ref/states/all/salt.states.file.html
http://docs.saltstack.com/en/latest/ref/states/all/salt.states.user.html#management-of-user-accounts
