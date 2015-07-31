Title: Salt tutorial part 2
Date: 2015-07-08 11:45
Category: Tutorial
Id: 010102

# Salt-Cloud (Part 2 of 3)

Salt is a very useful set of software that allows commands to be executed across multiple servers simultaneously. This central administration allows many remote systems to be controlled with ease. This section will cover the installation of a Salt master and minion.

## Install Salt Master and Minion

If you're not already connected to your droplet, SSH in with:

    ssh saltuser@droplet-ip     

First we need to update the package database, to get an up-to-date list of software that is available. For any command that requires root access, we need to prepend ‘sudo’ to the command:

    sudo apt-get update

Add saltstack PPA (Personal Package Archive) and python dependencies. This is where we will install Salt from:

    sudo apt-get install python-software-properties
    sudo add-apt-repository ppa:saltstack/salt

Hit enter when prompted

Next, re-update the package database to get packages from the PPA we just added:

    sudo apt-get update

Salt operates with a master-minion structure. The master is the central administration hub that manages any number of minions. We eventually want to be able to do all the configuration on the master and push those configurations to the minions. Enter the following to install Salt master and minion:

    sudo apt-get install salt-master salt-minion

We are installing both the master and the minion on one device so that we can manage the droplet from itself! This may seem trivial, but it means that we can make the master manageable. For example, we may have multiple minions and want to push a new configuration to them as well as the master. Installing the minion software on the master allows for this. 

## Configure the Minion

Next, we need modify the salt minion config file. We will do so by using vim, a command-line text editor:

    sudo vim /etc/salt/minion

Using the arrow keys, navigate to the line that begins with `#master:`. Press `i` to edit. Uncomment the line by removing the `#`. Next, the option to `localhost` so that the line resembles the following:

    master: localhost

This tells the minion software to find the master on the local machine. Save and exit by pressing `ESC` followed by entering `:wq` and then pressing return.

To apply our changes, we need to restart the salt minion:

    sudo service salt-minion restart

## Connect Minion to Master

When the minion is created, it tries to connect to the master with a unique key. You can list all the minion keys that the master knows about:

    salt-key -L

Your droplet's name should show up under unaccepted keys similar to the following:

    Accepted Keys:
    Denied Keys:
    Unaccepted Keys:
    droplet-name
    Rejected Keys:

Now we need to have the master accept the minion's public keys:

    salt-key -a 'droplet-name'

Now that we have accepted the minion, we can check that it responds. The `*` means "all minions" and can be substituted for any specific minion name:

    salt '*' test.ping

## Connect Master to DigitalOcean

Now we need to configure SSH key authentication between DigitalOcean and our Salt master. This allows the master to connect to DigitalOcean in order to spin up new minion droplets.

Make a directory on the master to store the keys in:

    sudo mkdir /etc/salt/keys

Next, make an ssh keypair with:

    sudo ssh-keygen -f /etc/salt/keys/DigitalOcean-Key

You will be prompted through a process similar to the SSH keys we configured earlier. For our purposes, do not specify an optional passphrase. 

Now we need to add our new public ssh key to the DigitalOcean API. We will use curl to do this:

    curl -X POST -H 'Content-Type: application/json' -H 'Authorization: Bearer YourToken' -d '{"name":"DigitalOcean-Key.pub","public_key":"SaltPublicKey"}' "https://api.digitalocean.com/v2/account/keys"

You will need to put your full public key in place of SaltPublicKey. You can obtain it by copying your public key to clipboard with:

    cat /etc/salt/keys/DigitalOcean-Key.pub

## Install Salt-Cloud

Salt-Cloud is a sub-unit of the Salt package that controls the provisioning of minions. Install salt-cloud with:

    sudo apt-get install salt-cloud

Check that salt-cloud was installed:

    salt-cloud --version

We now need to edit the cloud configuration file so that future minions will be pointed to the correct master. The cloud configuration file is given to the minion when it is created and tells it what master to connect to.

    sudo vim /etc/salt/cloud

Add the following to the bottom of the file and change the red to the public domain name or ip address of your droplet

    provider: do
    # Set the location of the Salt master
    minion:
      master: domain or ip address

## Configure Cloud Providers and Profiles

The cloud provider configuration file specifies what provider to connect to and controls access to your DigitalOcean account:

    sudo vim /etc/salt/cloud.providers.d/digitalocean.conf

Add the following to the digitalocean.conf file:

    do:
      provider: digital_ocean
      # DigitalOcean account keys
      personal_access_token: YourToken
      ssh_key_name: DigitalOcean-Key.pub
      # Directory & filename on your Salt master
      ssh_key_file: /etc/salt/keys/DigitalOcean-Key

After editing the cloud providers file, you gain access to these commands:

    salt-cloud --list-images do
    salt-cloud --list-sizes do
    salt-cloud --list-locations do
    
Try these out to verify that the cloud providers file was configured accurately.
For more information on these commands, and others like them, you can use:

    salt-cloud --help

Edit the cloud profiles file. The profiles file holds "templates" of droplets that you may need to spin up:

    sudo vim /etc/salt/cloud.profiles.d/digital_ocean.conf

Add the following. Later on, you can add as many different profiles as you want. Simply replace the GREEN sections with valid options (these options can be found with the above `salt-cloud --list` commands):

    ubuntu_512MB_ny3:
        provider: do
        image: 14.04 x64
        size: 512MB
    #  script: Optional Deploy Script Argument
        location: New York 3
        private_networking: True
        
    # Create additional profiles, if you wish
    #[profile_alias_of_your_choosing]:
    #    provider: do
    #    image: [from salt-cloud --list-images do]
    #    size: [from salt-cloud --list-sizes do]
    #    script: [optional deployment script e.g. Ubuntu, Fedora, python-bootstrap, etc.]
    #    location: [from salt-cloud --list-locations do]
    #    private_networking: [True or False]

## Deploy Our First Minion

Now we're ready to make our first droplet! Simply execute the following command, and put in whatever name you would like for the minion:

    salt-cloud -p ubuntu_512MB_ny3 minion1

Note: If you are getting permission denied, make sure you are using a unique name for your ssh key. Open related issue: https://github.com/saltstack/salt/issues/25079

You can easily delete this minion by applying (we will keep it for now!):

    salt-cloud -d minion1

We have successfully finished installing Salt-Cloud and deploying our first minion! Next, we will install NGINX through our Salt master without issuing a single command on the minion!
