Title: Pillar Tutorial
Date: 2015-07-14 10:14
Category: Tutorial
Id: 010105

#Salt Pillars Tutorial

*This tutorial is an alternative, more advanced method of the additional configuration at the end of Part 3.*

Salt pillars are tree-like structures of data that can be used to securely store and distribute minion data (such as passwords and user accounts). Pillars are secure because they distribute data in a targeted fashion and only give data to the relevant minion. This brief tutorial covers the creation and distribution of a user account and SSH keys to a minion.

The pillar file system is similar to salt states. Within */srv/pillar* is a top.sls file as well as any other sls files that you need to create for various configurations. For our purposes, we are going to create a user.sls file to hold the login information that we want to push to minion1.

Vim into */srv/pillar/users.sls* and add the following lines:
```
users:
  minionuser:
    fullname: Yellow Minion
    groups:
      - sudo
    pub_ssh_keys:
      - ssh key
```
This configuration adds a user called minionuser. Minionuser is assigned a full name of Yellow Minion and is added to the sudo group. Adding minionuser to the sudo group allows the user to execute sudo commands (which allow for root configuration commands to be made). The full public ssh key of the minion is also added to the file.

Similar to salt states, pillars require a top.sls file that specifies which minions should be pushed pillar data. Make this file in /srv/pillar/top.sls:
```
base:
  'minion1':
    - users
```
This applies the user.sls pillar to minion1.

In order to use our pillar file, we need to create a state that applies it. This state file will be called users.sls and will be located in /srv/salt/. If you completed the previous tutorial that configured the users.sls file, you will need replace that file with this new one. If you are starting fresh, then simply create the new file:

`sudo vim /srv/salt/users.sls`

Add the following to the file:
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
Woah! Suddenly things just got a lot more complex! This file uses Jinja which is a Python templating language. Although it looks scary, the contents of this file accomplish a simple task. The Jinja code loops over the users.sls pillar file and actually creates the user on the minion. For example, if we had specified five different users in our users.sls pillar file, then the Jinja code would have looped over five times and created five different user accounts.
> You can read more about Jinja [here](http://jinja.pocoo.org/)

All that is left is to apply our new pillar information. Run the following commands:
```
sudo salt '*' saltutil.refresh_pillar
sudo salt '*' pillar.items
sudo salt ‘*’ state.highstate
```
The first command tells the minions to fetch their pillar data from the master. The second command applies the pillar (not sure if this is accurate). Finally, the highstate is called to actually create the user account on the minion.

Verify that your user account was created successfully by SSHing into the minion. That’s it for this tutorial!
