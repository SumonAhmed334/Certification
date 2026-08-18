# RHCSA Practice Exam — Node A (servera)

## VM Setup Info
- **Password:** redhat
- **IP:** 172.25.250.10/24
- **Gateway:** 172.25.250.254
- **DNS:** 172.25.250.220
- **NB:** All partitions should be created on `/dev/vdb`

---

## Password Reset (Grub / Rescue)

```bash
# Configure Grub
vim /etc/default/grub
grub2-mkconfig -o /boot/grub2/grub.cfg
```

**Method 1**
```
rd.break rw console=tty1
chroot sysroot
passwd
touch /.autorelabel
reboot -f   # or ctrl+d, or /usr/sbin/reboot -f
```

**Method 2**
```
init=/bin/bash rw
passwd
touch /.autorelabel
reboot -f   # or ctrl+d, or /usr/sbin/reboot -f
```

---

## Network Configuration (Add IP)

```bash
nmcli connection add con-name jubaer type ethernet ifname eth0 \
  ipv4.method manual ipv4.addresses 172.25.250.10/24 \
  ipv4.gateway 172.25.250.254 \
  ipv4.dns 172.25.250.220,8.8.8.8 \
  connection.autoconnect yes

systemctl restart NetworkManager

# Verify DNS got applied
cat /etc/resolv.conf

# If DNS didn't get applied, check if file is immutable
lsattr /etc/resolv.conf
chattr -i /etc/resolv.conf
systemctl restart NetworkManager
```

---

## 01. Set the Hostname

```bash
hostnamectl hostname nodea.lab.example.com
# or
nmcli general hostname nodea.lab.example.com
```
(Both commands work.)

---

## 02. Yum Repository Configuration

```bash
cd /etc/yum.repos.d
rm -fr *          # remove previous repo files if needed
vim anyName.repo
```

**anyName.repo contents:**
```ini
[appstream]
name=AppStream
baseurl=http://172.25.254.254/content/rhel9.0/x86_64/dvd/AppStream/
gpgcheck=0
enabled=1

[baseOS]
name=BaseOS
baseurl=http://172.25.254.254/content/rhel9.0/x86_64/dvd/BaseOS/
gpgcheck=0
enabled=1
```

```bash
yum clean all
yum repolist all
# Try downloading a package to confirm it works
```

---

## 03. Cron Jobs

Create users first if they don't exist, and set a password.

**User natasha** — daily at 14:23, runs `/bin/echo "hi alex"`:
```
23  14  *  *  * natasha  /bin/echo "hi alex"
```

**User harry** — every 3 minutes, runs `/bin/echo I got RHCE certificate.`:
```
*/3  *  *  *  * harry  /bin/echo I got RHCE certificate.
```

To set cron jobs for other users from root:
```bash
vim /etc/crontab
```
```
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
# For details see man 4 crontabs
# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
23  14  *  *  * natasha  /bin/echo "hi alex"
*/3  *  *  *  * harry  /bin/echo I got RHCE certificate.
```
(Other lines are already present in the default crontab file — only add the last two.)

---

## 04. Debug SELinux / HTTPD

```bash
vim /etc/httpd/conf/httpd.conf
```
- At line ~47, change `Listen` port to `82`.

```bash
# Create an index.html with any message
mv index.html /var/www/html/
```

**Fix SELinux (port + file context), then restore:**
```bash
semanage port -a -t http_port_t -p tcp 82
semanage fcontext -a -t httpd_sys_content_t /var/www/html/index.html
restorecon /var/www/html/index.html
```
Ensure SELinux is in **Enforcing** mode (`getenforce` / `setenforce 1`) and the firewall allows port 82 if required.

---

## 05. Users, Groups, and Group Memberships

```bash
groupadd sysadmin
useradd natasha -G sysadmin
useradd sarah -G sysadmin
getent group sysadmin          # verify natasha & sarah are in sysadmin

useradd harry -s /sbin/nologin
cat /etc/passwd | grep harry   # verify harry has no interactive shell

# Set password 'password' for all three
echo password | passwd --stdin natasha
echo password | passwd --stdin sarah
echo password | passwd --stdin harry
```

---

