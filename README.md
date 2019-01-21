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

## Virtual Machine Installation <a id="VMinstall"></a>

For this project i choose to emulate a debian 9.6.0 64bits, [Download Debian](https://www.debian.org/distrib/) hosted on macOS X with VirtualBox 5.2.18r124319 [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## OS Installation Process <a id="OSinstall"></a>

1. I choose `roger` as hostname
2. I setup the root password
3. I create a new non-root user called `gde` and his password.
4. I create a primary partition mounted on `/` with 4.2Gb of space and a other one logical mounted on `/home`
5. I choose XFCE as desktop environnement (he is really light)
6. Finally I've installed GRUB on the master boot record

## Install Depedency <a id="depedency"></a>

As root:

```bash
apt-get update -y && apt-get upgrade -y

apt-get install sudo vim resolvconf ufw ipset -y
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
gde ALL=NOPASSWD: /usr/bin/apt-get
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

3. Make sure we have a `resolv.conf` file with our favorite DNS

```bash
cat /etc/resolv.conf
```

Output

```console
nameserver 8.8.8.8
nameserver 8.8.4.4
```

otherwise, you can edit the `/ etc / resolvconf / resolv.conf.d / base` file and write them

4. You can now restart the network service to make changes effective

```bash
sudo service networking restart
```

5. You can check the result with the following command:

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
      - HTTP : `sudo ufw allow out 80/tcp`
      - DNS : `sudo ufw allow out 53/udp`
      
3. Setup Denial Of Service Attack with ufw
      -limit SSH : `sudo ufw limit 50683/tcp`
      -limit http `sudo vim  /etc/ufw/before.rules`
      
And add the following lines: 

```
### Add those lines after *filter near the beginning of the file
:ufw-http - [0:0]
:ufw-http-logdrop - [0:0]



### Add those lines near the end of the file

### Start HTTP ###

# Enter rule
-A ufw-before-input -p tcp --dport 80   -j ufw-http
-A ufw-before-input -p tcp --dport 443  -j ufw-http

# Limit connections per Class C
-A ufw-http -p tcp --syn -m connlimit --connlimit-above 50 --connlimit-mask 24 -j ufw-http-logdrop

# Limit connections per IP
-A ufw-http -m state --state NEW -m recent --name conn_per_ip --set
-A ufw-http -m state --state NEW -m recent --name conn_per_ip --update --seconds 10 --hitcount 20 -j ufw-http-logdrop

# Limit packets per IP
-A ufw-http -m recent --name pack_per_ip --set
-A ufw-http -m recent --name pack_per_ip --update --seconds 1  --hitcount 20  -j ufw-http-logdrop

# Finally accept
-A ufw-http -j ACCEPT

# Log-A ufw-http-logdrop -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW HTTP DROP] "
-A ufw-http-logdrop -j DROP

### End HTTP ###
```
> With the above rules we are limiting the connections per IP at 20 connections / 10 seconds / IP and the packets to 20 packets / second / IP for http and this does is rate limit that port to 6 new connection per ip per 30 seconds for SSH.

4. We can now block every others outgoing connection

```bash
sudo ufw default deny outgoing
```

5. (OPTIONNAL) if you want to allow ping you can add the following lines in `/etc/ufw/before.rules`

```console
# Allow ping
-A ufw-before-output -p icmp --icmp-type destination-unreachable -j ACCEPT
-A ufw-before-output -p icmp --icmp-type source-quench -j ACCEPT
-A ufw-before-output -p icmp --icmp-type time-exceeded -j ACCEPT
-A ufw-before-output -p icmp --icmp-type parameter-problem -j ACCEPT
-A ufw-before-output -p icmp --icmp-type echo-request -j ACCEPT
```

6. Finally we need to reload our firewall

```bash
sudo ufw reload
```

## Protection against port scans. <a id="scanSecure"></a>

1. First create ipset lists

```bash
ipset create port_scanners hash:ip family inet hashsize 32768 maxelem 65536 timeout 600
ipset create scanned_ports hash:ip,port family inet hashsize 32768 maxelem 65536 timeout 60
```

2. And iptables rules

```bash
iptables -A INPUT -m state --state INVALID -j DROP
iptables -A INPUT -m state --state NEW -m set ! --match-set scanned_ports src,dst -m hashlimit --hashlimit-above 1/hour --hashlimit-burst 5 --hashlimit-mode srcip --hashlimit-name portscan --hashlimit-htable-expire 10000 -j SET --add-set port_scanners src --exist
iptables -A INPUT -m state --state NEW -m set --match-set port_scanners src -j DROP
iptables -A INPUT -m state --state NEW -j SET --add-set scanned_ports src,dst
```

> Here we store scanned ports in scanned_ports set and we only count newly scanned ports on our hashlimit rule. If a scanner send packets to 5 different port (see --hashlimit-burst 5) that means it is a probably scanner so we will add it to port_scanners set.
Timeout of port_scanners is the block time of scanners(10 minutes in that example). It will start counting from beginning (see --exist) till attacker stop scan for 10 seconds (see --hashlimit-htable-expire 10000)

## Stop the services we don’t need <a id="stopServices"></a>

```bash
sudo systemctl disable anacron.servive
sudo systemctl disable anacron.timer
sudo systemctl disable lightdm.service
sudo systemctl disable avahi-daemon.service
sudo systemctl disable avahi-daemon.socket
sudo systemctl disable console-setup.service
sudo systemctl disable dbus-org.freedesktop.ModemManager1.service
sudo systemctl disable dbus-org.freedesktop.nm-dispatcher.service
sudo systemctl disable keyboard-setup.service
sudo systemctl disable lm-sensors.service
sudo systemctl disable apt-daily.timer
sudo systemctl disable apt-daily-upgrade.timer
sudo systemctl disable syslog.service
sudo systemctl disable rtkit-daemon.service
sudo systemctl disable pppd-dns.service
sudo systemctl disable NetworkManager-wait-online.service
```

## Update Packages <a id="updateApt"></a>

1. Create the `update.sh` file and write the following lines inside

```bash
echo "sudo apt-get update -y && apt-get upgrade -y >> /var/log/update_script.log" >> ~/update.sh
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
```

## Monitor Crontab Changes <a id="cronChange"></a>

1. Create the `cronMonitor.sh` file and write the following lines inside


```console
gde@roger-skyline-1:~$ cat ~/cronMonitor.sh
#!/bin/bash

FILE="/var/tmp/checksum"
FILE_TO_WATCH="/etc/crontab"
MD5VALUE=$(md5sum $FILE_TO_WATCH)

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
