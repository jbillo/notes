# notes
Scratch area for notes while working on my main website

# 2020-07-01
I'm looking to retry the Lightsail + WordPress migration for some services I still am responsible for hosting on Linode. Last time I did this I ran it with Ansible + Terraform but that's a huge maintenance headache. I think the fix will be, if the Lightsail instance goes down, to pay for snapshots - if that fails rebuild the system and restore an at-most 1-day old backup of the databases and relevant filesystem contents from S3. Key things to handle:

* Initial Lightsail host configuration - likely OK if this is somewhat manual as long as the steps are documented, and may let us go to whatever the latest Ubuntu or Amazon Linux 2 version is at the time
* Let's Encrypt certificates need DNS validation (not HTTP, since the record will still be pointed at Linode at the time I'd like to set things up)
* Back up the website contents and database, then shuttle them over to Lightsail and import them successfully
* nginx + php + mysql configuration sanity
* admin interfaces for all of this (phpmyadmin) bound to localhost - port forward for this (SOCKS proxy again? wireguard?)
* wp2static support for my own site as a nice-to-have (can run "lokl" maybe with some Docker containers on a large enough host?)

## Getting Started
* Create new or use existing S3 bucket and get its ARN
* Create new IAM user with access key (needed for Lightsail or non-AWS) with the appropriate policy attached, replacing _MY_BUCKET_NAME_ with your real bucket name:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListBackupBucket",
            "Action": [
                "s3:ListBucket"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::MY_BUCKET_NAME"
            ]
        },
        {
            "Sid": "GetObjectsFromBackupBucket",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::MY_BUCKET_NAME/*"
            ]
        },
        {
            "Sid": "PutObjectsToBackupBucket",
            "Action": [
                "s3:PutObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::MY_BUCKET_NAME/*"
            ]
        }            
    ]
}
```

* Create new Lightsail instance in AWS account (512MB/20GB at $3.50/month, ca-central-1d) via Web console
  * with fresh new keypair
  * Ubuntu 20.04
  * automatic snapshots off for now, but consider enabling them later
* Wait for instance to come up
* Allocate new static IP under Networking and assign to instance
* Create new, or use existing Lightsail DNS zone and add A record for this host (convenience and initial virtual host testing)
* Connect to instance (SSH or in-browser console) and
  * `sudo -i`  # to get in a superuser shell
* Apply hostname: 
  * `hostnamectl set-hostname edgelink-hosted-sites`
  * You can log out of the root prompt and `sudo -i` again immediately to confirm this was set for the current session
* Do initial round of updates - with noninteractive this could probably be put in instance userdata:

```
export DEBIAN_FRONTEND=noninteractive
apt-get --yes update
apt-get --yes upgrade
```

* Ensure automatic upgrades are enabled for good measure - they do appear to be on the Lightsail Ubuntu 20.04 "AMI":
  * `cat /etc/apt/apt.conf.d/20auto-upgrades`

## Enable swapfile
512MB isn't a whole lot of dedotated WAM. Enable a swapfile since we have some moderate EBS gp2 storage:

```
dd if=/dev/zero of=/swapfile bs=1M count=512
chmod 600 /swapfile
chown root /swapfile
chgrp root /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

`sysctl vm.swappiness` is already 60 on this AMI so no need to adjust.

Confirm output of `free -m` contains a `Swap:` line.
Reboot system to confirm that fstab entry and persistent hostname is effective. Remember to `sudo -i` your way to success.

## system and server utils

```
# handy utilities
apt-get --yes install python3-pip whois traceroute unzip fail2ban

# certbot
snap install core; snap refresh core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot

# awscliv2
mkdir -p /tmp/awscliv2; cd /tmp/awscliv2; curl -O https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip; unzip *.zip; aws/install; cd -

# php
apt-get --yes install php-pear php-fpm php-dev php-zip php-curl php-xmlrpc php-gd php-mysql php-mbstring php-xml
systemctl enable php7.4-fpm
systemctl start php7.4-fpm
systemctl status php7.4-fpm

# nginx
apt-get --yes install nginx
service nginx status

# MariaDB (mysql)
apt-get --yes install mariadb-server
mysql_secure_installation  # interactive, no root password, don't set root password, accept all other defaults
```

## wp-cli and daily updates to it
```
# awesome, curl the direct executable from GitHub!
cd /tmp
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
sudo mv wp-cli.phar /usr/local/bin/wp
cd -
wp --allow-root --no-color --info
cat > /etc/cron.daily/wp-cli-update <<EOT
#!/bin/sh
/usr/local/bin/wp cli update --allow-root --yes
EOT
chmod +x /etc/cron.daily/wp-cli-update

```

## build first nginx site structure and templates

Note that the `php-site` template below has a 64MB client upload (POST) limit, you may want to go in and adjust this.
I dislike being errored out when uploading large JPEGs to my own site.

```
SITE_NAME="edgelink.example.com"
dig $SITE_NAME
mkdir -p "/srv/$SITE_NAME"
mkdir -p "/var/log/nginx/$SITE_NAME"
cat > "/srv/$SITE_NAME/index.html" <<EOT
Site $SITE_NAME being served through nginx

EOT
cat "/srv/$SITE_NAME/index.html"
chown -R www-data:www-data "/srv/$SITE_NAME"
mkdir -p /etc/nginx/templates; cd /etc/nginx/templates

# https://serverfault.com/a/399432
cat > "/etc/nginx/templates/static-site" <<"EOT" 
server {
    root /srv/{{ SITE_NAME }};
    index index.html index.htm index.nginx-debian.html;
    server_name {{ SITE_NAME }} www.{{ SITE_NAME }};
    access_log /var/log/nginx/{{ SITE_NAME }}/access.log;
    error_log /var/log/nginx/{{ SITE_NAME }}/error.log;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    # Deny access to any files with a .php extension AT ALL
    location ~* .*\.php$ {
        deny all;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }
    
    location / {
        try_files $uri $uri/ /index.html$is_args$args /index.htm$is_args$args;
    }

    location ~* \.(gif|jpg|jpeg|png|css|js)$ {
        expires max;
    }
}

EOT


cat > "/etc/nginx/templates/php-site" <<"EOT"
server {
    root /srv/{{ SITE_NAME }};
    index index.php index.html index.htm index.nginx-debian.html;
    server_name {{ SITE_NAME }} www.{{ SITE_NAME }};
    access_log /var/log/nginx/{{ SITE_NAME }}/access.log;
    error_log /var/log/nginx/{{ SITE_NAME }}/error.log;

    client_max_body_size 64M;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    # Deny access to any files with a .php extension in the uploads directory
    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~* \.(gif|jpg|jpeg|png|css|js)$ {
        expires max;
    }

    # pass the PHP scripts to FastCGI server
    location ~ \.php$ {
        try_files $uri =404;
        include /etc/nginx/fastcgi_params;
        fastcgi_intercept_errors on;
        fastcgi_read_timeout 3600s;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_pass unix:/run/php/php7.4-fpm.sock;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_index index.php;
    }
}
EOT

sed "s/{{ SITE_NAME }}/$SITE_NAME/g" /etc/nginx/templates/static-site > /etc/nginx/sites-enabled/$SITE_NAME

service nginx reload
curl http://$SITE_NAME

```

## For my next act, build the stub for the next (WordPress) website

```
SITE_NAME="example.com"
mkdir -p "/srv/$SITE_NAME"
mkdir -p "/var/log/nginx/$SITE_NAME"
sed "s/{{ SITE_NAME }}/$SITE_NAME/g" /etc/nginx/templates/php-site > /etc/nginx/sites-enabled/$SITE_NAME
service nginx reload
```

## Bundle up the previous database and website (on the remote system, assuming wp-cli is also installed there)

```
SITE_NAME="example.com"
BUCKET_NAME="example-s3"
wp db export --allow-root --path="/srv/$SITE_NAME" /tmp/$SITE_NAME-db.sql
gzip /tmp/$SITE_NAME-db.sql  # should create .sql.gz file
cd /srv/$SITE_NAME; tar czvf /tmp/$SITE_NAME-www.tar.gz .; cd -

export AWS_ACCESS_KEY_ID="AKIAEXAMPLE9999999"
export AWS_SECRET_ACCESS_KEY="nothingtoseehere"

aws s3 cp /tmp/$SITE_NAME-db.sql.gz s3://$BUCKET_NAME/
aws s3 cp /tmp/$SITE_NAME-www.tar.gz s3://$BUCKET_NAME/
```

## Pull down the database and website from S3 on the new Lightsail instance

```
SITE_NAME="example.com"
BUCKET_NAME="example-s3"
export AWS_ACCESS_KEY_ID="AKIAEXAMPLE9999999"
export AWS_SECRET_ACCESS_KEY="nothingtoseehere"

aws s3 cp s3://$BUCKET_NAME/$SITE_NAME-db.sql.gz /tmp/
aws s3 cp s3://$BUCKET_NAME/$SITE_NAME-www.tar.gz /tmp/
```

## Extract the website contents
This allows us to read wp-config.php to get the username/password used for MySQL/MariaDB, so we can then create those credentials and the database for the import.

```
tar xvf /tmp/$SITE_NAME-www.tar.gz -C /srv/$SITE_NAME
```

## Get the database credentials and create the necessary user

This was somewhat painful not having written PHP for many years. 

```
php > /dev/null <<EOT
<?php
function write_db_vars() {
    \$output = "#!/bin/sh\n";
    \$output .= 'export DB_NAME="' . DB_NAME . "\"\n";
    \$output .= 'export DB_USER="' . DB_USER . "\"\n";
    \$output .= 'export DB_PASSWORD="' . DB_PASSWORD . "\"\n";
    file_put_contents('/tmp/$SITE_NAME.dbcreds.sh', \$output);
}
register_shutdown_function('write_db_vars');
include('/srv/$SITE_NAME/wp-config.php');
write_db_vars();

EOT

source /tmp/$SITE_NAME.dbcreds.sh

mysql mysql <<EOT
CREATE DATABASE IF NOT EXISTS $DB_NAME;
GRANT USAGE ON $DB_NAME.* TO '$DB_USER'@localhost IDENTIFIED BY '$DB_PASSWORD';
EOT
```