## 06. Collaborative Directory `/common/admin`

```bash
mkdir /common/admin/ -p
chown :sysadmin /common/admin/
chmod g+rwxs /common/admin/
```
- `g+s` (setgid) ensures new files created inside inherit the `sysadmin` group.
- Directory permissions restrict access to group members only (adjust `chmod 2770` if "no other users" access is strictly required).

---

## 07. Copy `/etc/passwd` to `/var/tmp` with ACLs

```bash
cp /etc/passwd /var/tmp/

setfacl -m u:harry:rw /var/tmp/passwd    # harry can read & write
setfacl -m u:sarah:--- /var/tmp/passwd   # sarah has no access (--- = 000)

getfacl /var/tmp/passwd
```

Expected output:
```
# file: var/tmp/passwd
# owner: root
# group: root
user::rw-
user:harry:rw-
user:sarah:---
group::r--
mask::rw-
other::r--
```
This satisfies: owned by root, group root, not executable, harry rw, sarah no access, others can read.

---

## 08. Synchronise Time with classroom.example.com

```bash
yum install chrony
systemctl start chronyd.service

vim /etc/chrony.conf
# Add line:
server classroom.example.com iburst

systemctl restart chronyd.service

chronyc tracking     # Leap status should be "Normal"
timedatectl           # "NTP service: active" should be shown
```

---

## 09. Configure AutoFS

**Server side (workstation.lab.example.com / 172.25.250.9):**
```bash
yum install nfs-utils
systemctl enable nfs-server.service --now

# /etc/exports
/home/guests    172.25.250.10/24(rw,sync)

systemctl restart nfs-server.service
exportfs

firewall-cmd --add-service={nfs,rpc-bind,mountd} --permanent
firewall-cmd --reload
```

**Client side (nodea):**
```bash
yum install autofs
systemctl enable autofs.service --now

showmount -e 172.25.250.9

rpm -qc autofs
vim /etc/auto.master
```
Add to `/etc/auto.master`:
```
/home/guests    /etc/auto.misc
```

Edit `/etc/auto.misc`:
```
*               -rw,sync                172.25.250.9:/home/guests/&
remote10        -ro,sync                172.25.250.9:/home/guests/remote10
```
> Note: Task requires **remote5** to have read/write access and **remote10** to have read-only access — the wildcard `*` entry with `-rw,sync` covers remote5 (and any other user), while the explicit `remote10` line overrides it with `-ro,sync`.

```bash
systemctl restart autofs
```

---

## 10. Create Backup Archives of `/etc`

```bash
tar -cvjf /home/backup.tar.bz2 /etc
tar -cvzf /home/backup.tar.gz /etc

cd /home
file *      # verify archive files were created properly
```

---

## 11. Deny Cron Job for User susan

```bash
vim /etc/cron.deny
# Add the line:
susan
```

---

## 12. Find All Files Owned by User `brain`

```bash
# Create user brain first if not already present
useradd brain

mkdir -p /root/brain

find / -user brain -exec cp -fvpr {} /root/brain/ \;
```

---

## 13. Download word.dict and Filter Lines Containing "mail"

```bash
wget http://172.25.254.254/content/word.dict -O /root/word.dict
# -O and the path aren't necessary if run from /root directory

grep mail /root/word.dict >> /root/sorted.dict
```

---

## Quick Checklist

| # | Task | Status |
|---|------|--------|
| 1 | Set hostname | ☐ |
| 2 | Yum repo config | ☐ |
| 3 | Cron jobs (natasha, harry) | ☐ |
| 4 | SELinux / httpd on port 82 | ☐ |
| 5 | Users & groups (sysadmin) | ☐ |
| 6 | `/common/admin` collaborative dir | ☐ |
| 7 | `/var/tmp/passwd` ACLs | ☐ |
| 8 | Time sync with classroom.example.com | ☐ |
| 9 | AutoFS for remote5/remote10 | ☐ |
| 10 | Backup `/etc` as tar.bz2 & tar.gz | ☐ |
| 11 | Deny cron for susan | ☐ |
| 12 | Find files owned by brain | ☐ |
| 13 | Download word.dict & grep "mail" | ☐ |
