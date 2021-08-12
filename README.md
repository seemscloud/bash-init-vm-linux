## Prepare linux types:
 - Ubuntu 18 / 20
 - Debian 10 (`probably works on 8/9`)
 - RHEL Family 7 / 8

---

```bash
find /root/ -mindepth 1 -maxdepth 1 -exec rm -rf {} \;

#####################################################################

BASE_DIR="/root/scripts"

mkdir -p "${BASE_DIR}"

cat > "${BASE_DIR}/init.sh" << "EndOfMessage"
#!/bin/bash

(apt-get update || yum check-update) 2>/dev/null

(apt-get install chrony postfix lsb-release cron rsyslog git -y || 
yum install chrony postfix redhat-lsb-core rsyslog cronie git -y) 2>/dev/nullsystem

DISTRO_NAME="`(lsb_release -a 2>/dev/null | grep -Po "Distributor ID:\t\K[a-zA-Z]*")`"

if [ "${DISTRO_NAME}" == "Ubuntu" ] || [ "${DISTRO_NAME}" == "Debian" ] ; then
  dpkg-reconfigure openssh-server
elif [ "${DISTRO_NAME}" == "OracleServer" ] ; then
  ssh-keygen -q -N '' -t rsa -f /etc/ssh/ssh_host_rsa_key
  ssh-keygen -q -N '' -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
  ssh-keygen -q -N '' -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
fi

BASE_SERVICES="ssh sshd postfix chrony chronyd cron crond rsyslog"

for i in ${BASE_SERVICES} ; do 
  systemctl enable "${i}" 2>/dev/null
  systemctl start "${i}" 2>/dev/null
done
 
EndOfMessage

#####################################################################

cat > "${BASE_DIR}/clean.sh" << "EndOfMessage"
#!/bin/bash

rm -rfv /etc/ssh/ssh_host_*

BASE_SERVICES="ssh sshd postfix chrony chronyd cron crond rsyslog"

for i in ${BASE_SERVICES} ; do 
  systemctl disable "${i}" 2>/dev/null
  systemctl stop "${i}" 2>/dev/null
done

(apt-get update &&
apt-get -y upgrade &&
apt-get -y dist-upgrade ||
yum update -y) 2>/dev/null

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

chmod 700 "${BASE_DIR}"

find "${BASE_DIR}" -mindepth 1 -maxdepth 1 -type f -exec chown root:root {} \;
find "${BASE_DIR}" -mindepth 1 -maxdepth 1 -type f -exec chmod 600 {} \;

#####################################################################

BASE_DIR="/root/cron.d"

mkdir -p "${BASE_DIR}/system"

BASE_NAME="${flush.sh}"

cat > "${BASE_DIR}/system/${BASE_NAME}" << "EndOfMessage"
#!/bin/bash

sync; echo 1 > /proc/sys/vm/drop_caches > /dev/null
sync; echo 2 > /proc/sys/vm/drop_caches > /dev/null
sync; echo 3 > /proc/sys/vm/drop_caches > /dev/null
EndOfMessage

find "${BASE_DIR}" -exec chown root:root {} \;
find "${BASE_DIR}" -type d -exec chmod 700 {} \;

chmod 600 "${BASE_DIR}/system/${BASE_NAME}"

BASE_NAME="crontab.file"

cat > "${BASE_NAME}" << "EndOfMessage"
# Flush Memory
*/1 * * * * /bin/bash /root/cron.d/system/flush.sh
EndOfMessage

crontab "${BASE_NAME}"

rm -f "${BASE_NAME}"

#####################################################################

git init .
git remote add origin https://github.com/theanotherwise/dotfiles.git
git fetch --all
rm -f .bashrc .bash_profile .vimrc .gitignore .dotfiles

git checkout master

rm -f README.md .gitignore

#####################################################################

BASE_NAME="/etc/resolv.conf"

chattr -i "${BASE_NAME}"
rm -f "${BASE_NAME}"

cat > "${BASE_NAME}" << "EndOfMessage"
nameserver 10.10.10.10

nameserver 8.8.8.8
nameserver 8.8.4.4

options rotate
options timeout:1
options attempts:2
EndOfMessage

chown root:root "${BASE_NAME}"
chmod 644 "${BASE_NAME}"

chattr +i "${BASE_NAME}"
```

---

`network`

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

`checks`

```bash
REPORT_NAME="report.txt"
>"${REPORT_NAME}"

hostname >>"${REPORT_NAME}" 2>&1
hostname -f >>"${REPORT_NAME}" 2>&1

lsb_release -a >>"${REPORT_NAME}" 2>&1
uname -a >>"${REPORT_NAME}" 2>&1

cat /etc/hosts /etc/resolv.conf >>"${REPORT_NAME}" 2>&1

cat /etc/network/interfaces.d/eth0.conf /etc/sysconfig/network-scripts/ifcfg-eth0 2>/dev/null >>"${REPORT_NAME}"

BASE_SERVICES="ssh sshd postfix chrony chronyd cron crond rsyslog"

for i in ${BASE_SERVICES} ; do
systemctl is-enabled "${i}" 2>/dev/null >>"${REPORT_NAME}"
done

netstat -pltun >>"${REPORT_NAME}" 2>&1
timedatectl >>"${REPORT_NAME}" 2>&1

clear ; clear
cat "${REPORT_NAME}"
rm -f "${REPORT_NAME}"
```

---

`zeroing disk`

```bash
rm -f zero

dd if=/dev/zero of=zero

rm -f zero
```

---

`shirk vmware vmdk`

```bash
vmware-vdiskmanager -k disk.vmdk
```
