---
title: "Create Hugo Site with Cactus theme and isso Comments on Rocky Linux Served by nginx"
date: 2022-05-24T12:21:45-04:00
draft: true
---

I had been using flask to run my personal site, then decided to switch to [Hugo](https://gohugo.io/) for simplicity. I looked through the [Hugo Themes](https://themes.gohugo.io/) and really liked the [Cactus Theme](https://themes.gohugo.io/themes/hugo-theme-cactus/). It's minimal and just looks better in my opinion. It had places for my blog, projects, and info about me which is all I needed for now. Now let's talk about how to set all of this up.

## Prerequisites
1. Rocky Linux is installed and running (CentOS, RHEL, or AlmaLinux will work too)
2. A user with admin rights (this username is user)
3. You have a domain name (mysite.com is used here)

```commandline
    sudo dnf install epel-release -y
    sudo dnf upgrade -y
    sudo dnf install python3 python3-devel sqlite gcc git nginx snapd -y
    
    sudo firewall-cmd --zone=public --add-service=http --permanent
    sudo firewall-cmd --zone=public --add-service=https --permanent
    sudo firewall-cmd --reload
```


## Setup Hugo
Hugo is installed by [Homebrew]() which is not installed by default on Rocky Linux. To install Homebrew, we use the command:
```commandline
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
Press Enter to continue the install and wait a few minutes. Once it's finished, it will give you two commands to run:
```
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> /home/user/.bash_profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```
Now we can install Hugo with 
```
    brew install hugo
```
With Hugo installed, we can create our site by using the command: 
```
    hugo new site mysite
```
You should replace mysite with the name of your site. Next we need to download the Cactus theme and add it to the site config with these commands:
```commandline
    cd mysite
    git init --initial-branch=main
    git submodule add https://github.com/monkeyWzr/hugo-theme-cactus themes/cactus
    echo theme = \"cactus\" >> config.toml
```
This will create the files for mysite:
```commandline
    hugo -D
```
Test it out with this:
```commandline
    sudo firewall-cmd --add-port=1313/tcp --zone=public
    hugo server --bind=server-ip --baseURL=server-ip
    sudo firewall-cmd --reload
```

## Setup Isso
To install [Isso](), we have to install the dependencies, create a isso user, and create a python virtual environment. Using these commands, we can install the dependencies:
The following commands will create the isso user, deny isso user access to ssh, switch to the isso user, and change the directory to /home/isso: 
```commandline
    sudo useradd isso
    sudo echo "DenyUsers isso" >> /etc/ssh/sshd_config
    sudo systemctl restart sshd.service
    sudo su isso
    cd
```
Now we will create the python virtual environment and install Isso with these commands:
```commandline
    mkdir db etc log
    python3 -m venv isso
    source isso/bin/activate
    (isso) pip3 install --upgrade pip
    (isso) pip3 install isso
    (isso) pip3 install gevent
```
Create the isso config file using ```nano ~/etc/comments.cfg```. This is my config file:
```commandline
    [general]
    ; Batabase location, check permissions, automatically created if not exists
    dbpath = /home/isso/db/comments.db
    ; Blog website url
    host = https://mysite.com/comments
    ; Log file
    log-file = /home/isso/log/isso.log
    
    [moderation]
    ; Enable comment moderation
    enabled = true
    purge-after = 30d
    
    [server]
    ; Listen custom port 7845
    listen = http://localhost:7845
    
    [guard]
    ; Enable basic spam protection features
    ; REMEMBER to edit the client configuration accordingly
    enabled = true
    ; limit to 2 new comments per minute
    ratelimit = 2
    ; limit 25 comments directly to the thread
    direct-reply = 25
    ; allow commenters to reply to their own comments when they could 
    ; still edit the comment
    reply-to-self = true
    ; force commenters to enter a value into the author field
    require-author = true
    require-email = false
    
    [markup]
    options = strikethrough, superscript, autolink
    allowed-elements =
    allowed-attributes =
    
    [hash]
    salt = Eec77co7Ohloo3o9Ol6baimi
    algorithm = pbkdf2
    
    [rss]
    ; provide an Atom feed for each comment thread
    
    [admin]
    ; Provide a web administration interface
    enabled = true
    password = isso_admin_password
```
You should replace isso_admin_password on the last line with your own password. Next run ```(isso) isso -c ~/etc/comments.cfg run``` to test the config. As long as there are no error messages, it's good. With Nginx and Isso running we can go to https://mysite.com/comments/admin and log in with the isso_admin_password. Now we'll create a systemd service file to run isso on boot. Create the file /usr/lib/systemd/system/isso.service with this config:
```commandline
    [Unit]
    Description=Isso Comment Server
    
    [Service]
    Type=simple
    User=isso
    workingDirectory=/home/isso
    Environment="PATH=/home/isso/isso/bin"
    ExecStart=/home/isso/isso/bin/isso -c /home/isso/etc/comments.cfg
    Restart=on-failure
    RestartSec=5
    
    [Install]
    WantedBy=multi-user.target
```
Then run these command to start the isso service and enable it to run on boot:
```commandline
    sudo systemctl daemon-reload
    sudo systemctl start isso.service
    sudo systemctl enable isso.service
```

## Configure Nginx
Now we need to configure Nginx to serve mysite. We make our config file in /etc/nginx/conf.d/mysite.conf, you can name the .conf file whatever you want. Here's my config:
```commandline
    server {
        listen 443 ssl;
        server_name mysite.com;
    
        root /home/user/mysite/public/;
        index index.html;
    
        location / {
            try_files $uri $uri/ =404;
        }
    
        location /comments {
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Script-Name /comments;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_pass http://localhost:7845;
        }
    }
    
    server {
        listen 80;
        server_name mysite.com;
        return 301 https://$host$request_uri;
    }
```
Next we need to test the nginx config and restart the service if there are no errors:
```commandline
    sudo nginx -t
    sudo systemctl start nginx
    sudo systemctl status nginx
    sudo systemctl enable nginx
```
At this point we can test if our site is working at least

## Setup SSL Certificate for https
```commandline
    sudo systemctl enable --now snapd.socket
    sudo ln -s /var/lib/snapd/snap /snap
    sudo snap install core # may get error: too early for operation, give it a couple seconds
    sudo snap refresh core
    sudo snap install --classic certbot
    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    sudo certbot --nginx
```

## Integrate Isso with mysite
We need to tell Hugo to use Isso comments by editing the config.toml in the mysite directory. At the end of config.toml, add these lines:
```commandline
    [params.comments]
        enabled = true
        engine = isso
        
    [params.isso]
        data = "/comments"
        css = true
        lang = "en"
        avatar = true
        avatarBg = #f0f0f0
```
Then have to edit an HTML file to display the comments. In mysite/themes/cactus/layouts/partials/comments.html, above the last {{ end }} add these lines: