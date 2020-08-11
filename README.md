# aws-lamp-server
My typical setup for a ubuntu lamp server on aws


After logging in as the default "ubuntu" user, we can create a new admin user and delete that one

```console
ubuntu@localhost:~$ sudo useradd -s /bin/bash -m -d /home/administrator -g sudo administrator
ubuntu@localhost:~$ sudo usermod -a -G sudo administrator
ubuntu@localhost:~$ sudo passwd administrator
```

I am not a big fan of the default cert-based auth, I think you can have just as much security with very strong passwords and a good lockout policy. For lockout, all we need to do is install the fail2ban package

```console
ubuntu@localhost:~$ sudo apt-get update
ubuntu@localhost:~$ sudo apt-get install fail2ban
```

If an IP ever gets banned by mistake, the command to unban it is:
```console
ubuntu@localhost:~$ sudo fail2ban-client unban m.y.i.p
```

Now to disable cert-based auth, we need to do
```console
ubuntu@localhost:~$ sudo nano /etc/ssh/sshd_config
```
and change these settings:
```
PubkeyAuthentication no
PasswordAuthentication yes
```
and then restart ssh (can be done remotely, will not disconnect session)
```console
ubuntu@localhost:~$ sudo service ssh restart
```

now we can connect to ssh as the administrator user created easlier with only a password and remove the ubuntu user
```console
administrator@localhost:~$ sudo userdel ubuntu
administrator@localhost:~$ sudo rm /home/ubuntu/ -R
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

And if you want to answer "Y" then this is the conf you want:
```
# keep old configs on upgrade, move new versions to <file>.dpkg-old
Dpkg::Options {
   "--force-confdef";
   "--force-confnew";
}
```

But if you would answer "N" then this is the conf you want (This is the setting I usually go with):
```
# keep old configs on upgrade, move new versions to <file>.dpkg-dist
Dpkg::Options {
   "--force-confdef";
   "--force-confold";
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

The next thing I do to squeeze out all the performance I can from an aws instance is to compress the memory. This is only a good idea if your application is memory-contrained and not something that you would want to do if you were running a cpu-constrained app. But my workloads are always memory-constrained
```console
administrator@localhost:~$ sudo apt-get install zram-config linux-image-extra-virtual
administrator@localhost:~$ sudo apt-get install linux-virtual
administrator@localhost:~$ sudo apt-get purge linux-*aws 
```
Running that last command will throw a warning that you are uninstalling the kernel that you are using and ask to abort. Do not abort, 

