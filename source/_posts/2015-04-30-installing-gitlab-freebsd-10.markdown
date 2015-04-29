---
layout: post
title: "Installing gitlab on FreeBSD 10"
date: 2015-04-30 00:32
comments: true
categories: git freebsd
---
{% img center https://forums.freebsd.org/styles/freebsd/xenforo/freebsd_logo.png %}
{% img center https://about.gitlab.com/images/wordmark_inverted_small.png %}
{% blockquote from https://blog.assemblyco.de/installing-gitlab-on-freebsd-10 %}
{% endblockquote %}

### 1. Update system

    pkg update  
    pkg upgrade  

### 2. Install dependencies

#### Install system packages:

    pkg install sudo bash icu cmake pkgconf git nginx ruby ruby20-gems logrotate redis postgresql94-server postfix krb5  

#### Install bundler gem system-wide:

    gem install bundler --no-ri --no-rdoc  

Add this to /etc/rc.conf:

    # Core services
    sshd_enable="YES"  
    ntpd_enable="YES"  
    ntpd_sync_on_start="YES"

    # GitLab services
    redis_enable="YES"  
    postgresql_enable="YES"  
    gitlab_enable="YES"

    # Web server
    nginx_enable="YES"

    # Postfix/Sendmail
    postfix_enable="YES"  
    sendmail_enable="NO"  
    sendmail_submit_enable="NO"  
    sendmail_outbound_enable="NO"  
    sendmail_msp_queue_enable="NO"  

### 3. Create git user for GitLab

#### Set up user and groups:

    # Create user
    pw add user -n git -m -s /usr/local/bin/bash -c "GitLab"

    # Add 'git' user to 'redis' group (this will come in useful later!)
    pw user mod git -G redis  

### 4. Set up Postgres database

As root, make sure that Postgres is running: service postgresql restart

Check this with service postgresql status

Set up the database:

    # Log in to Postgres user account
    su - pgsql

    # Initialise Postgres db
    initdb /usr/local/pgsql/data

    # Connect to Postgres database
    psql -d template1  

When logged into the database:

    # Create a user for GitLab
    # Do not type the 'template1=#', this is part of the prompt
    template1=# CREATE USER git CREATEDB;

    # Create the GitLab production database & grant all privileges on database
    template1=# CREATE DATABASE gitlabhq_production OWNER git;

    # Quit the database session
    template1=# \q  

Then type exit to drop back to the root user.
Try connecting to the new database with the git user:

    su - git  
    psql -d gitlabhq_production  

If this succeeds, quit the database session by typing \q or hitting CTRL-D.

### 5. Install and set up Redis

Back up the original Redis config file:

    cp /usr/local/etc/redis.conf /usr/local/etc/redis.conf.orig  

Run the following commands to get Redis running:

    # Disable Redis listening on TCP by setting 'port' to 0
    sed 's/^port .*/port 0/' /usr/local/etc/redis.conf.orig | sudo tee /usr/local/etc/redis.conf

    # Enable Redis socket
    echo 'unixsocket /usr/local/var/run/redis/redis.sock' | sudo tee -a /usr/local/etc/redis.conf

    # Grant permission to the socket to all members of the redis group
    echo 'unixsocketperm 770' | sudo tee -a /usr/local/etc/redis.conf

    # Create the directory which contains the socket
    mkdir -p /usr/local/var/run/redis  
    chown redis:redis /usr/local/var/run/redis  
    chmod 755 /usr/local/var/run/redis

    # Restart redis
    sudo service redis restart  

### 6. Install and set up GitLab

    # Change to git home directory
    cd /home/git

    # Clone GitLab source
    sudo -u git -H git clone https://gitlab.com/gitlab-org/gitlab-ce.git -b 7-6-stable gitlab

    # Go to GitLab source folder
    cd /home/git/gitlab

    # Copy the example GitLab config
    sudo -u git -H cp config/gitlab.yml.example config/gitlab.yml  

Edit the GitLab configuration file (sudo -u git -H vim config/gitlab.yml):

    The option host: should be set to your domain, e.g. "gitlab.mysite.com".
    The line bin_path: should be set to FreeBSD's git location: /usr/local/bin/git.

#### 6.1. Set up Gitab Config

As root:

    cd /home/git/gitlab  
    chown -R git log/  
    chown -R git tmp/  
    chmod -R u+rwX,go-w log/  
    chmod -R u+rwX tmp/

    # Change back to 'git' user
    su - git  
    cd /home/git/gitlab

    # Make folder for satellites and set the right permissions
    mkdir /home/git/gitlab-satellites  
    chmod u+rwx,g=rx,o-rwx /home/git/gitlab-satellites

    # Make sure GitLab can write to the tmp/pids/ and tmp/sockets/ directories
    chmod -R u+rwX tmp/pids/  
    chmod -R u+rwX tmp/sockets/

    # Make sure GitLab can write to the public/uploads/ directory
    chmod -R u+rwX  public/uploads

    # Copy the example Unicorn config
    cp config/unicorn.rb.example config/unicorn.rb

    # Set the number of workers to at least the number of cores
    vim config/unicorn.rb

    # Copy the example Rack attack config
    cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

    # Configure Git global settings for git user, useful when editing via web
    # Edit user.email according to what is set in gitlab.yml
    git config --global user.name "GitLab"  
    git config --global user.email "example@example.com"  
    git config --global core.autocrlf input

    # Copy Redis connection settings
    cp config/resque.yml.example config/resque.yml

    # Configure Redis to use the modified socket path
    # Change 'production' line to 'unix:/usr/local/var/run/redis/redis.sock'
    vim config/resque.yml

    # Copy database config
    cp config/database.yml.postgresql config/database.yml

    # Install Ruby Gems
    bundle install --deployment --without development test mysql aws  

### 7. GitLab Shell

    # Run the rake task for installing gitlab-shell
    bundle exec rake gitlab:shell:install[v2.4.0] REDIS_URL=unix:/usr/local/var/run/redis/redis.sock RAILS_ENV=production

    # Edit the gitlab-shell config
    # Change the 'socket' option to '/usr/local/var/run/redis/redis.sock'
    # Change the 'gitlab_url' option to 'http://localhost:8080/'
    # Don't bother configuring any SSL stuff in here because it's used internally
    vim /home/git/gitlab-shell/config.yml  

### 8. Initialise Database

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production  
    # Type 'yes' to create the database tables.
    # When done you see 'Administrator account created:'

Note: You can set the initial Administrator password by supplying it in the environment variable GITLAB_ROOT_PASSWORD as seen below. If you don't set the password (and it is set to the default one) please don't expose GitLab to the public internet until the installation is complete and you have changed the default password.

    sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production GITLAB_ROOT_PASSWORD=yourpassword  

### 9. Init script

As root:

    cp /home/git/gitlab/lib/support/init.d/gitlab /usr/local/etc/rc.d/gitlab  

### 10. Check Configuration and Compile Assets

    cd /home/git/gitlab  
    su - git  
    bundle exec rake gitlab:env:info RAILS_ENV=production  

If this all passes (all green and/or no errors are reported), then go ahead and compile all of the assets for GitLab. This can take ~10-15 minutes on a smaller machine, so don't panic if it takes a while!

    bundle exec rake assets:precompile RAILS_ENV=production  

### 11. Start GitLab service

If all of the above steps complete with no errors and everything has gone
smoothly, then start the GitLab service.

As root:

    service gitlab start  

### 12. Nginx

The officially supported web server in GitLab is nginx - and GitLab provide
an nginx configuration file in /home/git/gitlab/lib/support/nginx/gitlab,
so you can copy that if you prefer, and modify their template.

I didn't want to do that, so this is the configuration I used:

    server {  
        server_name yourserver.yourdomain;
        server_tokens off;

        listen 80 default accept_filter=httpready;

        # Uncomment if you want to use SSL
        # listen 443 ssl;

        # Configure your SSL certificate locations here
        # ssl_certificate        /usr/local/etc/nginx/ssl/gitlab/ssl-bundle.crt;
        # ssl_certificate_key  /usr/local/etc/nginx/ssl/gitlab/gitlab.key;

        # Uncomment to force SSL connections
        # if ($ssl_protocol = "") {
        #    rewrite ^   https://$server_name$request_uri? permanent;
        # }

        location / {
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   Host      $http_host;
            proxy_pass         http://127.0.0.1:8080;
        }
    }

Restart nginx with sudo service nginx restart, and you should be up and
running.

Check everything with this command just to be sure:

    su - git  
    cd /home/git/gitlab

    bundle exec rake gitlab:check RAILS_ENV=production  

Good to Go

If everything comes up green, then GitLab should work.

If some things show up as red, blue, pink or any colour that's not green - read any error messages thoroughly before trying any suggested fixes. Google comes in extremely handy when trying to diagnose unhelpful Ruby error messages.

### Troubleshooting

#### 504 - Gateway Timed Out errors

This can be caused by several different things with GitLab. The best bet is to go back up through the install guide and check each step has been properly executed.

    Check the logs! Look in /home/git/gitlab/log for clues.
    Check what's running! The command sockstat -4l usually gives an idea of which services are running on which ports. (Redis uses port 6379, Unicorn uses port 8080, and Postgres uses port 5432).

What it usually boils down to:

    GitLab's assets haven't been precompiled (there is a command above)
    Postgres isn't running or the database isn't set up properly
    Redis isn't running
    Nginx isn't set up properly

#### Gem timfel-krb5-auth fails to build

Install the Kerberos package: pkg install krb5. As far as I know, there's no way to disable the Kerberos authentication in GitLab (even if it's unused) so unfortunately the only solution is to install the missing packages.

[edit: newer versions of GitLab (7.8 and upwards) depend on the krb5-111 package instead of krb5 - so if you're installing a newer version of GitLab, run: pkg install krb5-111]
Postfix/sendmail: "postdrop: warning: unable to look up public/pickup: No such file or directory"

Sometimes Postfix and/or sendmail might complain if they're not set up
correctly or have only just been installed.

    mkfifo /var/spool/postfix/public/pickup  
    killall $(pgrep sendmail) # Kill all sendmail processes  
    sudo service postfix restart # Restart Postfix  

http://www.databasically.com/2009/12/02/ubuntu-postfix-error-postdrop-warning-unable-to-look-up-publicpickup-no-such-file-or-directory/

#### Unicorn / nginx: "Failed to set accept_filter=httpready"

This is to do with an HTTP buffering kernel module in FreeBSD that some HTTP
servers expect to be loaded. Run this:

    kldload accf_http  
    echo 'accf_http_load="YES"' >> /boot/loader.conf

    sudo service gitlab restart  
    sudo service redis restart  
    sudo service nginx restart  

http://www.cyberciti.biz/faq/failed-to-enable-the-httpready-accept-filter/

### References

    https://blog.assemblyco.de/installing-gitlab-on-freebsd-10
    https://github.com/gitlabhq/gitlabhq/blob/7-6-stable/doc/install/installation.md
    http://www.luizgustavo.pro.br/blog/2014/08/21/instalacao-gitlab-no-freebsd/
    
