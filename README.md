`Prepared for:`
 - Ubuntu 18/20
 - Debian 10 (`probably works on 8/9)
 - RHEL Family 7/8

```bash
#!/bin/bash

BASE_DIR="/root/scripts"
BASE_NAME="init.sh"

mkdir -p "${BASE_DIR}"

cat > "${BASE_DIR}/${BASE_NAME}" << "EndOfMessage"
#!/bin/bash

(apt-get update || yum check-update) 2>/dev/null

(apt-get install lsb-release -y || yum install redhat-lsb-core -y) 2>/dev/null

DISTRO_NAME="`(lsb_release -a 2>/dev/null | grep -Po "Distributor ID:\t\K[a-zA-Z]*")`"

if [ "${DISTRO_NAME}" == "Ubuntu" ] || [ "${DISTRO_NAME}" == "Debian" ] ; then
  dpkg-reconfigure openssh-server
elif [ "${DISTRO_NAME}" == "OracleServer" ] ; then
  ssh-keygen -q -N '' -t rsa -f /etc/ssh/ssh_host_rsa_key
  ssh-keygen -q -N '' -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
  ssh-keygen -q -N '' -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
fi

(systemctl enable ssh || systemctl enable sshd) 2>/dev/null
(systemctl restart ssh || systemctl restart sshd) 2>/dev/null
EndOfMessage

chmod 700 "${BASE_DIR}"

chmod 600 "${BASE_DIR}/${BASE_NAME}"
chown root:root -R "${BASE_DIR}/${BASE_NAME}"
```
---

```bash
#!/bin/bash

BASE_DIR="/root/scripts"
BASE_NAME="clean.sh"

mkdir -p "${BASE_DIR}"

cat > "${BASE_DIR}/${BASE_NAME}" << "EndOfMessage"
#!/bin/bash

(systemctl disable ssh || systemctl disable sshd) 2>/dev/null
(systemctl stop ssh || systemctl stop sshd) 2>/dev/null
rm -rfv /etc/ssh/ssh_host_*

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
