# RHCE (EX294) Ansible Exam — Questions & Solutions

---

## 01. Install and Configure Ansible

**Requirements:**
- Install Ansible and required packages on the control node `workstation`.
- Create a static inventory file `/home/devops/ansible/inventory` so that:
  - `servera` is a member of the **dev** host group
  - `serverb` is a member of the **test** host group
  - `serverc` and `serverd` are members of the **prod** host group
  - `bastion` is a member of the **balancers** host group
  - The **prod** group is a member of the **webservers** host group
- Create a configuration file `/home/devops/ansible/ansible.cfg` where:
  - The host inventory file is `/home/devops/ansible/inventory`
  - The location of roles used in playbooks includes `/home/devops/ansible/roles`

### Solution

```bash
mkdir ansible
cd ansible/
pwd
# /home/devops/ansible
```

**inventory:**
```ini
[dev]
servera

[test]
serverb

[prod]
serverc
serverd

[balancers]
bastion

[webservers:children]
prod
```

```bash
mkdir roles
cat /etc/ansible/ansible.cfg | grep role
# roles_path = /etc/ansible/roles
vim ansible.cfg
```

**ansible.cfg:**
```ini
[defaults]
inventory=/home/devops/ansible/inventory
remote_user=devops
ask_pass=false
roles_path=/etc/ansible/roles:/usr/share/ansible/roles:/home/devops/.ansible/roles:/home/devops/ansible/roles

[privilege_escalation]
become=true
become_method=sudo
become_user=root
become_ask_pass=false
```

**Verify:**
```bash
ansible all --list-hosts
# hosts (5): servera serverb bastion serverc serverd

ansible all -m ping
```

---

## 02. Create and Run Ansible Ad-Hoc Commands

**Requirements:**
Create a shell script `/home/devops/ansible/adhoc.sh` that uses ad-hoc commands to create a yum repository on each managed node:

**Repository 1:**
- Name: `EX294_BASE`
- Description: `EX294 base software`
- Base URL: `http://content.example.com/rhel8.0/x86_64/dvd/BaseOS`
- GPG checking enabled
- GPG key URL: `http://content.example.com/rhel8.0/x86_64/dvd/RPM-GPG-KEY-redhat-release`
- Repository enabled

**Repository 2:**
- Name: `EX294_STREM`
- Description: `EX294 stream software`
- Base URL: `http://content.example.com/rhel8.0/x86_64/dvd/AppStream`
- GPG checking enabled
- Same GPG key URL
- Repository enabled

### Solution

**adhoc.sh:**
```bash
#!/bin/bash
# REPO-01
ansible all -m yum_repository -a "name='EX294_BASE' description='EX294 base software' baseurl='http://content.example.com/rhel8.0/x86_64/dvd/BaseOS' enabled=yes gpgkey='http://content.example.com/rhel8.0/x86_64/dvd/RPM-GPG-KEY-redhat-release' gpgcheck=yes"

# REPO-02
ansible all -m yum_repository -a "name='EX294_STREM' description='EX294 stream software' baseurl='http://content.example.com/rhel8.0/x86_64/dvd/AppStream' enabled=yes gpgkey='http://content.example.com/rhel8.0/x86_64/dvd/RPM-GPG-KEY-redhat-release' gpgcheck=yes"
```

```bash
chmod +x adhoc.sh
source adhoc.sh
ansible all -a "yum repolist all"
```

**Verification:**
```bash
ansible all -m yum -a "name=httpd state=present"
ansible all -a "rpm -qa httpd"
```

---

## 03. Install Packages

**Requirements:**
Create a playbook `/home/devops/ansible/packages.yml` that:
- Installs `php` & `mariadb` on hosts in the **dev**, **test**, and **prod** host groups
- Installs the "Development Tools" package group on hosts in the **dev** host group
- Updates all packages to the latest version on hosts in the **dev** host group

### Solution

**packages.yml:**
```yaml
---
- name: Configure php & mariadb & Development Tools & Update all Packages
  hosts: dev,test,prod
  tasks:
    - name: Installs the php & mariadb
      yum:
        name:
          - php
          - mariadb
        state: present

    - name: Installs the Development Tools
      yum:
        name: "@Development Tools"
        state: present
      when: "'dev' in group_names"

    - name: Update all packages to the latest version
      yum:
        name: "*"
        state: latest
      when: "'dev' in group_names"
```

```bash
ansible-playbook packages.yml --syntax-check
ansible-playbook packages.yml
```

