---
title: "Create Hugo Site with Cactus theme and isso Comments on Rocky Linux Served by nginx"
date: 2022-07-12T12:21:45-04:00
draft: false
---

I had been using flask to run my personal site, then decided to switch to [Hugo](https://gohugo.io/) for simplicity. I looked through the [Hugo Themes](https://themes.gohugo.io/) and really liked the [Cactus Theme](https://themes.gohugo.io/themes/hugo-theme-cactus/). It's minimal and just looks better in my opinion. It had places for my blog, projects, and info about me which is all I needed for now. I wanted to add a comment section to my blog posts. Disqus is the big one but I don't like the tracking they do. I searched for an alternative and found [Isso](https://isso-comments.de/). It's a Python package that uses SQLite so I liked that. Now let's talk about how to set all of this up.

## Prerequisites
1. Rocky Linux is installed and running (CentOS, RHEL, or AlmaLinux will work too)
2. A user with admin rights (this username is user)
3. You have a domain name (mysite.com is used here)
4. DNS has an A record pointing to the server IP

### Note
I recommend going through my post on how to [setup your VPS instance to make it more secure](https://randomflea.dev/posts/tutorials/rocky-linux-vps-initial-setup/). 

## Install Dependencies
We need to install all the dependencies for Hugo, Isso, and nginx. We need the Extra Packages for Enterprise Linux, then run an upgrade. Next, we can install the dependencies.
```commandline
sudo dnf install epel-release -y
sudo dnf upgrade -y
sudo dnf install python3 python3-devel sqlite gcc git nginx snapd -y
```


## Setup Hugo
Hugo is installed by [Homebrew](https://brew.sh/) which is not installed by default on Rocky Linux. To install Homebrew, we use the command:
```commandline
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
Press Enter to continue the install and wait a few minutes. Once it's finished, it will give you two commands to run:
```
echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> /home/user/.bash_profile
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
```
Now we can run ```brew install hugo``` to install Hugo. With Hugo installed, we can create our site by using this command:
```
hugo new site mysite
```
You should replace mysite with the name of your site. This will create a directory called mysite in the current directory. Next we need to download the Cactus theme and add it to the site config with these commands:
```commandline
cd mysite
git init --initial-branch=main
git submodule add https://github.com/monkeyWzr/hugo-theme-cactus themes/cactus
```
Let's edit the config file for mysite. I use nano to edit files from the command line.

```nano ~/mysite/config.toml```

This is an example config. Replace http://mysite.com with your domain. You can change the title, copyright, and anything else you need to.
```commandline
baseURL = 'http://mysite.com/'
languageCode = 'en-us'
title = 'My Site'
theme = "cactus"
copyright = "You or your company"

[params]
    colortheme = "dark"
    description = "This is my site. Hope you enjoy"
    showAllPostsOnHomePage = false
    postsOnHomePage = 5
    showProjectsList = true
    projectsUrl = "https://github.com/"

    [[params.social]]
        name = "github"
        link = "https://github.com/"
    [[params.social]]
        name = "linkedin"
        link = "https://www.linkedin.com"
    [[params.social]]
        name = "email"
        link = "mailto:mysite@gmail.com"

    [[menu.main]]
        name = "Home"
        url = "/"
        weight = 1

    [[menu.main]]
        name = "All posts"
        url = "/posts"
        weight = 2

    [[menu.main]]
        name = "Tags"
        url = "/tags"
        weight = 3

    [[menu.main]]
        name = "About"
        url = "/about"
        weight = 4

```
Save and exit Nano with ctrl+x. Type y to confirm saving then press enter to save the file. Running ```hugo -D``` will create the files for mysite. It's time to test it out. We need to allow port 1313 through the firewall temporarily, then start the Hugo server. Replace server-ip with the actual IP.
```commandline
sudo firewall-cmd --add-port=1313/tcp --zone=public
hugo server --bind=server-ip --baseURL=server-ip
```
Open a browser and go to http://server-ip:1313. If your site comes up without errors, we can ctrl+c to stop the server. We need to remove the firewall rule to allow port 1313 by running ```sudo firewall-cmd --reload```. Now we need to copy the mysite directory to /var/www/mysite. This will be our production directory and /home/user/mysite will be our test directory where we make changes and test.
```commandline
sudo mkdir /var/www
sudo chown -R user:nginx /var/www
sudo chmod 0755 /var/www
cp -rf ~/mysite /var/www/
```

## Configure Nginx for mysite
Now we need to configure Nginx to serve mysite. We make our config file in /etc/nginx/conf.d/mysite.conf, you can name the .conf file whatever you want. I use nano to edit files.

```sudo nano /etc/nginx/conf.d/mysite.conf```
```commandline
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite/public/;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
Save and exit Nano with ctrl+x. Type y to confirm saving then press enter to save the file. We also need to disable the default nginx config. Let's open /etc/nginx/nginx.conf and comment out the server block under http by adding a # to start of each line:
```commandline
#    server {
#        listen       80 default_server;
#        listen       [::]:80 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }
```
Save and exit nano. Port 80 and 443 need to be allowed through the firewall. Next we need to test the nginx config and start the nginx service. We can check if nginx is running with the systemctl status command. If you see active (running), it's good. Press q to exit that. Then we can enable nginx to run on boot.
```commandline
sudo firewall-cmd --zone=public --add-service=http --permanent
sudo firewall-cmd --zone=public --add-service=https --permanent
sudo firewall-cmd --reload
sudo nginx -t
sudo systemctl start nginx
sudo systemctl status nginx
sudo systemctl enable nginx
```
At this point we can test if our site is working. Open a browser and go to mysite.com. If you can see My New Huo Site, it's working like it should.

## Setup Isso
To install [Isso](), we have to create a isso user and create a python virtual environment. ```sudo useradd isso``` will create the isso user. Then we need to block ssh access for the isso user. We'll edit /etc/ssh/sshd_config and add "DenyUsers isso" without the quotes to the end of it. Now let's restart sshd and switch to the isso user.
```
sudo systemctl restart sshd.service
sudo su isso
cd
```
Now we will create the python virtual environment and install Isso with these commands:
```commandline
mkdir db etc log
python3 -m venv isso
source isso/bin/activate
pip3 install --upgrade pip
pip3 install isso
pip3 install gevent
```
Create the isso config file using ```nano ~/etc/comments.cfg```. This is my config file. Replace http://mysite.com with your domain. Do not replace the salt. You should replace isso_admin_password on the last line with your own password. 
```commandline
[general]
; Batabase location, check permissions, automatically created if not exists
dbpath = /home/isso/db/comments.db
; Blog website url
host = http://mysite.com/comments
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
Next run ```isso -c ~/etc/comments.cfg run``` to test the config. As long as there are no error messages, it's good and you can ctrl+c to stop it. We'll need sudo to run the rest of the commands so type ```exit``` and press enter. That'll log off isso and go back to user. Now we'll create a systemd service file to run isso on boot. Create the file /usr/lib/systemd/system/isso.service with this config:

```nano /usr/lib/systemd/system/isso.service```
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

## Configure Nginx for Isso
Now we need to configure Nginx for Isso. We'll edit our config file in /etc/nginx/conf.d/mysite.conf. We need to add the location /comments block under the location / block.

```nano /etc/nginx/conf.d/mysite.conf```
```commandline
server {
    listen 80;
    server_name mysite.com;

    root /var/www/mysite/public/;
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
```
Next we'll test the nginx config again and restart the service if there are no errors:
```commandline
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```
Let's see if Isso working through nginx by browsing to mysite.com/comments/admin. You should get a login page. Enter the isso_admin_password from the config file and make sure you can login. If that's working, we can integrate Hugo and Isso.

## Integrate Isso with mysite
We need to tell Hugo to use Isso comments by editing the config.toml in the /home/user/mysite directory. At the end of config.toml, add these lines:
```commandline
[params.comments]
    enabled = true
    engine = "isso"
    
[params.isso]
    data = "/comments"
    css = true
    lang = "en"
    avatar = true
    avatarBg = "#f0f0f0"
```
Then we have to edit an HTML file to display the comments. In mysite/themes/cactus/layouts/partials/comments.html, below ```{{ partial "comments/cactus_comments.html" . }}``` add these lines:
```commandline
{{ else if eq .Site.Params.Comments.Engine "isso" }}
      {{ partial "comments/isso.html" }}
```
Let's create the mysite/themes/cactus/layouts/partials/comments/isso.html file. Add this to it:
```commandline
<div class="blog-post-comments">
  <script
    data-isso="{{ .Site.Params.isso.data }}"
    data-isso-id="{{ .Site.Params.isso.id}}"
    data-isso-css="{{ .Site.Params.isso.css }}"
    data-isso-lang="{{ .Site.Params.isso.lang }}"
    data-isso-avatar="{{ .Site.Params.isso.avatar }}"
    data-isso-avatar-bg="{{ .Site.Params.isso.avatarBg }}"
    src="/comments/js/embed.min.js">
  </script>
  <noscript>Please enable JavaScript to view the comments powered by <a href="https://posativ.org/isso/">Isso</a>.</noscript>
  <div>
    <section id="isso-thread"></section>
  </div>
</div>
```
Let's make a test post and see if the comments show up. Run this command to create a new post: ```hugo new posts/first.md```. This will be stored in mysite/content/post/first.md. Open that file and add anything below the last ---. Also change draft: true to draft: false. Now we can run ```hugo -D``` again to create the site files. Then copy them to the production directory.
```commandline
cp -rf ~/mysite /var/www/
```
As long as the comment section shows up, it's all working like it should. All that's left is to enable SSL on the site.

## Setup SSL for mysite
We'll use Let's Encrypt to generate the certificates. We need to setup snap to install certbot.
```commandline
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap
sudo snap install core # may get error: too early for operation, give it a couple seconds
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```
In the Isso config file change the host variable to be https.

```nano /home/isso/etc/comments.cfg```
```
host = https://mysite.com
```
Cerbot will handle changing the mysite.conf nginx file to include the certificate. It will also redirect port 80 to 443 so everything works like you would expect. The very last step is to restart nginx and make sure everything still works.
```commandline
sudo systemctl restart nginx
```

## Closing Thoughts
You now have a Hugo website with Isso comments served by nginx. Hugo makes it easy to customise the site. You can play around with the different options for the config.toml file and see what you like best. Enjoy your new site and happy posting!