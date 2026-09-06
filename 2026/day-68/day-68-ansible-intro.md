# Day 68 -- Introduction to Ansible and Inventory Setup

## Task

Terraform provisions infrastructure. Ansible was used for post-provision configuration such as installing packages, configuring servers, managing files, and maintaining the desired state. Ansible was used as an agentless configuration management tool that connected to managed nodes through SSH.

---

## All tf files

[ansible playbooks files](https://github.com/Mujakkir-Pathan/ansible-playbooks/tree/main/day-1)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Understand Ansible

1. **What is configuration management? Why do we need it?**

Configuration management is the process of managing server configuration in a consistent and repeatable way. It was needed to automate tasks such as installing packages, configuring services, managing files, and keeping servers in the desired state.

2. **How is Ansible different from Chef, Puppet, and Salt?**

Ansible is agentless and normally connects to Linux managed nodes through SSH. Chef and Puppet commonly use agents on managed nodes, while Salt commonly uses a Salt minion. Ansible uses simple YAML playbooks and modules, making it easy to start with.

3. **What does "agentless" mean? How does Ansible connect to managed nodes?**

Agentless means that Ansible does not require an Ansible agent to be installed on the managed servers. Ansible connected to the Ubuntu EC2 instances using SSH and the configured private key.

4. **Ansible architecture**

```text
                    Control Node
                 (Ansible installed)
                        |
                        | SSH
                        |
          +-------------+-------------+
          |             |             |
       Web Node      App Node      DB Node
     Managed Node  Managed Node  Managed Node

              Inventory
                 |
        +--------+--------+
        |        |        |
       web      app      db

Modules → units of work such as ping, command, apt, and copy

Playbooks → YAML files that define repeatable tasks
```

* **Control Node** -- The Ubuntu EC2 instance where Ansible was installed and executed.
* **Managed Nodes** -- The Web, App, and DB Ubuntu EC2 instances configured by Ansible.
* **Inventory** -- The `inventory.ini` file containing the managed hosts and groups.
* **Modules** -- Units of work executed by Ansible, such as `command`, `apt`, `copy`, and `ping`.
* **Playbooks** -- YAML files used to define repeatable configuration tasks.

---

### Task 2: Set Up Your Lab Environment

The lab was created using **Option B: Launch manually from AWS Console**.

Three Ubuntu EC2 instances were created as managed nodes:

| Instance   | Role       | Private IP      |
| ---------- | ---------- | --------------- |
| Instance 1 | Web server | `172.31.15.193` |
| Instance 2 | App server | `172.31.4.75`   |
| Instance 3 | DB server  | `172.31.5.252`  |

A dedicated Ubuntu EC2 instance was used as the **Ansible Control Node**.

The SSH key `terra.pem` was stored on the Control Node with restrictive permissions:

```text
-r-------- 1 ubuntu ubuntu 1674 Sep  5 04:30 terra.pem
```

SSH connectivity from the Control Node to each managed node was verified successfully:

```bash
ssh -i terra.pem ubuntu@172.31.15.193
ssh -i terra.pem ubuntu@172.31.4.75
ssh -i terra.pem ubuntu@172.31.5.252
```

All three SSH connections were successful.

---

### Task 3: Install Ansible

Ansible was installed on the dedicated Ubuntu Control Node.

The installed version was:

```text
ansible [core 2.20.1]
```

The verification also showed:

```text
config file = None
executable location = /usr/bin/ansible
python version = 3.14.4
```

**Document:** Ansible was installed on the dedicated Ubuntu EC2 Control Node. It was only needed on the Control Node because Ansible is agentless and connects to the managed nodes remotely through SSH.

---

### Task 4: Create Your Inventory File

The project directory `ansible-practice` was created.

The `inventory.ini` file was configured with three groups:

```ini
[web]
web-server ansible_host=172.31.15.193

[app]
App-server ansible_host=172.31.4.75

[db]
DB-server ansible_host=172.31.5.252

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/terra.pem
```

Ansible successfully reached all hosts:

```text
DB-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

App-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

web-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

The SSH key permissions were correctly restricted, the EC2 security group allowed SSH access, and `ansible_user` was set to `ubuntu` because the managed nodes were Ubuntu instances.

---

### Task 5: Run Ad-Hoc Commands

Ad-hoc commands were used to perform quick one-off tasks without creating a playbook.

1. **Check uptime on all servers:**

```bash
ansible all -i inventory.ini -m command -a "uptime"
```

Output:

```text
App-server | CHANGED | rc=0 >>
06:39:18 up 2:27, 1 user, load average: 0.00, 0.00, 0.00

web-server | CHANGED | rc=0 >>
06:39:18 up 2:27, 1 user, load average: 0.00, 0.00, 0.00

DB-server | CHANGED | rc=0 >>
06:39:19 up 2:27, 1 user, load average: 0.00, 0.00, 0.00
```

2. **Check free memory on web servers only:**

```bash
ansible web -i inventory.ini -m command -a "free -h"
```

Output:

```text
web-server | CHANGED | rc=0 >>
               total        used        free      shared  buff/cache   available
Mem:           908Mi       325Mi       139Mi       2.8Mi       555Mi       583Mi
Swap:             0B          0B          0B
```

3. **Check disk space on all servers:**

```bash
ansible all -i inventory.ini -m command -a "df -h"
```

The root filesystem usage was:

```text
web-server | /dev/root  6.7G  2.2G  4.5G  33% /
App-server | /dev/root  6.7G  2.1G  4.6G  32% /
DB-server  | /dev/root  6.7G  2.1G  4.6G  32% /
```

4. **Install a package on the web group:**

Because the managed nodes were Ubuntu, the `apt` module was used instead of `yum`:

```bash
ansible web -i inventory.ini -m apt -a "name=git state=present update_cache=yes" --become
```

Output:

```text
web-server | SUCCESS => {
    "cache_updated": true,
    "changed": false
}
```

5. **Copy a file to all servers:**

```bash
echo "Hello from Ansible" > hello.txt
ansible all -i inventory.ini -m copy -a "src=hello.txt dest=/tmp/hello.txt"
```

The file was successfully copied to:

```text
App-server → /tmp/hello.txt
DB-server  → /tmp/hello.txt
web-server → /tmp/hello.txt
```

6. **Verify the file was copied:**

```bash
ansible all -i inventory.ini -m command -a "cat /tmp/hello.txt"
```

Output:

```text
App-server | CHANGED | rc=0 >>
Hello from Ansible

web-server | CHANGED | rc=0 >>
Hello from Ansible

DB-server | CHANGED | rc=0 >>
Hello from Ansible
```

**Document:** `--become` was used to execute a task with elevated privileges, normally through `sudo`. It was needed when installing system packages because package installation requires root privileges.

---

### Task 6: Explore Inventory Groups and Patterns

1. **Create a group of groups**

The following groups were added to `inventory.ini`:

```ini
[application:children]
web
app

[all_servers:children]
application
db
```

This created:

```text
application
├── web
└── app

all_servers
├── application
│   ├── web
│   └── app
└── db
```

2. **Run commands against different groups**

```bash
ansible application -i inventory.ini -m ping
```

Result:

```text
web-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

App-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

```bash
ansible db -i inventory.ini -m ping
```

Result:

```text
DB-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

```bash
ansible all_servers -i inventory.ini -m ping
```

Result:

```text
App-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

web-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

DB-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

3. **Use patterns**

```bash
ansible 'web:app' -i inventory.ini -m ping
```

The `web:app` pattern successfully targeted the Web and App servers:

```text
web-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

App-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

```bash
ansible 'all:!db' -i inventory.ini -m ping
```

The `all:!db` pattern successfully targeted all servers except the DB server:

```text
web-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

App-server | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

4. **Create an `ansible.cfg` to avoid typing `-i inventory.ini` every time**

The following `ansible.cfg` was created:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/terra.pem
```

Ansible was then run without specifying the inventory file:

```bash
ansible all -m ping
```

**Verify:** Does `ansible all -m ping` work without specifying the inventory file?

Yes. The command worked successfully without `-i inventory.ini`, and all three managed nodes returned `"ping": "pong"`.

---

* Ansible architecture in your own words

Ansible used a Control Node to manage remote Managed Nodes through SSH. The Inventory defined the servers and their groups. Modules performed individual tasks, while Playbooks were used to define repeatable automation.

* How you set up your lab with Terraform with instance details

The lab was not provisioned with Terraform. Three Ubuntu EC2 instances were launched manually from the AWS Console and used as Web, App, and DB managed nodes. A separate Ubuntu EC2 instance was used as the Ansible Control Node.

* Your `inventory.ini` file (redact IPs if sharing publicly)

```ini
[web]
web-server ansible_host=172.31.15.193

[app]
App-server ansible_host=172.31.4.75

[db]
DB-server ansible_host=172.31.5.252

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/terra.pem

[application:children]
web
app

[all_servers:children]
application
db
```

* Five ad-hoc commands you ran and their outputs

```bash
ansible all -i inventory.ini -m command -a "uptime"
```

```text
App-server | CHANGED | rc=0
web-server | CHANGED | rc=0
DB-server | CHANGED | rc=0
```

```bash
ansible web -i inventory.ini -m command -a "free -h"
```

```text
web-server | CHANGED | rc=0
Mem: 908Mi total, 325Mi used, 139Mi free
```

```bash
ansible all -i inventory.ini -m command -a "df -h"
```

```text
web-server | CHANGED | rc=0
/dev/root  6.7G  2.2G  4.5G  33% /

App-server | CHANGED | rc=0
/dev/root  6.7G  2.1G  4.6G  32% /

DB-server | CHANGED | rc=0
/dev/root  6.7G  2.1G  4.6G  32% /
```

```bash
ansible web -i inventory.ini -m apt -a "name=git state=present update_cache=yes" --become
```

```text
web-server | SUCCESS
"changed": false
```

```bash
ansible all -i inventory.ini -m copy -a "src=hello.txt dest=/tmp/hello.txt"
```

```text
App-server | CHANGED
DB-server | CHANGED
web-server | CHANGED
```

* Difference between `command` and `shell` modules

The `command` module executes commands directly without using a shell. The `shell` module executes commands through the system shell, so it supports shell features such as pipes (`|`), redirection (`>`), variables, and command chaining.

