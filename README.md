# aws-lamp-server
My typical setup for a ubuntu lamp server on aws


After logging in as the default "ubuntu" user, we should set a password so that we can use the aws serial console if we ever need to

```console
ubuntu@localhost:~$ sudo passwd ubuntu
```

For lockout, all we need to do is install the fail2ban package

```console
ubuntu@localhost:~$ sudo apt-get update
ubuntu@localhost:~$ sudo apt-get install fail2ban
```

If an IP ever gets banned by mistake, the command to unban it is:
```console
ubuntu@localhost:~$ sudo fail2ban-client unban m.y.i.p
```

for security sake, security patches should be installed automatically. To do that, we need to install the unattended-upgrades package
```console
administrator@localhost:~$ sudo apt-get install unattended-upgrades
```

Then we need to configure the package by editing the config
```console
administrator@localhost:~$ sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

in `Allowed-Origins` we want to make sure that "security" related lines are uncommented. The following settings should also be enabled/configured:

```
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Automatic-Reboot-Time "07:00";
```

The automatic upgrades are still not actually enabled, to do that we have to edit another file:
```console
administrator@localhost:~$ sudo nano /etc/apt/apt.conf.d/10periodic
```
and set the following settings:
```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

even now the automatic upgrades are not entirely set up. The whole thing will come to a hault and the system will stop upgrading if dpkg ever wants to prompt with a question of if it should replace a config file you modified with the default config file in the upgraded package. To prevent this, we need to just preemptively answer it's question now
```console
administrator@localhost:~$ sudo nano /etc/apt/apt.conf.d/local
```
This file does not exist by default, but we will populate it with one of the following:

So the question dpkg would be asking is:
```
Configuration file `/path/to/something.conf'
 ==> Modified (by you or by a script) since installation.
 ==> Package distributor has shipped an updated version.
   What would you like to do about it ?  Your options are:
    Y or I  : install the package maintainer's version
    N or O  : keep your currently-installed version
      D     : show the differences between the versions
      Z     : start a shell to examine the situation
```

if you would answer "N" then this is the conf you want (This is the setting I usually go with):
```
# keep old configs on upgrade, move new versions to <file>.dpkg-dist
Dpkg::Options {
   "--force-confdef";
   "--force-confold";
}
```

but if you want to answer "Y" then this is the conf you want:
```
# keep new configs on upgrade, move old versions to <file>.dpkg-old
Dpkg::Options {
   "--force-confdef";
   "--force-confnew";
}
```


You can confirm that the `/etc/apt/apt.conf.d/local` conf is taking effect with:
```console
administrator@localhost:~$ apt-config dump | grep "DPkg::Options"
```

You can now test the automatic upgrades by running:

```console
administrator@localhost:~$ sudo unattended-upgrades --dry-run
```
There should be no errors


You can see the next time the system will DOWNLOAD updates with:

```console
administrator@localhost:~$ systemctl list-timers apt-daily --all
```

You can edit the DOWNLOAD schedule with:

```console
administrator@localhost:~$ sudo systemctl --full edit apt-daily.timer
administrator@localhost:~$ sudo systemctl restart apt-daily.timer
administrator@localhost:~$ sudo systemctl status apt-daily.timer
```

You can see the next time the system will INSTALL updates with:

```console
administrator@localhost:~$ systemctl list-timers apt-daily-upgrade --all
```

You can edit the INSTALL schedule with:

```console
administrator@localhost:~$ sudo systemctl --full edit apt-daily-upgrade.timer
administrator@localhost:~$ sudo systemctl restart apt-daily-upgrade.timer
administrator@localhost:~$ sudo systemctl status apt-daily-upgrade.timer
```

The next thing I do to squeeze out all the performance I can from an aws instance is to compress the memory. This is only a good idea if your application is memory-contrained and not something that you would want to do if you were running a cpu-constrained app. But my workloads are always memory-constrained
```console
administrator@localhost:~$ sudo apt-get install zram-config linux-modules-extra-aws
```

now we need to reboot so the new kernel will be used
```console
administrator@localhost:~$ sudo reboot
```

Next we should make a swap file. 

```console
administrator@localhost:~$ sudo fallocate -l 2G /swapfile
administrator@localhost:~$ sudo chmod 600 /swapfile
administrator@localhost:~$ sudo mkswap /swapfile
administrator@localhost:~$ sudo swapon /swapfile
administrator@localhost:~$ echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

digitalocean reccommends that we should tweak the swap for better performance like so:

```console
administrator@localhost:~$ sudo sysctl vm.swappiness=10
administrator@localhost:~$ sudo sysctl vm.vfs_cache_pressure=50
administrator@localhost:~$ echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
administrator@localhost:~$ echo 'vm.vfs_cache_pressure=50' | sudo tee -a /etc/sysctl.conf
```

we can check that the swap (and memory compression) is working with:
```console
administrator@localhost:~$ swapon -s
```

Since this is a VM, there is a risk of the enprophy pool drying up, which can cause webpages not to load if this is a webserver which needs to generate keys for example. So to prevent that, we need to install haveged

```console
administrator@localhost:~$ sudo apt-get install haveged
```

