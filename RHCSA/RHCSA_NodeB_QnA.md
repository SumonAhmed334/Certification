# RHCSA Practice Exam — Node B (serverb)

## VM Setup Info
- **Password:** TombigSmall
- **IP:** 172.25.250.11/24
- **Gateway:** 172.25.250.254
- **DNS:** 172.25.250.220

---

## 01. Set the Hostname

```bash
hostnamectl hostname nodeb.lab.example.com
```

---

## 02. Yum Repository Configuration

Packages available at:
- url1: `http://172.25.254.254/content/rhel9.0/x86_64/dvd/AppStream/`
- url2: `http://172.25.254.254/content/rhel9.0/x86_64/dvd/BaseOS/`

```bash
cd /etc/yum.repos.d
vim anyName.repo
```
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
```

---

## 03. Set a Recommended Tuning Profile

```bash
# Check if tuned is installed; if not, install it
yum install tuned

# Check the recommended profile for the system
tuned-adm recommend

# Check the currently active profile
tuned-adm active

# Set the recommended profile (should already match in most cases)
tuned-adm profile <profile_name>

# Example:
tuned-adm profile virtual-guest
```

---

## 04. Create a 250MB SWAP Partition (Persistent)

**Step 1 — Create a new partition:**
```bash
fdisk /dev/<device_name>
# ex: fdisk /dev/nvme0n3
```
Inside `fdisk`:
1. `g` — change partition table to GPT
2. `n` — create new partition
   - Partition number: any (e.g., 1)
   - First sector: leave blank/default (don't add extra space here)
   - Last sector: `+250M`
3. `t` — change filesystem type
   - `l` or `L` — list available filesystem types
   - `q` — exit the list
   - `19` — Linux swap (code may vary by system)
4. `p` — print/verify the new partition info
5. `w` — write and save (or `q` to exit without saving if something's wrong)

**Step 2 — Format as swap:**
```bash
mkswap <device/partition_name>
# ex: mkswap /dev/nvme0n3p1
```

**Step 3 — Enable swap (temporary):**
```bash
swapon /dev/nvme0n3p1
```

**Step 4 — Make it permanent** by adding an entry to `/etc/fstab`:
```
UUID=1af6d00a-0592-4762-933f-067b8549577c   swap   swap   defaults   0   0
```
> You can use either the `UUID=` form or the device path (e.g., `/dev/nvme0n3p1`) in the first column.

---

## 05. Create Volume Group `myvolume` (8MiB PE) and LV `mydatabase` (100 PE, ext4)

**Step 1 — Create a partition (~800MiB+) for the LVM:**
Use the same `fdisk` steps as above, sized around 850MiB.

**Step 2 — Create the physical volume:**
```bash
pvcreate <device/partition_name>
# ex: pvcreate /dev/nvme0n3p2

pvdisplay   # verify
```
> Note: creating a PV manually isn't strictly necessary — `vgcreate` will create it automatically if needed.

**Step 3 — Create the volume group with 8MiB PE size:**
```bash
vgcreate myvolume /dev/nvme0n3p2 -s 8MiB
vgdisplay
```

**Step 4 — Create the logical volume with 100 extents:**
```bash
lvcreate -n mydatabase -l 100 myvolume
```

**Step 5 — Format with ext4:**
```bash
mkfs.ext4 /dev/myvolume/mydatabase
# or
mkfs.ext4 /dev/mapper/myvolume-mydatabase
```

**Step 6 — Create mount point and mount:**
```bash
mkdir /database
mount /dev/mapper/myvolume-mydatabase /database/
# or
mount /dev/myvolume/mydatabase /database/
```
> Don't forget to add an entry in `/etc/fstab` for a **permanent** mount, as required by the task.

---

## 06. Resize LVM `/dev/myvolume/mydatabase` to 500MiB

```bash
# Unmount first
umount /database

# Resize both filesystem and LV in one step
lvresize -r -L 500M /dev/myvolume/mydatabase