---

## 04. Modify File Content

**Requirements:**
Create a playbook `/home/student/ansible/issue.yml` that:
- Runs on all inventory hosts
- Replaces the content of `/etc/issue` with a single line of text:
  - **dev** group → `Development`
  - **test** group → `Test`
  - **prod** group → `Production`

### Solution

**issue.yml:**
```yaml
---
- name: Modify File Content
  hosts: dev,test,prod
  tasks:
    - name: Updating issue File on dev host group
      copy:
        content: "Development"
        dest: /etc/issue
      when: "'dev' in group_names"

    - name: Updating issue File on test host group
      copy:
        content: "Test"
        dest: /etc/issue
      when: "'test' in group_names"

    - name: Updating issue File on prod host group
      copy:
        content: "Production"
        dest: /etc/issue
      when: "'prod' in group_names"
```

```bash
ansible-playbook issue.yml --syntax-check
ansible-playbook issue.yml -C     # check/dry-run
ansible-playbook issue.yml
```

---

## 05. Create a Web Content Directory

**Requirements:**
Create a playbook `/home/student/ansible/webcontent.yml` that:
- Runs on managed nodes in the **dev** host group
- Creates directory `/webdev`:
  - owned by group `webdev`
  - permissions: owner=rwx, group=rwx, other=rx
  - special permission: set GID
- Symbolically links `/var/www/html/webdev` → `/webdev`
- Creates `/web/index.html` with the text `Development`
- Browsing `http://servera.lab.example.com/webdev/` should output: `Development`

### Solution

**webcontent.yml:**
```yaml
---
- name: Manage Web Content Directory
  hosts: dev
  tasks:
    - name: Install httpd
      yum:
        name: httpd
        state: present

    - name: Start and Enable httpd Service
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Configure firewalld
      firewalld:
        service: http
        permanent: yes
        immediate: yes
        state: enabled

    - name: Create webdev group
      group:
        name: webdev
        state: present

    - name: Create a directory if it does not exist
      file:
        path: /webdev
        state: directory
        mode: '2775'
        group: webdev
        setype: httpd_sys_content_t

    - name: Symbolically link /var/www/html/webdev to /webdev
      file:
        src: /webdev
        dest: /var/www/html/webdev
        state: link

    - name: Create Content
      copy:
        content: "Development"
        dest: /webdev/index.html
        setype: httpd_sys_content_t
```

> **Note:** If the index.html content is provided as a downloadable link instead of literal text, use the `get_url` module instead of `copy`:
> ```yaml
> - name: Download index file
>   get_url:
>     url: url
>     dest: "/webdev/index.html"
>     setype: "httpd_sys_content_t"
> ```

```bash
ansible-playbook webcontent.yml --syntax-check
ansible-playbook webcontent.yml
ansible servera -a "curl http://servera/webdev/index.html"
```

---

## 06. Create a Cron Job

**Requirements:**
Create `/home/devops/ansible/cron.yml` that:
- Runs on all managed nodes
- Configures a cron job that runs every 2 minutes, executes `logger "EX294 exam in progress"`, running as user `natasha`

### Solution

**cron.yml:**
```yaml
---
- name: Configure User and Crontab for Natasha
  hosts: all
  tasks:
    - name: Create User Natasha
      user:
        name: natasha
        state: present

    - name: Perform Cronjob for User Natasha
      cron:
        user: natasha
        minute: "*/2"
        job: logger "EX294 exam in progress"
```

```bash
ansible-playbook cron.yml --syntax-check
ansible-playbook cron.yml -C
ansible-playbook cron.yml
ansible servera -a "crontab -l -u natasha"
# servera | CHANGED | rc=0 >>
# */2 * * * * logger "EX294 exam in progress"
```

---

## 07. Create a Password Vault

**Requirements:**
Create an Ansible vault `/home/devops/ansible/locker.yml` containing:
- `PW_developer` = `Imadev`
- `PW_manager` = `Imamgr`
- Vault password: `whenyouwishuponastar`, stored in `/home/devops/ansible/secret.txt`

### Solution

```bash
vim secret.txt
# whenyouwishuponastar

chmod 600 secret.txt

ansible-vault create locker.yml --vault-password-file secret.txt
```
Inside the vault file:
```yaml
PW_developer: Imadev
PW_manager: Imamgr
```

```bash
ansible-vault view locker.yml --vault-password-file secret.txt
```

---