Now we can install/configure the lamp stack

```console
administrator@localhost:~$ sudo apt-get install apache2 mariadb-server php php-gd php-imap php-xml php-mbstring php-intl php-curl php-memcached php-apcu php-mysqli memcached brotli zip sendmail
administrator@localhost:~$ mysql_secure_installation
```

The first thing you should do is configure the permissions on the webroot directory so that this administrator user can manipulate webpage files yet the permission are still secure.

```console
administrator@localhost:~$ cd /var/www
administrator@localhost:~$ sudo chown www-data:www-data -R * # Let Apache be owner of subdirs
administrator@localhost:~$ sudo chown www-data:www-data -R . # Let Apache be owner of this dir
administrator@localhost:~$ sudo usermod -a -G www-data administrator # add administrator to www-data group
administrator@localhost:~$ sudo usermod -a -G www-data www-data # add to www-data to www-data group
administrator@localhost:~$ sudo find . -type d -exec chmod 2775 {} \;  # Change directory permissions rwxr-xr-x
administrator@localhost:~$ sudo find . -type f -exec chmod 664 {} \;  # Change file permissions rw-r--r--
```
NOTE: these group changes will not take effect until you log out and back in, so maybe do that now.
Note that you can run these commands again anytime you have website permission issues to reset everything to sane defaults

if we want just just use cloudflare to manage our ssl, and we are not worried about the MITM attack between AWS and cloudflare, then we can disable ssl in apache with:
```console
administrator@localhost:~$ sudo a2dismod ssl
administrator@localhost:~$ sudo service apache2 restart
```

We can now edit the default apache config:
```console
administrator@localhost:~$ sudo nano /etc/apache2/sites-available/000-default.conf
```

which should at least involve setting the serveradmin email but may also involve moving the documentRoot

We can enable the use of `.htaccess` files by editing:
```console
administrator@localhost:~$ sudo nano /etc/apache2/apache2.conf
```
and setting AllowOverride All in /var/www:
```
<Directory /var/www/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
</Directory>
```
and append to the bottom of `apache2.conf`:

```
ServerName 127.0.0.1
```

and then restart apache

```console
administrator@localhost:~$ sudo service apache2 restart
```

now we can set up a virualhost like this 
```console
administrator@localhost:~$ cd /etc/apache2/sites-available/
administrator@localhost:~$ sudo cp 000-default.conf example.com.conf
administrator@localhost:~$ sudo nano example.com.conf
```

change the `DocumentRoot` path and add:

```
ServerName example.com
ServerAlias www.example.com
```

then enable and activate the site with:

```console
administrator@localhost:~$ sudo a2ensite example.com
administrator@localhost:~$ sudo systemctl reload apache2
```

here is a redirect index.html that can be used for the default docroot:
```html
<!DOCTYPE html><html lang='en'><title>redirect</title><script>window.location.replace("https://www.example.com");</script>redirecting to <a href='https://www.example.com'>example.com</a>
```

likely want to have the mod_rewrite and brotli modules up and running
```console
administrator@localhost:~$ sudo a2enmod brotli
administrator@localhost:~$ sudo a2enmod rewrite
administrator@localhost:~$ sudo systemctl restart apache2
```

At some point you will need to connect to mysql as root. To enable that do:
```
sudo mysql -u root
use mysql;
update user set plugin='' where User='root';
flush privileges;
exit;
```

php may not be able to connect to mysql unless you edit the conf file:
```console
administrator@localhost:~$ sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

and add the following option under `[mysqld]`
```
default_authentication_plugin= mysql_native_password
character-set-server= utf8
```

and then restart the service with 
```console
administrator@localhost:~$ sudo service mysql restart 
```

adminter can be downloaded to manage mysql via gui:
```console
administrator@localhost:~$ wget https://github.com/vrana/adminer/releases/download/v4.7.7/adminer-4.7.7.php
```

### In the future, if we run out of disk space, the following can be used to expand the filesystem (after expanding the disk in the aws console)


First, find the disk and partition of / using lsblk
```console
administrator@localhost:~$ sudo lsblk
```

Assuming the disk is /dev/nvme0n1, we first need to expand the GPT disk:
```console
administrator@localhost:~$ sudo sgdisk -e /dev/nvme0n1
```

partprobe allows the system to learn of the changes
```console
administrator@localhost:~$ sudo partprobe
```

We will use growpart to expand the partition, but if the partition is full, then growpart can't work because it won't be able to write to /tmp.
To work around that, we can make /tmp a ramdrive for a moment. Here we assume the partition we want to grow is partition 1 (/dev/nvme0n1p1) adjust accordingly 
```console
administrator@localhost:~$ sudo mount -o size=10M,rw,nodev,nosuid -t tmpfs tmpfs /tmp
administrator@localhost:~$ sudo growpart /dev/nvme0n1 1
administrator@localhost:~$ sudo umount /tmp
```

The last step is to expand the filesystem:
```console
administrator@localhost:~$ sudo resize2fs /dev/nvme0n1p1
```

Now you should see that you have plenty of free space
```console
administrator@localhost:~$ df -h
```
