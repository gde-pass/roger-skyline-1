# roger-skyline-1
This project, roger-skyline-1 let you install a Virtual Machine, discover the basics about system and network administration as well as a lots of services used on a server machine.

## Summary <a id="summary"></a>

- [Summary](#summary)
- [Virtual Machine Installation](#VMinstall)
- [OS Installation Process](#OSinstall)
- [Install Depedency](#depedency)
- [Setup a static IP](#staticIP)
- [Change SSH default Port](#sshPort)
- [Setup SSH access with publickeys](#sshKey)
- [Setup Firewall with UFW](#ufw)
- [Protection against port scans](#scanSecure)
- [Stop the services we don’t need](#stopServices)
- [Update Packages](#updateApt)
- [Monitor Crontab Changes](#cronChange)
- [Deploy a Web application reacheable on the machine IP's](#apache)

## Virtual Machine Installation <a id="VMinstall"></a>

For this project i choose to emulate a debian 9.6.0 64bits, [Download Debian](https://www.debian.org/distrib/) hosted on macOS X with VirtualBox 5.2.18r124319 [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## OS Installation Process <a id="OSinstall"></a>

1. I choose `roger` as hostname
2. I setup the root password
3. I create a new non-root user called `gde` and his password.
4. I create a primary partition mounted on `/` with 4.2Gb of space and a other one logical mounted on `/home`
5. I choose not to install desktop environnement
6. Finally I've installed GRUB on the master boot record

## Install Depedency <a id="depedency"></a>

As root:

```bash
apt-get update -y && apt-get upgrade -y

apt-get install sudo vim ufw portsentry fail2ban apache2 mailutils -y
```

## Configure SUDO <a id="sudo"></a>

Right after we have installed `sudo`, if we try to use it we will have this error message:
`gde is not in the sudoers file.`

To fix it we have to edit the file `/etc/sudoers` with the command `visudo`.

1. First you have to login as root:

```bash
su
```

2. Just type `visudo` and edit the file to have this output

```bash
cat /etc/sudoers
```

Output:

```console
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbi$

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL
gde     ALL=(ALL:ALL) NOPASSWD:ALL

# Members of the admin group may gain root privileges

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
```


## Setup a static IP <a id="staticIP"></a>

In the settings of our virtualbox machine you have to change the default `NAT` Network Adapter by `Bridged Adapter`

1. First, we have to edit the file `/etc/network/interfaces` and setup our primary network

```bash
cat /etc/network/interfaces
```

Output:

```console
source /etc/network/interfaces.d/*

#The loopback Network interface
auto lo
iface lo inet loopback

#The primary network interface
auto enp0s3
```

2. Now we have to configure this network with a static ip, to do that properly, we will create a file named `enp0s3` in the following directory `etc/network/interfaces.d/`

```bash
cat /etc/network/interfaces.d/enp0s3
```

Output:

```console
iface enp0s3 inet static
      address 10.11.200.247
      netmask 255.255.255.252
      gateway 10.11.254.254
```

3. You can now restart the network service to make changes effective

```bash
sudo service networking restart
```

4. You can check the result with the following command:

```bash
ip addr
```

## Change SSH default Port <a id="sshPort"></a>

1. As root, use your favorite text editor to edit the sshd configuration file.

```bash
sudo vim /etc/ssh/sshd_config
```

2. Edit the line 13 which states 'Port 22'. But before doing so, you'll want to read the note below. Choose an appropriate port, also making sure it not currently used on the system.

```
Port 50683
```

> **Note**: The Internet Assigned Numbers Authority (IANA) is responsible for the global coordination of the DNS Root, IP addressing, and other Internet protocol resources. It is good practice to follow their port assignment guidelines. Having said that, port numbers are divided into three ranges: Well Known Ports, Registered Ports, and Dynamic and/or Private Ports. The Well Known Ports are those from 0 through 1023 and SHOULD NOT be used. Registered Ports are those from 1024 through 49151 should also be avoided too. Dynamic and/or Private Ports are those from 49152 through 65535 and can be used. Though nothing is stopping you from using reserved port numbers, our suggestion may help avoid technical issues with port allocation in the future.

3. We can now login with ssh.

```bash
ssh gde@10.11.200.247 -p 50683
```

## Setup SSH access with publickeys. <a id="sshKey"></a>

1. First we have to generate a public/private rsa key pair, on the host machine (Mac OS X in my case).

```bash
ssh-keygen -t rsa
```

This command will generate 2 files `id_rsa` and `id_rsa.pub`

- **id_rsa**:  Our private key, should be keep safely, She can be crypted with a password.
- **id_rsa.pub** Our private key, you have to transfer this one to the server.

2. To do that we can use the `ssh-copy-id` command

```bash
ssh-copy-id -i id_rsa.pub gde@10.11.200.247 -p 50683
```

The key is automatically added in `~/.ssh/authorized_keys` on the server

> If you no longer want to have type the key password you can setup a SSH Agent with `ssh-add`

3. Edit the `sshd_config` file `/etc/ssh/sshd.config` to remove root login permit, password authentification 

```bash
sudo vim /etc/ssh/sshd.conf
```

- Edit line 32 like: `PermitRootLogin no`
- Edit line 56 like `PasswordAuthentication no`
> Don't forget to delete de **#** at the beginning of each line

4. We need to restart the SSH daemon service.

```bash
sudo service sshd restart
```

## Setup Firewall with UFW. <a id="ufw"></a>

1. Make sure ufw is enable

```bash
sudo ufw status
```
 if not we can start the service with
 
 ```bash
 sudo ufw enable
 ```
 
2. Setup firewall rules
      - SSH : `sudo ufw allow 50683/tcp`
      - HTTP : `sudo ufw allow 80/tcp`
      - HTTPS : `sudo ufw allow 443`
      
3. Setup Denial Of Service Attack with fail2ban
      
```bash
sudo vim /etc/fail2ban/jail.conf
```

```console
[sshd]
enabled = true
port    = 42
logpath = %(sshd_log)s
backend = %(sshd_backend)s
maxretry = 3
bantime = 600

#Add after HTTP servers:
[http-get-dos]
enabled = true
port = http,https
filter = http-get-dos
logpath = /var/log/apache2/access.log (le fichier d'access sur server web)
maxretry = 300
findtime = 300
bantime = 600
action = iptables[name=HTTP, port=http, protocol=tcp]
```

Add http-get-dos filter

```bash
sudo cat /etc/fail2ban/filter.d/http-get-dos.conf
```

Output:

```console
[Definition]
failregex = ^<HOST> -.*"(GET|POST).*
ignoreregex =
```

4. (OPTIONNAL) if you want to allow ping you can add the following lines in `/etc/ufw/before.rules`

```console
# Allow ping
-A ufw-before-output -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-output -p icmp --icmp-type source-quench -j ACCEPT
-A ufw-before-output -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-output -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-output -p icmp --icmp-type echo-request -j ACCEPT
```

5. Finally we need to reload our firewall and fail2ban

```bash
sudo ufw reload
sudo service fail2ban restart
```

## Protection against port scans. <a id="scanSecure"></a>

1. Config portsentry

First, we have to edit the `/etc/default/portsentry` file

```console
TCP_MODE="atcp"
UDP_MODE="audp"
```

After, edit the file `/etc/portsentry/portsentry.conf`

```console
BLOCK_UDP="1"
BLOCK_TCP="1"
```

Comment the current KILL_ROUTE and uncomment the following one:

```console
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```

Comment the following line:

```console
KILL_HOSTS_DENY="ALL: $TARGET$ : DENY
```

2. We can now restart the service to make changes effectives

```bash
sudo service portsentry restart
```

## Stop the services we don’t need <a id="stopServices"></a>

```bash
sudo systemctl disable console-setup.service
sudo systemctl disable keyboard-setup.service
sudo systemctl disable apt-daily.timer
sudo systemctl disable apt-daily-upgrade.timer
sudo systemctl disable syslog.service
```

## Update Packages <a id="updateApt"></a>

1. Create the `update.sh` file and write the following lines inside

```bash
echo "sudo apt-get update -y >> /var/log/update_script.log" >> ~/update.sh
echo "sudo apt-get upgrade -y >> /var/log/update_script.log" >> ~/update.sh
```

2. Add the task to cron

```bash
sudo crontab -e
```

3. Write in the openned file thoses lines

```bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

@reboot sudo ~/update.sh
0 4 * * 6 sudo ~/update.sh
```

## Monitor Crontab Changes <a id="cronChange"></a>

1. Create the `cronMonitor.sh` file and write the following lines inside

```console
gde@roger-skyline-1:~$ cat ~/cronMonitor.sh
#!/bin/bash

FILE="/var/tmp/checksum"
FILE_TO_WATCH="/var/spool/cron/crontabs/gde"
MD5VALUE=$(sudo md5sum $FILE_TO_WATCH)

if [ ! -f $FILE ]
then
	 echo "$MD5VALUE" > $FILE
	 exit 0;
fi;

if [ "$MD5VALUE" != "$(cat $FILE)" ];
	then
	echo "$MD5VALUE" > $FILE
	echo "$FILE_TO_WATCH has been modified ! '*_*" | mail -s "$FILE_TO_WATCH modified !" root
fi;
```

2. Add the task to cron

```bash
crontab -e
```

3. Write in the openned file thoses lines

```bash
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin

@reboot sudo ~/update.sh
0 4 * * 6 sudo ~/update.sh
0 0 * * * sudo ~/cronMonitor.sh
```

4. Make sure we have correct rights

```bash
sudo chmod 755 cronMonitor.sh
sudo chmod 755 update.sh
sudo chown gde /var/mail/gde
```

5. make sure cron service is enable

```bash
sudo systemctl enable cron
```

## Deploy a Web application reacheable on the machine IP's <a id="apache"></a>

You just have to copy into the folder `/var/www/html/` your web application.

## Configure SSL Certificates


Generate the SLL certificate with this command

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -subj "/C=FR/ST=IDF/O=42/OU=Project-roger/CN=10.11.200.247" -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```

Create the `/etc/apache2/conf-available/ssl-params.conf` file to have this output:

```console
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLHonorCipherOrder On

Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff

SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"

SSLSessionTickets Off
```

Edit the `/etc/apache2/sites-available/default-ssl.conf` to have this output

```console

<IfModule mod_ssl.c>
	<VirtualHost _default_:443>
		ServerAdmin gde-pass@student.42.fr
		ServerName	192.168.99.100

		DocumentRoot /var/www/html

		ErrorLog ${APACHE_LOG_DIR}/error.log
		CustomLog ${APACHE_LOG_DIR}/access.log combined

		SSLEngine on

		SSLCertificateFile	/etc/ssl/certs/apache-selfsigned.crt
		SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key

		<FilesMatch "\.(cgi|shtml|phtml|php)$">
				SSLOptions +StdEnvVars
		</FilesMatch>
		<Directory /usr/lib/cgi-bin>
				SSLOptions +StdEnvVars
		</Directory>

	</VirtualHost>
</IfModule>
```

And finally Edit the `/etc/apache2/sites-available/000-default.conf` file and writte the followings line inside

```console
<VirtualHost *:80>

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html

	Redirect "/" "https://192.168.99.100/"

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

And to load our new config run those commands:

```bash
sudo a2enmod ssl
sudo a2enmod headers
sudo a2ensite default-ssl
sudo a2enconf ssl-params
systemctl reload apache2
```