## 08. Create User Accounts

**Requirements:**
- Download the user list from `http://content.example.com/EX294_Ansible/Ansible_files/user_list.yml`
- Using the vault from Q07, create `/home/devops/ansible/users.yml` that:
  - Users with job = **developer**:
    - created on managed nodes in **dev** and **test** groups
    - password from `PW_developer`
    - member of secondary group `devops`
  - Users with job = **manager**:
    - created on managed nodes in **prod** group
    - password from `PW_manager`
    - member of secondary group `opsmgr`
  - Passwords use SHA512 hash format and expire after a max of 30 days
  - The playbook must work using the vault password file `/home/devops/ansible/secret.txt`

### Solution

Add to `ansible.cfg` defaults section:
```ini
vault_password_file=/home/devops/ansible/secret.txt
```

```bash
wget http://content.example.com/EX294_Ansible/Ansible_files/user_list.yml
vim users.yml
```

**users.yml:**
```yaml
---
- name: Configure user accounts for developer
  hosts: dev,test
  vars_files:
    - locker.yml
    - user_list.yml
  tasks:
    - name: Create group devops
      group:
        name: devops
        state: present

    - name: Create user for developer
      user:
        name: "{{ item.name }}"
        state: present
        groups: devops
        password: "{{ PW_developer | password_hash('sha512') }}"
      when: item.job == "developer"
      loop: "{{ users }}"

    - name: Password will expire maximum 30 days for developer
      shell: "chage -M 30 {{ item.name }}"
      when: item.job == "developer"
      loop: "{{ users }}"

- name: Create user accounts for Manager
  hosts: prod
  vars_files:
    - locker.yml
    - user_list.yml
  tasks:
    - name: Create group opsmgr
      group:
        name: opsmgr
        state: present

    - name: Create user for manager
      user:
        name: "{{ item.name }}"
        state: present
        groups: opsmgr
        password: "{{ PW_manager | password_hash('sha512') }}"
      when: item.job == "manager"
      loop: "{{ users }}"

    - name: Password will expire maximum 30 days
      shell: "chage -M 30 {{ item.name }}"
      when: item.job == "manager"
      loop: "{{ users }}"
```

```bash
ansible-playbook users.yml
```

**Verification:**
```bash
ansible dev -a "id bob"
ansible prod -a "id sally"
```

---

## 09. Rekey an Ansible Vault

**Requirements:**
- Download the vault from `http://content.example.com/EX294_Ansible/Ansible_files/salaries.yml` to `/home/devops/ansible/`
- Current vault password: `insecure4sure`
- New vault password: `bbe2de98389b`
- The vault must remain encrypted with the new password

### Solution

```bash
wget http://content.example.com/EX294_Ansible/Ansible_files/salaries.yml

ansible-vault rekey salaries.yml --ask-vault-pass
# Vault password: insecure4sure
# New Vault password: bbe2de98389b
# Confirm New Vault password: bbe2de98389b
# Rekey successful

ansible-vault view salaries.yml --ask-vault-pass
# Vault password: bbe2de98389b
```

---

## 10. Generate a Hardware Report

**Requirements:**
Create `/home/devops/ansible/hwreport.yml` that produces `/root/hwreport.txt` on all managed nodes containing:
- Inventory hostname
- Total memory
- BIOS version
- Size of disk `vda`
- Size of disk `vdb`
- One `key=value` pair per line

Steps:
- Download `http://content.example.com/EX294_Ansible/ansible_files/hwreport.empty` and save as `/root/hwreport.txt`
- Modify the values in the file
- If a hardware item doesn't exist, its value should be `NONE`

### Solution

```bash
wget http://content.example.com/EX294_Ansible/ansible_files/hwreport.empty
cat hwreport.empty
```
```
hostname=inventoryhostname
memory=memory_in_MB
bios_version=BIOS_version
sda_size=disk_sda_size
sdb_size=disk_sdb_size
```

