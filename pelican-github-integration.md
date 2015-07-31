Title: Pelican Github Integration
Date: 2015-07-14 12:31
Category: Tutorial
Id: 010104

#Pelican Blog and Github Integration

*This tutorial assumes that you have completed Part 1-3 of the Salt Tutorials.*

Welcome to the tutorial for the installation of the Pelican blog software as well as integration with Github.  Pelican is a static site generator that is lightweight and easy to set up. A static site requires little server resources and is therefore a viable choice to host a blog with. We will integrate Pelican, NGNIX, and github in order to automatically pull new markdown files from github and turn them into a simple blog.

We need to start off by making a git state file. This state will install git on the minion and pull files from github:

`sudo vim /srv/salt/git.sls:`

Add the following to the file:
```
git:
  pkg:
	- installed
https://github.com/lvl1/courses.git:
  git.latest:
	- target: /opt/courses
```
This pulls files from the repository URL of your choice and saves them to */opt/courses* on the minion.

Ensure that NGINX is installed on your minion. The state file for NGINX should look similar to the following:
```
nginx:
  pkg:
	- installed
  service:
	- running
	- require:
  	- pkg: nginx
```
Now that we have git and NGINX ready to go, we need to make a pelican state file in */srv/salt/pelican.sls*:
```
python-pelican:
  pkg:
	- installed
markdown:
  pkg:
	- installed
/opt/courses:
  pelican.build_site:
	- output: /usr/share/nginx/html
```
This installs Pelican and markdown. Markdown is a very simple way to add formatting to text. If the Github files we write use markdown, then we need to have markdown installed on the minion as well. The last part of this file takes the markdown (.md) files that we downloaded from github and gets pelican to convert them to html and store them in */usr/share/nginx/html*. This is the directory that NGINX serves its websites from. Don’t be confused if you don’t know where “*pelican.build_site*” is coming from - we are going to be building a custom state that uses the function “*build_site*” to run pelican for us.

Now let’s update the top.sls file (*/srv/salt/top.sls*) to reflect our new configuration:
```
base:
  'minion1':
	- nginx
	- git
	- pelican
```

/srv/salt/_states/pelican.py:

```python
import salt.exceptions
import subprocess

def build_site(name, output="/srv/www"):
        # Generates static site with pelican -o $output $name

        ret = {'name': name, 'changes': {}, 'result': False, 'comment': ''}

        #current_state = __salt__['pelican.current_state'](name)
        current_state = "cool"

        if __opts__['test'] == True:
                ret['comment'] = 'Markdown files from "{$0}" will be converted to HTML and put in "{1}"'.format(name,output)
                ret['changes'] = {
                'old': current_state,
                'new': 'New!',
                }
                ret['result'] = None

        subprocess.call(['pelican', '-o', output, name])

        #ret['comment'] = 'Static site generated from "{$0}"'.format(name)
        ret['comment'] = 'Static site generated.'

        ret['changes'] = {
        'old': current_state,
        'new': 'Whoopee!',
        }

        ret['result'] = True

        return ret
```
