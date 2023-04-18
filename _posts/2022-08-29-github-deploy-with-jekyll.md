---
layout: post
title: Github deploy with Jekyll
date: '2022-08-29T16:14:00.000+01:00'
author: Luke Briner
tags: 
  - github
  - deploy
  - webhook
  - nginx
  - jekyll
---

## Deploying a Jekyll website to an external server from Github

It's funny sometimes when something that sounds like it should be quite simple ends up being complicated, not just because of the number of parts involved but also because the various guides describe doing the same thing in lots of different ways.

This is what I wanted:

+ Github repository with a jekyll site in it
+ When a PR is merged into `main` it should trigger a deploy
+ The site is on an external Ubuntu server
+ Since you don't commit jekyll output, we would need to run jekyll build on the web server
+ The webhook should be private and TLS secured so that the auth secret is not exposed

## Webhook on your server
In order to call a webhook on your server, you need something to serve that webhook, something with the ability to call a script to deploy the site/build the jekyll output. Fortunately, there is [webhook](https://github.com/adnanh/webhook) designed exactly for this and written in Go.

Start by installing the latest (currently 1.19) version of Go

    sudo apt update && sudo apt upgrade -y
    wget https://dl.google.com/go/go1.19.linux-amd64.tar.gz 
    sudo tar -C /usr/local -xzf go1.19.linux-amd64.tar.gz
    export PATH=$PATH:/usr/local/go/bin

Next, install the code for webhook. Note, you should try and set this up to run as a normal user (`ubuntu` for me) and not root.

    go install github.com/adanh/webhook@latest

If you run this in your home directory, it will install it into `~/go/bin/webhook`

### Configure the webhook server
We will create a directory to keep all of our config in, although there is currently only one webhook.

    mkdir ~/webhooks
    mkdir ~/webhooks/postfix

Note that my webhook was called postfix because it was for a site about postfix.

    nano ~/webhooks/hooks.json

Add the following:

    [{
        "id": "postfix",
        "execute-command": "/home/ubuntu/webhooks/postfix/deploy.sh",
        "command-working-directory": "/home/ubuntu/sites/postfix/",
        "response-message": "Executing deploy script...",
        "trigger-rule": {
            "match": {
                "type": "payload-hash-sha1",
                "secret": "random generated secret",
                "parameter": {
                    "source": "header",
                    "name": "X-Hub-Signature"
                }
            }
        }
    }]

Note that the secret could come from anywhere like a password manager or some CLI incantation of `/dev/random` but should be something reasonably secure. Also note that the working directory is the directory on my machine where the website will be built and served from (from the `_site` sub directory).

If you have more than one webhook, then add another entry to the array in `hooks.json`. Note that the paths should be absolute.

### Configure the actual website update script

    nano ~/webhooks/postfix/deploy.sh

My content looks like this:

    #!/bin/bash
    git fetch
    git checkout --force main
    git pull
    jekyll build

Then run

    chmod +x ~/webhooks/postfix/deploy.sh

It won't work yet, but will run under my site to pull from git and run the jekyll build.

## Setup the site directory
You might have already done this but if you go into your site directory (mine is /home/ubuntu/sites/postfix), we need to run:

    git init
    git remote add origin git@github.com:githubusername/githubreponame.git

If your repo is public, then you should be good to go, otherwise you will need to setup public key auth to permit access to your private repo.

### Create Access Key for Repo Access
If you are using a private github repo, first check if you have a public key on your server `cat ~/.ssh/id_rsa.pub`. If so, you need to copy the contents, otherwise run `ssh-keygen` and accept all the defaults to generate your default public/private keys and then cat the key again.

Store the contents in a notepad for later when we configure github.

## Install Jekyll on your server
Again, might be already done, but just remember that the instructions for Jekyll work in an interactive shell (by setting the path) so be aware that if you have already installed Jekyll to a different user than the one you will be using for the site deployment, you will probably come up against some issues.

The pre-requisites for [Jekyll](https://jekyllrb.com/docs/installation/ubuntu/) are the same regardless but when you then run the commands to update the path and then `gem install jekyll bundler` you should run these as the deployment user since they are based on the home directory of the current user.

## Configure the Webhook Service
Since you probably want to leave this running after reboots etc, you should create a service that will start the webhooks server in the background.

    sudo nano /etc/systemd/system/webhook.service

And add:

    [Unit]
    Description=Webhooks
    After=network.target

    [Service]
    Environment=PATH=/home/ubuntu/gems/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
    Environment=GEM_HOME=/home/ubuntu/gems
    ExecStart=/home/ubuntu/go/bin/webhook -hooks /home/ubuntu/webhooks/hooks.json -ip "127.0.0.1" -verbose
    User=ubuntu
    Group=ubuntu

    [Install]
    WantedBy=multi-user.target

 Now some important pointers:
 + We need the PATH and GEM_HOME to be set explicitly since we will run the service as ubuntu (in my case) and need the path to match what was added to `.bashrc`
 + We start the webhook server from where I installed it under /home/ubuntu, we specify the hooks file and bind it only to localhost since setting it up to use TLS is not straight-forward (but much easier with nginx, later)
 + We use the -verbose flag to pass more information to journalctl, which makes debugging a whole lot easier!
 + We need to specify to run as ubuntu, if not, it will run as root and all the stuff that works when you run it as ubuntu won't work here!

 Then we need to enable, start and check the service:

     sudo systemctl enable webhook.service
     sudo systemctl start webhook.service
     sudo systemctl status webhook.service

I usually run the status check after a few seconds to make sure it stays running and doesn't quit after, say, 5 seconds.

## Configure nginx to provide TLS functionality
Why not simply allow webhook to run publically with its -secure functionality? If we do that, we need to run as root to allow us to access the certs that are used for TLS. If we run as root, all of those problems related to installation locations and paths become harder and you should not run anything as root if you don't strictly need to.

Instead, it is much easier to get nginx to terminate TLS and simply upstream the request to the webhook server running on localhost. I won't detail how to modify and update nginx, which you hopefully already know but the config I used is below.

    sudo nano /etc/nginx/sites-available/webhooks

Content

    server {
        listen 9009 ssl;
        server_name example.ovh;

        ssl_certificate      /etc/letsencrypt/live/example.ovh/fullchain.pem;
        ssl_certificate_key  /etc/letsencrypt/live/example.ovh/privkey.pem;

        location / {
          try_files $uri @proxy;
        }

        location @proxy {
          proxy_pass http://webhooks;
        }
    }

    upstream webhooks {
        server 127.0.0.1:9000;
    }

The upstream section at the bottom is simply pointing to webhook on its default port (this port is not open to the internet on the firewall). The port 9009 at the top is just something else to use for the external connection to nginx. I used a non-default port simply to re-use a certificate that was already attached to a website. You could setup a unique DNS name and use Lets Encrypt in console mode if you wanted to do that instead.

nginx has access to the certs I am already using on other sites so this config is pretty easy to setup and use.

Make sure you open whatever port you are using in nginx e.g. 9009 on the firewall since github will call this.

## Test the webhook is running
You can use a browser to access your webhook endpoint e.g. `https://example.ovh:9009/` and you should get back "ok" and should see no certificate errors.

If you do get errors, it should be fairly straight-forward to trouble-shoot.

By this point, you should be able to prove that the webhook server is running, hopefully that you can reboot your server and the service comes back up OK and that the TLS and port are setup OK to accept incoming connections. Since we have configured the postfix webhook to take a secret, it will be easier to test the actual site deploy directly from github later!

## Setup Github
### Deploy Key
In the settings tab for the repo you will be pulling from, click on Deploy keys and "Add deploy key". Simply paste in the *public* key from your server and give it a suitable name. It only needs permissions to pull so don't give it write permissions.

### Webhook
The webhook is simply a url that github will call after certain actions take place. You can invoke after all kinds of things but we are only interested in "push" which means any time code is pushed or merged into all branches. Although this is a bit much for what we need, since our script will pull the `main` branch, it won't matter if it gets called after code is pushed to another branch.

It is quite easy to understand:
+ Payload URL. don't forget the port number and the fact that your DNS name needs to be public got github to see it
+ Content type: application/json
+ Secret: as you entered into `hooks.json`
+ Enable SSL verification. If you don't do this then you have done something wrong!
+ Just the push event

The first time you create it, it will call the webhook to ensure it works. I don't *think* it calls it if you update it later.

## Test the webhook works
Depending on whether you had already checked out your code from git, you might be able to test that it worked by looking for the git files now being present in your deploy directory (e.g. ~/sites/postfix) and also that the `_site` folder has been created by jekyll.

If this is NOT the case, then you can use journalctl to see events related to the webhook service and the -verbose flag will show you what you need to know:

    journalctl -u webhook.service -n 20

The service name needs to match what you called it (if different than my example) and `-n 20` is simply to show the last 20 entries.

If things didn't work, you will either see nothing in the log except perhaps:

    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17 version 2.8.0 starting
    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17 setting up os signal watcher
    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17 attempting to load hooks from /home/ubuntu/webhooks/hooks.json
    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17 found 1 hook(s) in file
    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17         loaded: postfix
    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17 serving hooks on http://127.0.0.1:9000/hooks/{id}
    Aug 29 12:12:17 vps-abc123ed webhook[1340]: [webhook] 2022/08/29 12:12:17 os signal watcher ready

If you only see that, the webhook is not being called and github will silently fail if it doesn't work for any reason. If this happens, check:

+ nginx restarted with the new config
+ port is correct and open on any firewalls
+ your tls certs are valid
+ your dns name is public so github can see it
+ the port mentioned by journalctl matches what you specified in your nginx conf

If you see errors in the log, then you will need to resolve these. You might see things like `git: unknown command` or `jekyll: could not find....` which might mean you have some user config stuff wrong or the path in your service is missing or incorrect.

If you attempt to run the lines from the deployment script in your deployment folder and it works, then clearly it is an issue with users. If that doesn't work then you might need to fix git, check your remote is correct etc.

## Serve the site with nginx
You might have already done this from before but obviously once everything else is working, you can serve the contents of the `_site` folder in your git repo and you should all be good!
    