**hwreport.yml:**
```yaml
---
- name: Generate Hardware Report
  hosts: all
  tasks:
    - name: Copy the hwreport.empty file
      get_url:
        url: http://content.example.com/EX294_Ansible/ansible_files/hwreport.empty
        dest: /root/hwreport.txt

    - name: Information about inventory hostname
      replace:
        path: /root/hwreport.txt
        regexp: "inventoryhostname"
        replace: "{{ ansible_hostname | default('NONE') }}"

    - name: Information about memory
      replace:
        path: /root/hwreport.txt
        regexp: "memory_in_MB"
        replace: "{{ ansible_memtotal_mb | default('NONE') }}"

    - name: Information about BIOS
      replace:
        path: /root/hwreport.txt
        regexp: "BIOS_version"
        replace: "{{ ansible_bios_version | default('NONE') }}"

    - name: Information about sda
      replace:
        path: /root/hwreport.txt
        regexp: "disk_sda_size"
        replace: "{{ ansible_devices.sda.size | default('NONE') }}"

    - name: Information about sdb
      replace:
        path: /root/hwreport.txt
        regexp: "disk_sdb_size"
        replace: "{{ ansible_devices.sdb.size | default('NONE') }}"
```

```bash
ansible-playbook hwreport.yml --syntax-check
ansible-playbook hwreport.yml
ansible all -a "cat /root/hwreport.txt"
```

---

## 11. Generate a Host File

**Requirements:**
- Download the initial template from `http://content.example.com/EX294_Ansible/ansible_files/host.j2` to `/home/devops/ansible/`
- Complete the template so it generates a file with one line per host, matching `/etc/hosts` format
- Create `/home/devops/ansible/host.yml` that uses this template to generate `/etc/myhosts` on hosts in the **dev** host group

Expected output on `/etc/myhosts`:
```
127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain
::1 localhost localhost.localdomain localhost6 localhost6.localdomain
172.25.254.10 servera.lab.example.com servera
172.25.254.11 serverb.lab.example.com serverb
172.25.254.12 serverc.lab.example.com serverc
172.25.254.13 serverd.lab.example.com serverd
172.25.250.254 bastion.lab.example.com bastion
```
(Order of hosts is not important.)

### Solution

```bash
wget http://content.example.com/EX294_Ansible/ansible_files/host.j2
vim host.j2
```

**host.j2 (append):**
```jinja2
{% for i in groups.all %}
{{ hostvars[i].ansible_default_ipv4.address }} {{ hostvars[i].ansible_fqdn }} {{ hostvars[i].ansible_hostname }}
{% endfor %}
```

**host.yml:**
```yaml
---
- name: Generate Host File
  hosts: all
  tasks:
    - name: Copy j2 file through template module
      template:
        src: host.j2
        dest: /etc/myhosts
      when: "'dev' in group_names"
```

```bash
ansible-playbook host.yml --syntax-check
ansible-playbook host.yml
ansible servera -a "cat /etc/myhosts"
```

---

## 12. Create and Use Partitions

**Requirements:**
Create `/home/student/ansible/partition.yml` that:
- Creates a 1500M primary partition (partition number 1) on `vdb`, formatted ext4
- **prod** group permanently mounts the partition to `/data`
- If there isn't enough disk space, print: `Could not create partition of that size` and create an 800M partition instead
- If `vdb` doesn't exist, print: `this disk is not exist`

### Solution

**partition.yml:**
```yaml
---
- name: Create Partition
  hosts: all
  tasks:
    - name: Checking if vdb exists
      debug:
        msg: "this disk is not exist"
      when: "'vdb' not in ansible_devices"

    - name: Create 1500m Partition
      block:
        - name: Create Partition
          parted:
            device: /dev/vdb
            state: present
            number: 1
            part_end: 1500MiB
          when: "'vdb' in ansible_devices"

        - name: Update-Info
          setup:
            filter: ansible_devices

      rescue:
        - name: Checking if vdb exists
          debug:
            msg: "Could not create partition of that size"
          when: "'vdb' not in ansible_devices"

        - name: Create Partition 800MiB
          parted:
            device: /dev/vdb
            state: present
            number: 1
            part_end: 800MiB
          when: "'vdb' in ansible_devices"

        - name: Update-Info
          setup:
            filter: ansible_devices

      always:
        - name: Format Filesystem
          filesystem:
            device: /dev/vdb1
            fstype: ext4
          when: "'vdb1' in ansible_devices.vdb.partitions"

        - name: Create directory
          file:
            path: /data
            state: directory

        - name: Mount in /data
          mount:
            src: /dev/vdb1
            path: /data
            fstype: ext4
            state: mounted
          when: "'prod' in group_names"
```

---

## 13. Create and Use a Logical Volume

**Requirements:**
Create `/home/student/ansible/lv.yml` that runs on all managed nodes and:
- Creates a logical volume:
  - in volume group `research`
  - name `data`
  - size 1500MiB
  - formatted ext4
