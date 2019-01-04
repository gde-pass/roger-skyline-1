# roger-skyline-1
This project, roger-skyline-1 let you install a Virtual Machine, discover the basics about system and network administration as well as a lots of services used on a server machine.

## Virtual Machine Installation

For this project i choose to emulate a debian 9.6.0 64bits, [Download Debian](https://www.debian.org/distrib/) hosted on macOS X with VirtualBox 5.2.18r124319 [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## OS Installation Process

1. I choose `roger` as hostname
2. I setup the root password
3. I create a new non-root user called `gde` and his password.
4. I create a primary partition mounted on `/` with 4.2Gb of space and a other one logical mounted on `/home`
5. I choose XFCE as desktop environnement (he is really light)
6. Finally I've installed GRUB on the master boot record

## Install Depedency

```bash
apt-get update -y && apt-get upgrade -y

apt-get install sudo
apt-get install vim
```

## Configure SUDO

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

```
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
gde     ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL

# See sudoers(5) for more information on "#include" directives:

#includedir /etc/sudoers.d
```


## Setup a static IP

1. First, we have to edit the file `/etc/network/interfaces` and setup our primary network

```bash
cat /etc/network/interfaces
```

Output:

```
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

```
iface enp0s3 inet static
      address 10.0.2.0
      netmask 255.255.255.252
      gateway 10.0.2.2
```

3. Make sure we have a `resolv.conf` file with our favorite DNS

```bash
cat /etc/resolv.conf
```

Output

```
nameserver 8.8.8.8
nameserver 8.8.4.4
```

4. You can now restart the network service to make changes effective

```bash
sudo service networking restart
```

5. You can check the result with the following command:

```bash
ip addr
```