# Remount
mount /dev/myvolume/mydatabase /database/
```
Target size range: **~450MiB to 550MiB**.

---

## 07. Custom Function `pandora` for User Output

**Goal:** running `pandora` as user "pandora" shows: `Labla lbal lahs ksbhs`

```bash
vim /etc/bashrc
```
Add:
```bash
pandora(){
        echo "Labla lbal lahs ksbhs"
};
```
```bash
source /etc/bashrc
pandora
```

---

## 08. Customize User Environment — `starton` Command

**Goal:** create a command `starton` that runs:
`ps -eo pid,tid,class,rtprio,ni,pri,psr,pcpu,stat,comm`

Add to `/etc/bashrc`:
```bash
starton(){
        (ps -eo pid,tid,class,rtprio,ni,pri,psr,pcpu,stat,comm)
};
```
```bash
source /etc/bashrc
starton
```
> Note: For tasks 07 and 08 you can alternatively use `alias` instead of a shell function.

---

## 09. Set Password Max Age for New Users Only

**Goal:** all *new* users get a max password age of 30 days; existing users keep default.

```bash
vim /etc/login.defs
```
Find and change (around line 131):
```
PASS_MAX_DAYS   30
```
> This only affects newly created users going forward — it does not retroactively change existing users' password aging (which is stored per-user in `/etc/shadow`).

---

# Container Tasks

## 1. Build Image `pdfconvert` and Run Container `monitor`

**Build from Containerfile:**
```bash
wget https://content.example.com/container/Containerfile

vim Containerfile
```
```
FROM docker.io/openviewdev/pdfconverter
```
```bash
podman build . -t pdfconvert
podman images
```

**Run container named `monitor` from the new image:**
```bash
podman run -dit --name monitor localhost/pdfconvert:latest
podman ps
```

**Verify by entering the container:**
```bash
podman exec -it monitor /bin/bash
# bash-4.3# exit
```

---

## 2. Create Container `pdfconverter` with Volumes + Systemd Service

**Enable linger for the user (so user services run without an active login session):**
```bash
loginctl enable-linger devuser1
loginctl show-user devuser1
```

**Prepare host directories and SELinux context:**
```bash
mkdir /opt/input/ /opt/processed/

semanage fcontext -a -t container_file_t "/opt/processed(/.*)?"
semanage fcontext -a -t container_file_t "/opt/input(/.*)?"
restorecon -Rv /opt/

setfacl -m u:devuser1:rwx /opt/input/
setfacl -m u:devuser1:rwx /opt/processed/
```

**Run the container as `devuser1`, mapping volumes:**
- `/opt/input/` → `/action/incoming/`
- `/opt/processed/` → `/action/outgoing/`

```bash
podman run -dit --name pdfconverter \
  -v /opt/input/:/action/incoming/ \
  -v /opt/processed/:/action/outgoing/ \
  localhost/pdfconvert:latest

podman ps
```

**Test the mapping:**
```bash
podman exec -it pdfconverter /bin/bash
# bash-4.3# cat /action/incoming/input.txt
# bash-4.3# echo "test out" > /action/outgoing/out.txt
```

From the host (as devuser1):
```bash
echo input > /opt/input/input.txt
cat /opt/processed/out.txt
```

**Create a user-level systemd service for the container:**
```bash
mkdir -p /home/devuser1/.config/systemd/user
cd /home/devuser1/.config/systemd/user/

podman generate systemd pdfconverter -f -n
# creates: /home/devuser1/.config/systemd/user/container-pdfconverter.service
```

**Reload and enable the service (as devuser1, e.g. via SSH):**
```bash
ssh devuser1@stream8-clone -X

systemctl --user daemon-reload
systemctl --user enable --now container-pdfconverter.service
systemctl --user status container-pdfconverter.service
```
> `enable-linger` (done earlier) ensures this user service can start automatically at boot even without an active login session — this satisfies the "run automatically at system boot" requirement.

---

## Quick Checklist

| # | Task | Status |
|---|------|--------|
| 1 | Set hostname | ☐ |
| 2 | Yum repo config | ☐ |
| 3 | Recommended tuned profile | ☐ |
| 4 | 250MB SWAP partition (persistent) | ☐ |
| 5 | VG `myvolume` + LV `mydatabase` (ext4, mounted) | ☐ |
| 6 | Resize LV to 500MiB | ☐ |
| 7 | `pandora` function/alias | ☐ |
| 8 | `starton` command | ☐ |
| 9 | PASS_MAX_DAYS 30 for new users | ☐ |
| C1 | Build `pdfconvert` image, run `monitor` container | ☐ |
| C2 | `pdfconverter` container with volumes + boot-enabled service | ☐ |
