`Prepared for:`
 - Ubuntu 18/20
 - Debian 10 (`probably works on 8/9`)
 - RHEL Family 7/8

`Network prepared with eth0 interface for`:
 - networking (Debian/Ubuntu)
 - network (RHEL Family)

---

`init.sh` / `clean.sh`

```bash
BASE_DIR="/root/scripts"
BASE_NAME="init.sh"

mkdir -p "${BASE_DIR}"

cat > "${BASE_DIR}/${BASE_NAME}" << "EndOfMessage"
#!/bin/bash

(apt-get update || yum check-update) 2>/dev/null

(apt-get install chrony postfix lsb-release cron rsyslog -y || 
yum install chrony postfix redhat-lsb-core rsyslog cronie -y) 2>/dev/nullsystem

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

chmod 700 "${BASE_DIR}"

chmod 600 "${BASE_DIR}/${BASE_NAME}"
chown root:root -R "${BASE_DIR}/${BASE_NAME}"

BASE_NAME="clean.sh"

mkdir -p "${BASE_DIR}"

cat > "${BASE_DIR}/${BASE_NAME}" << "EndOfMessage"
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

chmod 600 "${BASE_DIR}/${BASE_NAME}"
chown root:root -R "${BASE_DIR}/${BASE_NAME}"
```

---

`resolv.conf`

```bash
BASE_NAME="/etc/resolv.conf"

chattr -i "${BASE_NAME}"
rm -rf "${BASE_NAME}"

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

clear
cat "${REPORT_NAME}"
rm -f "${REPORT_NAME}"
```
