**Welcome to a three part tutorial on DigitalOcean, Salt-Cloud, and NGINX. This tutorial covers the creation of a DigitalOcean server (called a ‘droplet’), the installation and configuration of Salt Cloud and Salt Master, and the deployment of an NGINX web server on a Salt Minion.**

Prerequisites for this tutorial:
- Linux machine
- DigitalOcean account
- That’s it!

#DigitalOcean API (Part 1 of 3)

We’ll begin by creating a DigitalOcean access token. This token will be used to authenticate with DigitalOcean’s API (Application Program Interface). The API is useful because it is more powerful than simply using the DigitalOcean web interface; it will allow us to make in depth configuration.

##Create Your Access Token

1. Log in to the DigitalOcean Control Panel
2. Click on API in the top menu bar
3. Click Generate New Token
4. Enter a name for your token
5. Select the scope for this token (read or read/write)
6. Click Generate Token
7. Record your personal access token. It will not be shown again, for security purposes

##Let’s Make Our First Droplet!

Curl is a linux command line tool used to transfer data from or to a server. We will use it to access DigitalOcean’s API, without actually opening their site in a browser.

For the remainder of this tutorial, **BOLD** values signifies values that will work as shown but can also be changed to suit your needs. *ITALICISED* values signifies values that will be unique to your configuration and must be changed.

Open a terminal session on your linux machine and execute the following command:

    curl -X POST "https://api.digitalocean.com/v2/droplets" \
        -d'{"name":"salt-master","region":"nyc3","size":"512mb","image":"ubuntu-14-04-x64"}'\
        -H "Authorization: Bearer YourToken" \
        -H "Content-Type: application/json"


This creates a basic droplet named `Salt-Master`, in the New-York 3 location, with 512MB of RAM (which comes with a 20GB disk space), running Ubuntu 14.04.

##Configure SSH Key Authentication

A temporary SSH login password will be emailed to you. However, for added security, we will configure ssh keys between our local machine and the droplet we just created. 

Servers can authenticate with a variety of methods. Although passwords suffice, they are generally created manually and lack complexity (because that makes them easier to remember!). This makes them much more susceptible to brute-force attacks. 

SSH (Secure Shell) key authentication is a secure alternative. It involves two keys: the public key and private key. The private key is kept on the local machine and should never be shared. It can optionally be encrypted with a passphrase which limits its use if an unauthorized user gains access to it. The public key can be shared freely without any negative repercussions. It is used to encrypt any messages that only the private key can decrypt. This public-private pairing is a system that the majority of the world’s communication uses!

In your terminal session on the local machine, generate an SSH key with: 'ssh-keygen'

You will see the following output:

    Generating public/private rsa key pair.
    Enter file in which to save the key (/home/username/.ssh/id_rsa):

You may save it to it’s default location or enter a different path. For our purposes, press return to save it in the default location. This will store the keys in the .ssh directory within your user’s home directory. The public key will be called id_rsa.pub and the private key will be called id_rsa.

If you previously generated SSH keys, you will see the following prompt:

    /home/username/.ssh/id_rsa already exists.
    Overwrite (y/n)?

If you decide to overwrite the existing key, you will no longer be able to use the old key to authenticate. Only overwrite if you know you do not need the old key to authenticate with any other of your servers.

Next, you will see a prompt for a passphrase:

    Created directory '/home/username/.ssh'.
    Enter passphrase (empty for no passphrase):
    Enter same passphrase again:

This is an optional passphrase that will be used to encrypt the private key. If you choose to set a passphrase, you be required to enter it whenever you use the private key for authentication. This can provide another layer of security if the private key is somehow compromised. For our purposes, it is perfectly acceptable to press return twice and not use a passphrase.

##Copy the SSH Key to the Droplet

Now that we have generated an SSH key, we need to “link” the key between our local machine and the DigitalOcean droplet we created earlier. 

First, we need to SSH into our droplet and change the default password: 'ssh *root@droplet-ip*'

Because this is the first time connecting to the droplet, you will see a message such as the following:

    The authenticity of host '111.222.11.222 (111.222.11.222)' can't be established.
    ECDSA key fingerprint is fd:fd:d4:f9:77:fe:73:84:e1:55:00:ab:e6:6d:12:fe.
    Are you sure you want to continue connecting (yes/no)?

This simply means that linux machine does not recognize the host it is trying to connect to (because we have never connected to it before). Enter 'yes' to continue.

Once connected, you will be prompted for the current passphrase twice. This is a temporary passphrase that DigitalOcean will have emailed to you. After entering the current passphrase, you will need to specify a new passphrase and then confirm it again.

After finishing, log out of the droplet by holding down CTRL followed by D.

Now we need to link the SSH keys to the droplet. The command ‘ssh-copy-id’ will run a script on our droplet that will accomplish this.

In a terminal session on the local machine:

    ssh-copy-id root@droplet-ip
    
You will be prompted for the droplet’s passphrase (the new one that we set up when we first connected to it). This is the last time we will need to use it! Again, log out with CTRL-D.

Now you should be able to SSH into your droplet without having to enter a passphrase:

    ssh root@droplet-ip

That’s it! We have now setup our first droplet via the DigitaOcean API and configured SSH key authentication.