- If 1500MiB can't be created, display `Could not create logical volume of that size` and create 800MiB instead
- If `research` VG doesn't exist, display `volume group does not exist`
- Does **not** mount the LV

### Solution

**lv.yml:**
```yaml
---
- name: Creates a logical volume
  hosts: all
  tasks:
    - name: Checking if research VG exists
      debug:
        msg: "volume group does not exist"
      when: "'research' not in ansible_lvm.vgs"

    - name: Configure LVM
      block:
        - name: Create LVM of 1500 MiB
          lvol:
            vg: research
            lv: data
            size: 1500m
            state: present
          when: "'research' in ansible_lvm.vgs"

        - name: Update-Info
          setup:
            filter: ansible_lvm

      rescue:
        - name: Print Message
          debug:
            msg: "Could not create logical volume of that size"
          when: "'research' not in ansible_lvm.vgs"

        - name: Create LVM of 800MiB
          lvol:
            vg: research
            lv: data
            size: 800m
            state: present
          when: "'research' in ansible_lvm.vgs"

        - name: Update-Info
          setup:
            filter: ansible_lvm

      always:
        - name: Format File System
          filesystem:
            device: /dev/research/data
            fstype: ext4
          when: "'data' in ansible_lvm.lvs"
```

---

## 14. Use a RHEL System Role — Timesync

**Requirements:**
- Install the RHEL System Role package on the control node
- Create `/home/student/ansible/timesync.yml` that:
  - Runs on all managed nodes
  - Uses the `timesync` role
  - Uses the currently active NTP provider
  - Configures time server `172.25.254.254`
  - Enables the `iburst` parameter

### Solution

```bash
sudo yum install rhel-system-roles.noarch
vi /usr/share/ansible/roles/rhel-system-roles.timesync/README.md
# copy the example yml format, then paste into timesync.yml
```

**timesync.yml:**
```yaml
---
- name: NTP Time Synchronization
  hosts: all
  vars:
    timesync_ntp_servers:
      - hostname: 172.25.254.254
        iburst: yes
  roles:
    - rhel-system-roles.timesync
```

```bash
ansible-playbook timesync.yml
ansible all -a "chronyc tracking"
ansible all -a "chronyc sources -v"
```

---

## 15. Use a RHEL System Role — SELinux

**Requirements:**
- Install the RHEL system role package
- Create `/home/student/ansible/selinux.yml` that:
  - Runs on all managed nodes
  - Uses the `selinux` role
  - Sets SELinux to **enforcing** mode

### Solution

```bash
sudo yum install rhel-system-roles -y   # if not installed
vi /usr/share/ansible/roles/rhel-system-roles.selinux/README.md
# copy example yml format into selinux.yml

ansible all -m command -a 'sestatus'
```

**If SELinux is Permissive:**
```yaml
---
- name: use rhel system role to selinux
  hosts: all
  vars:
    selinux_policy: targeted
    selinux_state: enforcing
  roles:
    - rhel-system-roles.selinux
```

**If SELinux is Disabled** (requires reboot for the change to take effect):
```yaml
---
- name: use rhel selinux role
  hosts: all
  vars:
    selinux_policy: targeted
    selinux_state: enforcing
  tasks:
    - name: execute selinux role to set enforcing mode
      block:
        - name: apply selinux role
          include_role:
            name: rhel-system-roles.selinux
      rescue:
        - name: reboot failed managed host
          shell: "shutdown -r now"

        - name: wait for managed host to come back
          wait_for_connection:
            delay: 60
            timeout: 300

        - name: reapply the role
          include_role:
            name: rhel-system-roles.selinux
```

```bash
ansible-playbook selinux.yml
ansible all -a 'sestatus'
```

---

## 16. Install Roles Using Ansible Galaxy

**Requirements:**
Using a requirements file `/home/student/ansible/roles/requirement.yml`, download and install roles into `/home/student/ansible/roles`:
- `http://content.example.com/EX294_Ansible/Ansible_files/balancer.tar.gz` → role name `balancer`
- `http://content.example.com/EX294_Ansible/Ansible_files/phpinfo.tar.gz` → role name `phpinfo`

### Solution

**roles/requirement.yml:**
```yaml
- name: phpinfo
  src: http://content.example.com/EX294_Ansible/Ansible_files/phpinfo.tar.gz
- name: balancer
  src: http://content.example.com/EX294_Ansible/Ansible_files/balancer.tar.gz
```

