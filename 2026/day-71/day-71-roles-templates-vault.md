# Day 71 -- Roles, Galaxy, Templates and Vault

## Task

The playbooks were getting bigger, with tasks, variables, handlers, and files living in YAML files. Ansible Roles, Jinja2 Templates, Ansible Galaxy, and Ansible Vault were used to organize, reuse, share, and secure automation.

---

## All ansible playbooks & files

[ansible playbooks files](https://github.com/Mujakkir-Pathan/ansible-playbooks/tree/main/day-4)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Jinja2 Templates

Created `templates/nginx-vhost.conf.j2`:

```jinja2
# Managed by Ansible -- do not edit manually
server {
    listen {{ http_port | default(80) }};
    server_name {{ ansible_hostname }};

    root /var/www/{{ app_name }};
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

Created `template-demo.yml` and used `apt` instead of `yum` because the target web server was running Ubuntu.

Ran:

```bash
ansible-playbook template-demo.yml --diff
```

The playbook completed successfully:

```text
ok=6 changed=4 unreachable=0 failed=0
```

**Verify:** SSH into the web server and read the generated config. Are the variables replaced with actual values?

The generated configuration contained the actual values:

```text
listen 80;
server_name ip-172-31-15-193;

root /var/www/terraweek-app;
index index.html;

location / {
    try_files $uri $uri/ =404;
}

access_log /var/log/nginx/terraweek-app_access.log;
error_log /var/log/nginx/terraweek-app_error.log;
```

The index page was also rendered with actual host and IP values:

```html
<h1>terraweek-app</h1><p>Host: ip-172-31-15-193 | IP: 172.31.15.193</p>
```

---

### Task 2: Understand the Role Structure

Generated the role skeleton with:

```bash
ansible-galaxy init roles/webserver
```

The generated directory structure was:

```text
roles/webserver
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

**Document:** What is the difference between `vars/main.yml` and `defaults/main.yml`?

`defaults/main.yml` contains default variables with lower priority, so they can be easily overridden. `vars/main.yml` contains variables with higher priority and is intended for values that generally should not be changed.

---

### Task 3: Build a Custom Webserver Role

Created the `webserver` role with tasks, handlers, templates, defaults, and variables.

`roles/webserver/defaults/main.yml`:

```yaml
---
http_port: 80
app_name: myapp
max_connections: 512
```

The role used `apt` instead of `yum` because the target server was running Ubuntu.

The role included tasks to install Nginx, deploy the Nginx configuration, deploy the vhost configuration, create the web root, deploy the index page, and start and enable Nginx.

Created the Nginx configuration templates and the dynamic index page template.

Created `site.yml`:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  roles:
    - role: webserver
      vars:
        app_name: terraweek
        http_port: 80
```

Ran:

```bash
ansible-playbook site.yml
```

The playbook completed successfully:

```text
ok=8 changed=5 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

**Verify:** Curl the web server. Does the custom page load?

The custom page loaded successfully:

```text
<h1>terraweek-app</h1><p>Host: ip-172-31-15-193 | IP: 172.31.15.193</p>
```

---

### Task 4: Ansible Galaxy -- Use Community Roles

Searched for Nginx and MySQL roles:

```bash
ansible-galaxy search nginx --platforms EL
ansible-galaxy search mysql
```

The searches returned:

```text
Nginx: 819 roles
MySQL: 690 roles
```

Installed the Docker role:

```bash
ansible-galaxy install geerlingguy.docker
```

The installed version was:

```text
geerlingguy.docker (8.0.0) was installed successfully
```

Verified the installed role with:

```bash
ansible-galaxy list
```

Created `docker-setup.yml`:

```yaml
---
- name: Install Docker using Galaxy role
  hosts: app
  become: true
  roles:
    - geerlingguy.docker
```

Ran the playbook successfully:

```text
App-server : ok=14 changed=4 unreachable=0 failed=0 skipped=13
```

Created `requirements.yml`:

```yaml
---
roles:
  - name: geerlingguy.docker
    version: "7.4.1"
  - name: geerlingguy.ntp
```

Installed the roles with:

```bash
ansible-galaxy install -r requirements.yml
```

`geerlingguy.ntp` version `4.0.1` was installed successfully. The installed Docker role was already version `8.0.0`, while `requirements.yml` specified `7.4.1`.

**Document:** Why use a `requirements.yml` instead of installing roles manually?

A `requirements.yml` file defines role dependencies and versions in one place. This makes role installation reproducible, consistent, and easier to manage across environments and automated pipelines.

---

### Task 5: Ansible Vault -- Encrypt Secrets

Created the encrypted Vault file:

```bash
ansible-vault create group_vars/db/vault.yml
```

The Vault file contained:

```yaml
vault_db_password: SuperSecretP@ssw0rd
vault_db_root_password: R00tP@ssw0rd123
vault_api_key: sk-abc123xyz789
```

Verified that the file was encrypted:

```bash
head -n 1 group_vars/db/vault.yml
```

Output:

```text
$ANSIBLE_VAULT;1.1;AES256
```

The encrypted file was edited and viewed using Ansible Vault commands.

An attempt was made to encrypt `group_vars/db/secrets.yml`, but the file did not exist. The Vault file had already been created and encrypted.

Created `db-setup.yml`:

```yaml
---
- name: Configure database
  hosts: db
  become: true

  tasks:
    - name: Show DB password (never do this in production)
      debug:
        msg: "DB password is set: {{ vault_db_password | length > 0 }}"
```

Ran:

```bash
ansible-playbook db-setup.yml --ask-vault-pass
```

The playbook completed successfully:

```text
DB-server : ok=2 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

The Vault variable was successfully loaded:

```text
DB password is set: True
```

Created `.vault_pass`, protected it with `chmod 600`, and added `.vault_pass` to `.gitignore`.

Ran:

```bash
ansible-playbook db-setup.yml --vault-password-file .vault_pass
```

The playbook completed successfully and loaded the Vault variable.

**Document:** Why is `--vault-password-file` better than `--ask-vault-pass` for automated pipelines?

`--vault-password-file` allows the Vault password to be supplied automatically without interactive input, making it suitable for automated pipelines.

---

### Task 6: Combine Roles, Templates, and Vault

Updated `site.yml` to configure the web, app, and database servers using the custom role, Galaxy role, and Vault-backed template.

Created `templates/db-config.j2`:

```jinja2
# Database Configuration -- Managed by Ansible
DB_HOST={{ ansible_default_ipv4.address }}
DB_PORT={{ db_port | default(3306) }}
DB_PASSWORD={{ vault_db_password }}
DB_ROOT_PASSWORD={{ vault_db_root_password }}
```

Ran:

```bash
ansible-playbook site.yml
```

The playbook completed successfully:

```text
App-server : ok=12 changed=0 unreachable=0 failed=0 skipped=13
DB-server  : ok=2 changed=1 unreachable=0 failed=0 skipped=0
web-server : ok=7 changed=0 unreachable=0 failed=0 skipped=0
```

**Verify:** SSH into the db server and check `/etc/db-config.env`. Are the secrets rendered correctly? Is the file permission `600`?

The file permissions were verified as:

```text
-rw------- 1 root root 146 Sep 9 10:53 /etc/db-config.env
```

The file was then read with root privileges:

```bash
ansible db -b -m command -a "cat /etc/db-config.env"
```

The generated configuration contained the Vault values and dynamic database host:

```text
# Database Configuration -- Managed by Ansible
DB_HOST=172.31.5.252
DB_PORT=3306
DB_PASSWORD=SuperSecretP@ssw0rd
DB_ROOT_PASSWORD=R00tP@ssw0rd123
```

The secrets were rendered correctly and the file permission was `600`.

---

## Webserver Role Directory Structure

Created the custom `webserver` role with the following structure:

```text
roles/
  webserver/
    tasks/
      main.yml
    handlers/
      main.yml
    templates/
      nginx.conf.j2
      vhost.conf.j2
      index.html.j2
    files/
    vars/
      main.yml
    defaults/
      main.yml
    meta/
      main.yml
```

## Galaxy Role Installation and Usage

Installed the `geerlingguy.docker` role from Ansible Galaxy:

```bash
ansible-galaxy install geerlingguy.docker
```

Verified the installed role using:

```bash
ansible-galaxy list
```

Used the Galaxy role in `docker-setup.yml`:

```yaml
roles:
  - geerlingguy.docker
```

The Docker role was successfully used to configure the app server.

Created `requirements.yml` to manage multiple Galaxy roles and their versions.

## Vault Workflow

Created the encrypted Vault file:

```bash
ansible-vault create group_vars/db/vault.yml
```

Edited the encrypted file using:

```bash
ansible-vault edit group_vars/db/vault.yml
```

Viewed the encrypted file using:

```bash
ansible-vault view group_vars/db/vault.yml
```

Verified that the file was encrypted:

```bash
head -n 1 group_vars/db/vault.yml
```

Output:

```text
$ANSIBLE_VAULT;1.1;AES256
```

The command for encrypting an existing file was:

```bash
ansible-vault encrypt group_vars/db/secrets.yml
```

The command for decrypting a Vault file was:

```bash
ansible-vault decrypt group_vars/db/vault.yml
```

The Vault file was used successfully in the database configuration.

## When to Use Roles vs Playbooks vs Ad-hoc Commands

* **Roles:** Used to organize and reuse related automation such as tasks, handlers, templates, files, and variables.
* **Playbooks:** Used to define complete automation for one or more groups of servers.
* **Ad-hoc commands:** Used for quick, one-time operations and verification tasks.

