## Tested on:
 - Ubuntu 18 / 20
 - Debian 10
 - RHEL Family 7 / 8

---
# Dot files

```bash
cat > ~/init.sh << "EndOfMessage"
#!/bin/bash

find "${HOME}" -mindepth 1 -maxdepth 1 -exec rm -rf {} \;

(apt-get update || yum check-update) 2>/dev/null
(apt-get install git -y || yum install git -y) 2>/dev/null

git init .
git remote add origin https://github.com/theanotherwise/dotfiles.git
git fetch --all
git checkout master

/bin/bash ~/.dotfiles.sh

rm -rf ~/.git ~/.gitignore ~/.dotfiles.sh
EndOfMessage

/bin/bash ~/init.sh
```

---
# Scripts

```bash
INIT_DIR="${HOME}/scripts/init" ; mkdir -p "${INIT_DIR}"

INIT_PATH="${INIT_DIR}/setup.sh"

cat > "${INIT_PATH}" << "EndOfMessage"
#!/bin/bash

(apt-get update || yum check-update) 2>/dev/null

(apt-get -y upgrade && apt-get -y dist-upgrade ||
yum update -y) 2>/dev/null

(apt-get install chrony lsb-release cron rsyslog -y ||
yum install chrony redhat-lsb-core cronie rsyslog -y) 2>/dev/null

DISTRO_NAME="`(lsb_release -a 2>/dev/null | grep -Po "Distributor ID:\t\K[a-zA-Z]*")`"

if [ "${DISTRO_NAME}" == "Ubuntu" ] || [ "${DISTRO_NAME}" == "Debian" ] ; then
  dpkg-reconfigure openssh-server
elif [ "${DISTRO_NAME}" == "OracleServer" ] ; then
  ssh-keygen -q -N '' -t rsa -f /etc/ssh/ssh_host_rsa_key
  ssh-keygen -q -N '' -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
  ssh-keygen -q -N '' -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
fi

BASE_SERVICES="ssh sshd chrony chronyd cron crond rsyslog"

for i in ${BASE_SERVICES} ; do
  systemctl enable "${i}" 2>/dev/null
  systemctl start "${i}" 2>/dev/null
done
EndOfMessage

#####################################################################

INIT_PATH="${INIT_DIR}/clean.sh"

cat > "${INIT_PATH}" << "EndOfMessage"
#!/bin/bash

rm -rfv /etc/ssh/ssh_host_*

BASE_SERVICES="ssh sshd chrony chronyd cron crond rsyslog"

for i in ${BASE_SERVICES} ; do
  systemctl disable "${i}" 2>/dev/null
  systemctl stop "${i}" 2>/dev/null
done

(apt-get autoremove --purge -y apparmor ufw &&
apt-get autoremove --purge &&
apt-get autoclean &&
apt-get clean ||
yum clean all) 2>/dev/null

for i in /tmp /var/log /var/tmp /var/cache ; do
  find $i -maxdepth 1 -mindepth 1 -exec rm -rf {} \;
done

(dpkg -l | grep linux-image ||
package-cleanup --oldkernels --count=1 ||
dnf remove --oldinstallonly --setopt installonly_limit=1 kernel -y) 2>/dev/null

(update-grub2 ||
grub2-mkconfig -o /boot/grub2/grub.cfg) 2>/dev/null

history -w
history -c
EndOfMessage

#####################################################################

INIT_PATH="${INIT_DIR}/report.sh"

cat > "${INIT_PATH}" << "EndOfMessage"
clear ; clear

echo -e "\e[31mHostname\e[m: `hostname`\n" 
echo -e "\e[31mHostname (FQDN)\e[m: `hostname -f`\n"

echo -e "\e[31mLSB Release\e[m\n`lsb_release -a 2>&1`\n"
echo -e "\e[31mUname\e[m: `uname -a`\n"

echo -e "\e[31m/etc/hosts\e[m\n`cat /etc/hosts`\n"
echo -e "\e[31m/etc/resolv.conf\e[m\n`cat /etc/resolv.conf`\n"

echo -e "\e[31mNetwork configuration\e[m"

(cat /etc/network/interfaces.d/eth0.conf || 
cat /etc/sysconfig/network-scripts/ifcfg-eth0) 2>/dev/null ; echo

BASE_SERVICES="ssh sshd chrony chronyd cron crond rsyslog"

for i in ${BASE_SERVICES} ; do
  echo -e "\e[31mService '${i}' is \e[m `systemctl is-enabled "${i}" 2>/dev/null`"
  echo -e "\e[31mService '${i}' is \e[m `systemctl is-active "${i}" 2>/dev/null`"
done

echo

echo -e "\e[31mOpen ports\e[m\n`netstat -pltun`\n"

echo -e "\e[31mTimezones\e[m\n`timedatectl`\n"
EndOfMessage

#####################################################################

find "${INIT_DIR}" -type f -exec chmod 644 {} \;
```

