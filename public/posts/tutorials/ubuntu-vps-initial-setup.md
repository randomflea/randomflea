---
title: "Initial Setup of an Ubuntu Linux VPS"
date: 2022-10-15T12:21:46-04:00
draft: false
---

# Initial Setup of an Ubuntu VPS

So you just got a VPS with Ubuntu installed on it and you have questions about it. How do you keep this secure on the Internet? How do you connect to it? Here are a few simple steps to keep your VPS secure.

## Prerequisites

1. Ubuntu Installed
2. Root access to server
3. Terminal access (we'll be connecting with SSH)

## Initial Connection

Most VPS providers have SSH access setup by default. There's different steps to connect depending on your operating system. For Linux and MacOS, open a terminal and use the ssh command:
```commandline
    ssh root@server-ip
```
For Windows, you need to download and install [Putty](https://the.earth.li/~sgtatham/putty/latest/w64/putty-64bit-0.77-installer.msi). Open Putty and enter the server's IP address in the Host Name box. Click Open then Yes on any popup that comes up. The popup just says you haven't connected to the machine before and wants to verify that you want to connect. Enter root for the username and the password your VPS provider gave you.

## Passwords

First you want to change the root password. Use the passwd command for that. You'll need to enter the password in twice and won't look like your typing.
```commandline
    passwd
```
The root account is a default account, so it poses a security risk. To mitigate this we'll create another user account with admin privileges. I'll call this account user for this guide, but that's not a great one to use in production. Run these commands to create the account, set the passwd, and grant admin privileges. The sudo group grants admin rights to a user on Ubuntu.
```commandline
    useradd -d /home/user -G sudo -m -s /bin/bash -U user
    passwd user
    usermod -aG sudo user
```
Now we want to test the new account to make sure it has admin rights. We'll run updates to test this while updating the system as well. If no errors come up, then we're good.
```commandline
    apt update
    apt install sudo -y
    su user
    sudo apt upgrade -y
    exit
```
We'll start a new SSH session with the user account. We'll use the same process as the Initial Connection section, but use the user account instead of root. Once connected, we can close the other session. Now let's disable the root account by editing the /etc/passwd file. We'll open the file with nano then edit the first line that starts with root. The last part of the line, /bin/bash, needs to change to /sbin/nologin. We'll start using ```sudo``` here to say we want the command run as an admin. When finished, use ctrl+x to save the file and exit.
```commandline
    sudo nano /etc/passwd
    Change root:......:/bin/bash to root:.........:/sbin/nologin
```

## SSH

First we want to change the port SSH listens to. This will prevent automated tools from finding and attacking the default port 22. We'll edit /etc/ssh/sshd_config to change the port and disable root access. Then we'll restart the SSH service to implement the changes. You should use a different port.
```commandline
    sudo nano /etc/ssh/sshd_config
        Change #Port 22 to Port 4321
        Change PermitRootLogin yes to PermitRootLogin no
    
    sudo systemctl restart sshd
```

Let's open a new seesion using the new port. For Windows, put the new port number in the Port box of Putty. For Linux and MacOS, use the -p switch. Once we verified that it's working, we can close the old session.
```commandline
    ssh user@server-ip -p 4321
```
There's a more secure way to authenticate with SSH than just using a password. We can use a key and passphrase. First we need to do some setup on the server by creating a directory and file. Then assigned the correct permissions.
```commandline
    mkdir ~/.ssh
    touch ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys
```
Next we'll need to create the keys, then copy the public key to the server. For Windows, open Puttygen which comes with Putty. At the bottom, make sure RSA is clicked and set Nubmer of bits to 2048 or 4096. Click Generate and move the mouse in the window to generate randomness. Once it's done, save the private key. This is the key you'll use to authenticate so save it in a safe place. Copy the Public key for pasting int OpenSSH authorized_keys file: from ssh-rsa to ==. On the server, open ~/.ssh/authorized_keys, nano ~/.ssh/authorized_keys, and paste the public key into it. For Linux and MacOS, use the ssh-keygen and ssh-copy-id commands.
```commandline
    ssh-keygen -t rsa -b 4096
    ssh-copy-id -p 4321 user@server-ip
```
For Windows, open Putty and enter the IP and port. On the left side under Connection -> SSH -> Auth, enter the path to the private key in the Private key file for authentication box. Then go back to Session on the left. Type in a name under Saved Sessions and click Save. Now you can just double click on the name to connect. For Linux and MacOS, use the -i switch.
```commandline
    ssh user@server-ip -p 4321 -i ~/.ssh/id_rsa
```
When you connect using the private key, you'll enter your username and the passphrase for the private key. Once we've verified it is working, we can close the other session. We can disable password authentication in the /etc/ssh/sshd_config file.
```commandline
    sudo nano /etc/ssh/sshd_config
        change #PubkeyAuthentication yes to PubkeyAuthentication yes
        change PasswordAuthentication yes to PasswordAuthentication no
    sudo systemctl restart sshd
```

## fail2ban

The last step is to setup fail2ban. It will block IP addresses after so many failed login attempts. There are 3 commands to install it.
```commandline
    sudo apt install fail2ban -y
```
Now to configure fail2ban, we'll copy the jail.conf file to jail.local since jail.conf may be overwritten during updates. We'll use the jail.local file to set it up. There's 3 options for the basic setup: ignoreip, bantime, and maxretry. If there's a # in front of it, we need to delete the # to enable it. Fail2ban will ingore any IP in the ignoreip list. IP addresses will be blocked for the specified time in bantime, default is 10 minutes. A host will get blocked after the specified failures in maxretry.
```commandline
    sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    sudo nano /etc/fail2ban/jail.local
        ignoreip = ip-address ip-address2
        bantime = 10m
        maxretry = 5
        
    sudo systemctl start fail2ban
    sudo systemctl enable fail2ban
```

## Wrapping Up

You should now have a secure Ubuntu instance. I like to reboot it and test everything out again to make sure it all still works like it should. Next I plan on writing a tutorial on how I made my site. Thanks for reading.