```bash
ansible-galaxy list
ansible-galaxy install -r roles/requirement.yml -p roles/
ls -l roles/
```

---

## 17a. Create a Role — Apache

**Requirements:**
Create a role `apache` in `/home/student/ansible/roles`:
- `httpd` installed, enabled on boot, and started
- Firewall enabled/running with a rule allowing access to the web server
- Template `index.html.j2` serving `/var/www/html/index.html` with:
  `Welcome to HOSTNAME on IPADDRESS`
  (HOSTNAME = managed node's FQDN, IPADDRESS = managed node's IP)
- Restart the web server whenever the content file changes

Create `/home/student/ansible/newrole.yml` that uses this role, running on hosts in the **webservers** host group.

### Solution

```bash
ansible-galaxy list
ansible-galaxy init roles/apache
vi roles/apache/tasks/main.yml
```

**roles/apache/tasks/main.yml:**
```yaml
---
# tasks file for apache
- name: Installing httpd
  yum:
    name: httpd
    state: present

- name: Start and enable httpd
  service:
    name: httpd
    state: started
    enabled: yes

- name: Enable firewall service
  service:
    name: firewalld
    state: started
    enabled: yes

- name: Allow firewall for http
  firewalld:
    service: http
    permanent: yes
    immediate: yes
    state: enabled

- name: Create index.html file from template
  template:
    src: index.html.j2
    dest: /var/www/html/index.html
  notify: restart webserver
```

**roles/apache/templates/index.html.j2:**
```jinja2
Welcome to {{ ansible_fqdn }} on {{ ansible_default_ipv4.address }}
```

**roles/apache/handlers/main.yml:**
```yaml
---
# handlers file for apache
- name: restart webserver
  service:
    name: httpd
    state: restarted
```

**newrole.yml:**
```yaml
---
- name: use apache role
  hosts: webservers
  roles:
    - apache
```

```bash
ansible-playbook newrole.yml
curl http://serverc.lab.example.com
```

---

## 17b. Use Roles from Ansible Galaxy — Balancer + PHP Info

**Requirements:**
Create `/home/student/ansible/roles.yml`:
- Play 1: runs on hosts in the **balancers** host group, uses the `balancer` role
  - Configures load balancing between hosts in the **webservers** host group
  - Browsing a host in **balancers** (e.g. `http://serverd.lab.example.com`) should output:
    `Welcome to serverc.lab.example.com on 172.25.250.12` — then on reload, output from the alternate server:
    `Welcome to serverd.lab.example.com on 172.25.250.13`
- Play 2: runs on hosts in the **webservers** host group, uses the `phpinfo` role
  - Browsing `http://serverc.lab.example.com/hello.php` should output:
    `Hello PHP World from serverc.lab.example.com` plus PHP configuration details (including PHP version)
  - Similarly for `http://serverb.lab.example.com/hello.php`

### Solution

**roles.yml:**
```yaml
---
- name: uses the balancer role on balancers host group
  hosts: balancers
  roles:
    - balancer

- name: use the phpinfo role on webservers host group
  hosts: webservers
  roles:
    - phpinfo
```

```bash
ansible-playbook roles.yml
curl http://serverc.lab.example.com
```

---

## Quick Checklist

| # | Task | Status |
|---|------|--------|
| 01 | Install & configure Ansible (inventory + ansible.cfg) | ☐ |
| 02 | Ad-hoc commands for yum repos | ☐ |
| 03 | Install packages (php, mariadb, Dev Tools, updates) | ☐ |
| 04 | Modify `/etc/issue` per host group | ☐ |
| 05 | Web content directory `/webdev` | ☐ |
| 06 | Cron job for natasha every 2 min | ☐ |
| 07 | Create password vault (locker.yml) | ☐ |
| 08 | Create user accounts (developer/manager) | ☐ |
| 09 | Rekey vault (salaries.yml) | ☐ |
| 10 | Generate hardware report | ☐ |
| 11 | Generate `/etc/myhosts` via template | ☐ |
| 12 | Create partitions on vdb w/ rescue logic | ☐ |
| 13 | Create LV in `research` VG w/ rescue logic | ☐ |
| 14 | RHEL system role: timesync | ☐ |
| 15 | RHEL system role: selinux enforcing | ☐ |
| 16 | Install roles via Ansible Galaxy | ☐ |
| 17a | Create `apache` role | ☐ |
| 17b | Use `balancer` + `phpinfo` roles | ☐ |
