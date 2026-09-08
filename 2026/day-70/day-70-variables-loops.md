# Day 70 -- Variables, Facts, Conditionals and Loops

## Task

The playbooks were made flexible using variables, facts, conditionals,
and loops so automation could adapt to each host, group, and
environment.

------------------------------------------------------------------------

## All ansible playbooks files

[ansible playbooks files](https://github.com/Mujakkir-Pathan/ansible-playbooks/tree/main/day-3)

---

## All sceenshots

[screenshots of task](screenshots/)

------------------------------------------------------------------------

## Challenge Tasks

### Task 1: Variables in Playbooks

Created `variables-demo.yml`:

``` yaml
---
- name: Variable demo
  hosts: all
  become: true

  vars:
    app_name: terraweek-app
    app_port: 8080
    app_dir: "/opt/{{ app_name }}"
    packages:
      - git
      - curl
      - wget

  tasks:
    - name: Print app details
      debug:
        msg: "Deploying {{ app_name }} on port {{ app_port }} to {{ app_dir }}"

    - name: Create application directory
      file:
        path: "{{ app_dir }}"
        state: directory
        mode: '0755'

    - name: Install required packages
      apt:
        name: "{{ packages }}"
        state: present
```

The playbook was run and the variables resolved correctly.

The variables were overridden from the command line:

``` bash
ansible-playbook variables-demo.yml -e "app_name=my-custom-app app_port=9090"
```

The output showed:

``` text
Deploying my-custom-app on port 9090 to /opt/my-custom-app
```

**Verify:** Does the CLI variable override the playbook variable?

Yes. CLI variables override playbook variables.

------------------------------------------------------------------------

### Task 2: group_vars and host_vars

Created this structure:

``` text
ansible-practice/
  inventory.ini
  ansible.cfg
  group_vars/
    all.yml
    web.yml
    db.yml
  host_vars/
    web-server.yml
  playbooks/
    site.yml
```

**`group_vars/all.yml`** -- applied to every host:

``` yaml
---
ntp_server: pool.ntp.org
app_env: development
common_packages:
  - vim
  - htop
  - tree
```

**`group_vars/web.yml`** -- applied only to the web group:

``` yaml
---
http_port: 80
max_connections: 1000
web_packages:
  - nginx
```

**`group_vars/db.yml`** -- applied only to the db group:

``` yaml
---
db_port: 3306
db_packages:
  - mysql-server
```

**`host_vars/web-server.yml`** -- applied only to this specific host:

``` yaml
---
max_connections: 2000
custom_message: "This is the primary web server"
```

Created `site.yml` using these variables:

``` yaml
---
- name: Apply common config
  hosts: all
  become: true
  tasks:
    - name: Install common packages
      apt:
        name: "{{ common_packages }}"
        state: present
    - name: Show environment
      debug:
        msg: "Environment: {{ app_env }}"

- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Show web config
      debug:
        msg: "HTTP port: {{ http_port }}, Max connections: {{ max_connections }}"
    - name: Show host-specific message
      debug:
        msg: "{{ custom_message }}"
```

The playbook was run and the variables applied correctly to the matching
hosts.

The web server output showed:

``` text
HTTP port: 80, Max connections: 2000
This is the primary web server
```

**Document:** What is the variable precedence?

The tested precedence was:

``` text
host_vars > group_vars/<group> > group_vars/all
```

Extra variables supplied with `-e` override everything.

------------------------------------------------------------------------

### Task 3: Ansible Facts -- Gathering System Information

All facts for the web server were gathered:

``` bash
ansible web-server -m setup
```

Specific facts were filtered:

``` bash
ansible web-server -m setup -a "filter=ansible_os_family"
ansible web-server -m setup -a "filter=ansible_distribution*"
ansible web-server -m setup -a "filter=ansible_memtotal_mb"
ansible web-server -m setup -a "filter=ansible_default_ipv4"
```

Created `facts-demo.yml`:

``` yaml
---
- name: Facts demo
  hosts: all
  tasks:
    - name: Show OS info
      debug:
        msg: >
          Hostname: {{ ansible_hostname }},
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }},
          RAM: {{ ansible_memtotal_mb }}MB,
          IP: {{ ansible_default_ipv4.address }}

    - name: Show all network interfaces
      debug:
        var: ansible_interfaces
```

The facts were run and printed for each host.

**Document:** Name five facts you would use in real playbooks and why.

1.  `ansible_hostname` --- used to identify the hostname.
2.  `ansible_distribution` --- used to determine the operating system.
3.  `ansible_distribution_version` --- used to check the OS version.
4.  `ansible_memtotal_mb` --- used to check available system memory.
5.  `ansible_default_ipv4.address` --- used to get the default IPv4
    address.

------------------------------------------------------------------------

### Task 4: Conditionals with when

Created `conditional-demo.yml`:

``` yaml
---
- name: Conditional tasks demo
  hosts: all
  become: true

  tasks:
    - name: Install Nginx (only on web servers)
      apt:
        name: nginx
        state: present
      when: "'web' in group_names"

    - name: Install MySQL (only on db servers)
      apt:
        name: mysql-server
        state: present
      when: "'db' in group_names"

    - name: Show warning on low memory hosts
      debug:
        msg: "WARNING: This host has less than 1GB RAM"
      when: ansible_memtotal_mb < 1024

    - name: Run only on Amazon Linux
      debug:
        msg: "This is an Amazon Linux machine"
      when: ansible_distribution == "Amazon"

    - name: Run only on Ubuntu
      debug:
        msg: "This is an Ubuntu machine"
      when: ansible_distribution == "Ubuntu"

    - name: Run only in production
      debug:
        msg: "Production settings applied"
      when: app_env == "production"

    - name: Multiple conditions (AND)
      debug:
        msg: "Web server with enough memory"
      when:
        - "'web' in group_names"
        - ansible_memtotal_mb >= 512

    - name: OR condition
      debug:
        msg: "Either web or app server"
      when: "'web' in group_names or 'app' in group_names"
```

The playbook was run and the conditional tasks were skipped or executed
according to their conditions.

**Verify:** Are tasks correctly skipping on hosts that don't match the
condition?

Yes. Tasks correctly skipped on hosts that did not match the condition.

------------------------------------------------------------------------

### Task 5: Loops

Created `loops-demo.yml`:

``` yaml
---
- name: Loops demo
  hosts: all
  become: true

  vars:
    users:
      - name: deploy
        groups: sudo
      - name: monitor
        groups: sudo
      - name: appuser
        groups: users

    directories:
      - /opt/app/logs
      - /opt/app/config
      - /opt/app/data
      - /opt/app/tmp

  tasks:
    - name: Create multiple users
      user:
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      loop: "{{ users }}"

    - name: Create multiple directories
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop: "{{ directories }}"

    - name: Install multiple packages
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - git
        - curl
        - unzip
        - jq

    - name: Print each user created
      debug:
        msg: "Created user {{ item.name }} in group {{ item.groups }}"
      loop: "{{ users }}"
```

The playbook was run and each loop iteration was shown separately.

**Document:** What is the difference between `loop` and the older
`with_items`?

`loop` is the modern recommended syntax for repeating tasks, while
`with_items` is the older syntax.

------------------------------------------------------------------------

### Task 6: Register, Debug, and Combine Everything

Built the real-world playbook `server-report.yml` combining variables,
facts, conditionals, and register:

``` yaml
---
- name: Server Health Report
  hosts: all

  tasks:
    - name: Check disk space
      command: df -h /
      register: disk_result

    - name: Check memory
      command: free -m
      register: memory_result

    - name: Check running services
      shell: systemctl list-units --type=service --state=running | head -20
      register: services_result

    - name: Generate report
      debug:
        msg:
          - "========== {{ inventory_hostname }} =========="
          - "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
          - "IP: {{ ansible_default_ipv4.address }}"
          - "RAM: {{ ansible_memtotal_mb }}MB"
          - "Disk: {{ disk_result.stdout_lines[1] }}"
          - "Running services (first 20): {{ services_result.stdout_lines | length }}"

    - name: Flag if disk is critically low
      debug:
        msg: "ALERT: Check disk space on {{ inventory_hostname }}"
      when: "'9[0-9]%' in disk_result.stdout or '100%' in disk_result.stdout"

    - name: Save report to file
      copy:
        content: |
          Server: {{ inventory_hostname }}
          OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
          IP: {{ ansible_default_ipv4.address }}
          RAM: {{ ansible_memtotal_mb }}MB
          Disk: {{ disk_result.stdout }}
          Checked at: {{ ansible_date_time.iso8601 }}
        dest: "/tmp/server-report-{{ inventory_hostname }}.txt"
      become: true
```

The playbook was run successfully.

The generated reports contained:

``` text
App-server
OS: Ubuntu 26.04
IP: 172.31.4.75
RAM: 908MB
Disk: 42%

web-server
OS: Ubuntu 26.04
IP: 172.31.15.193
RAM: 908MB
Disk: 43%

DB-server
OS: Ubuntu 26.04
IP: 172.31.5.252
RAM: 1905MB
Disk: 48%
```

The critical-disk condition was skipped on all three servers because
disk usage was below 90%.

**Verify:** SSH into a server and read `/tmp/server-report-*.txt`. Does
it contain accurate information?

Yes. The report files were created successfully and contained accurate
server hostname, OS, IP, RAM, disk usage, and check timestamp
information.

------------------------------------------------------------------------

## `group_vars/` and `host_vars/` Directory Structure

``` text
ansible-practice/
├── group_vars/
│   ├── all.yml
│   ├── db.yml
│   └── web.yml
├── host_vars/
│   └── web-server.yml
└── playbooks/
    └── site.yml
```

## Variable Precedence

The tested precedence was:

``` text
host_vars > group_vars/<group> > group_vars/all
```

For example, `max_connections` was `1000` in `group_vars/web.yml`, but
`web-server` received `2000` from `host_vars/web-server.yml`.

CLI extra variables using `-e` override the other variables tested.

## Five Useful Ansible Facts

  -----------------------------------------------------------------------
  Fact                                Where it would be used
  ----------------------------------- -----------------------------------
  `ansible_hostname`                  Identify the server hostname

  `ansible_distribution`              Select tasks based on the operating
                                      system

  `ansible_distribution_version`      Check or apply OS-version-specific
                                      configuration

  `ansible_memtotal_mb`               Make decisions based on available
                                      memory

  `ansible_default_ipv4.address`      Use the server's default IPv4
                                      address in configurations or
                                      reports
  -----------------------------------------------------------------------