---
# System files

```bash
INIT_DIR="/etc"
INIT_PATH="${INIT_DIR}/issue"

chattr -i "${INIT_PATH}"
rm -fv "${INIT_PATH}"

cat > "${INIT_PATH}" << "EndOfMessage"
\l

EndOfMessage

chown root:root "${INIT_PATH}"
chmod 644 "${INIT_PATH}"

chattr +i "${INIT_PATH}"

#####################################################################

INIT_DIR="/etc"
INIT_PATH="${INIT_DIR}/resolv.conf"

chattr -i "${INIT_PATH}"
rm -fv "${INIT_PATH}"

cat > "${INIT_PATH}" << "EndOfMessage"
nameserver 10.10.10.10
nameserver 8.8.8.8
nameserver 8.8.4.4

options rotate timeout:1 attempts:2
EndOfMessage

chown root:root "${INIT_PATH}"
chmod 644 "${INIT_PATH}"

chattr +i "${INIT_PATH}"
```

---
# Network

```bash
export IP_ADDR="10.10.10.10"
export IP_NET="255.255.0.0"
export IP_GW="10.10.0.101"

export NEW_TIMEZONE="Europe/Warsaw"
export NEW_HOSTNAME="hostname"
export NEW_DOMAIN="localdomain"

hostnamectl set-hostname "${NEW_HOSTNAME}"
timedatectl set-timezone "${NEW_TIMEZONE}"

UBUNTU_NET_DIR="/etc/network/interfaces.d"
RHEL_NET_DIR="/etc/sysconfig/network-scripts"

cat > "/etc/hosts" << EndOfMessage
127.0.0.1  localhost.localdomain  localhost

${IP_ADDR}  ${NEW_HOSTNAME}.${NEW_DOMAIN}  ${NEW_HOSTNAME}
EndOfMessage

if [ -d "${UBUNTU_NET_DIR}" ] ;then
cat > "${UBUNTU_NET_DIR}/eth0.conf" << EndOfMessage
auto eth0
allow-hotplug eth0
iface eth0 inet static
        address ${IP_ADDR}
        netmask ${IP_NET}
        gateway ${IP_GW}
EndOfMessage
elif [ -d "${RHEL_NET_DIR}" ] ; then
cat > "${RHEL_NET_DIR}/ifcfg-eth0" << EndOfMessage
TYPE=Ethernet
NAME=eth0
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
DEFROUTE=yes
PEERDNS=no
PEERROUTE=no
IPADDR=${IP_ADDR}
NETMASK=${IP_NET}
GATEWAY=${IP_GW}
EndOfMessage
fi
```

---


# Zeroing disk

```bash
rm -f zero

dd if=/dev/zero of=zero

rm -f zero
```

---

# Shirk vmware vmdk

```bash
vmware-vdiskmanager -k disk.vmdk
```
