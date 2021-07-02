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
* Create new Lightsail instance in AWS account (512MB/20GB at $3.50/month, ca-central-1d) via Web console
  * with fresh new keypair
  * Ubuntu 20.04
  * automatic snapshots off for now, but consider enabling them later
* Wait for instance to come up
* Allocate new static IP under Networking and assign to instance
* Connect to instance (SSH or in-browser console) and
  * `sudo -i`  # to get in a superuser shell
* Apply hostname: 
  * `hostname -b edgelink-hosted-sites`
  * You can log out of the root prompt and `sudo -i` again immediately to confirm this was set
* Do initial round of updates - with noninteractive this could probably be put in instance userdata:

```
export DEBIAN_FRONTEND=noninteractive
apt-get --yes update
apt-get --yes upgrade
```

* Ensure automatic upgrades are enabled for good measure - they do appear to be on the Lightsail Ubuntu 20.04 "AMI":
  * `cat /etc/apt/apt.conf.d/20auto-upgrades`

## Setup Tasks (from Last Time)

### Enable swapfile
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

Confirm output of `free -m`.
Reboot system to confirm that fstab entry is effective.
