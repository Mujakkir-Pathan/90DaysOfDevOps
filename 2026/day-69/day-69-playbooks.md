# Day 69 -- Ansible Playbooks and Modules

## Task

Ad-hoc commands are useful for quick checks, but real automation lives in playbooks. A playbook is a YAML file that describes the desired state of your servers -- which packages to install, which services to run, which files to place where. You write it once, run it a hundred times, and get the same result every time.

Today I wrote my first playbooks and learned the modules that I will use on every project.

---

## All ansible playbooks files

[ansible playbooks files](https://github.com/Mujakkir-Pathan/ansible-playbooks/tree/main/day-2)

---

## All sceenshots

[screenshots of task](screenshots/)

---
## Challenge Tasks

### Task 1: Your First Playbook

Created `install-nginx.yml`:

```yaml
---
- name: Install and start Nginx on web servers
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: true

    - name: Create a custom index page
      copy:
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1>"
        dest: /var/www/html/index.html
```

`apt` was used instead of `yum` because the instances were running Ubuntu.

Ran it:

```bash
ansible-playbook install-nginx.yml
```

The first run showed `changed` for the Nginx installation and custom index page.

The playbook was run again, and the tasks showed `ok` with `changed=0`. This demonstrated **idempotency** -- Ansible only made changes when needed.

**Verify:** Curl the web server's public IP. Do you see your custom page?

Yes. The custom page was displayed:

```text
<h1>Deployed by Ansible - TerraWeek Server</h1>
```

---

### Task 2: Understand the Playbook Structure

The playbook structure was understood as follows:

```yaml
---                                    # YAML document start
- name: Play name                      # PLAY -- targets a group of hosts
  hosts: web                           # Which inventory group to run on
  become: true                         # Run tasks as root (sudo)

  tasks:                               # List of TASKS in this play
    - name: Task name                  # TASK -- one unit of work
      module_name:                     # MODULE -- what Ansible does
        key: value                     # Module arguments
```

Answer:

1. What is the difference between a play and a task?

A **play** defines what should be done on a group of hosts, while a **task** is one specific action performed as part of that play.

2. Can you have multiple plays in one playbook?

Yes, a playbook can contain multiple plays, and each play can target a different group of hosts.

3. What does `become: true` do at the play level vs the task level?

`become: true` tells Ansible to run tasks with elevated privileges, usually as root through `sudo`. At the play level it applies to all tasks in that play. At the task level it applies only to that specific task.

4. What happens if a task fails -- do remaining tasks still run?

If a task fails, Ansible stops the remaining tasks on that host by default. Other hosts can continue executing their tasks.

---

### Task 3: Learn the Essential Modules

Created `essential-modules.yml` with multiple tasks using the essential Ansible modules.

1. **`apt`** -- Installed multiple packages:

```yaml
- name: Install multiple packages
  apt:
    name:
      - git
      - curl
      - wget
      - tree
      - nginx
    state: present
```

2. **`service`** -- Managed the Nginx service:

```yaml
- name: Ensure Nginx is running
  service:
    name: nginx
    state: started
    enabled: true
```

3. **`copy`** -- Copied `files/app.conf` from the control node to the managed nodes:

```yaml
- name: Copy config file
  copy:
    src: files/app.conf
    dest: /etc/app.conf
    owner: root
    group: root
    mode: '0644'
```

4. **`file`** -- Created the application directory and managed its permissions:

```yaml
- name: Create application directory
  file:
    path: /opt/myapp
    state: directory
    owner: ubuntu
    mode: '0755'
```

5. **`command`** -- Ran a command without shell features:

```yaml
- name: Check disk space
  command: df -h
  register: disk_output

- name: Print disk space
  debug:
    var: disk_output.stdout_lines
```

6. **`shell`** -- Ran a command with shell features:

```yaml
- name: Count running processes
  shell: ps aux | wc -l
  register: process_count

- name: Show process count
  debug:
    msg: "Total processes: {{ process_count.stdout }}"
```

7. **`lineinfile`** -- Added the timezone line to `/etc/environment`:

```yaml
- name: Set timezone in environment
  lineinfile:
    path: /etc/environment
    line: 'TZ=Asia/Kolkata'
    create: true
```

Created the `files/` directory and added `files/app.conf`:

```text
APP_NAME=myapp
APP_ENV=development
```

The playbook was run against all servers successfully.

**Document:** What is the difference between `command` and `shell`? When should you use each?

`command` runs commands without shell features such as pipes and redirects. `shell` runs commands through a shell and supports pipes and redirects. `command` should be used when shell features are not required, while `shell` should be used when shell features are needed.

---

### Task 4: Handlers -- Restart Services Only When Needed

Handlers were used to restart Nginx only when the configuration changed.

Created `nginx-config.yml`:

```yaml
---
- name: Configure Nginx with handlers
  hosts: web
  become: true

  tasks:
    - name: Copy Nginx configuration
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: Restart Nginx

    - name: Create custom index page
      copy:
        content: "<h1>Configured by Ansible Handler</h1>"
        dest: /var/www/html/index.html

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

Created `files/nginx.conf` with a basic Nginx configuration.

Run the playbook:

**First run:** The Nginx configuration changed, so the handler was triggered:

```text
TASK [Copy Nginx configuration] ... changed
TASK [Create custom index page] ... changed
TASK [Ensure Nginx is running] ... ok

RUNNING HANDLER [Restart Nginx] ... changed
```

**Second run:** Nothing changed, so the handler did not trigger:

```text
TASK [Copy Nginx configuration] ... ok
TASK [Create custom index page] ... ok
TASK [Ensure Nginx is running] ... ok
```

**Verify:** Run it twice and compare the output. Does the handler run both times?

No. The handler ran on the first run because the configuration file changed. It did not run on the second run because nothing changed.

---

### Task 5: Dry Run, Diff, and Verbosity

Before running playbooks on production, changes were previewed first.

1. **Dry run (check mode)** -- showed what would change without changing anything:

```bash
ansible-playbook install-nginx.yml --check
```

The playbook completed in check mode and showed the custom index page as `changed` without applying the change.

2. **Diff mode** -- showed the actual file differences:

```bash
ansible-playbook install-nginx.yml --check --diff
```

The output showed the difference between the existing and expected index page:

```text
-<h1>Configured by Ansible Handler</h1>
+<h1>Deployed by Ansible - TerraWeek Server</h1>
```

3. **Verbosity** -- increased output detail for debugging:

```bash
ansible-playbook install-nginx.yml -v
ansible-playbook install-nginx.yml -vv
ansible-playbook install-nginx.yml -vvv
```

`-v` showed more detailed information, `-vv` provided more detailed output, and `-vvv` was used for connection debugging.

4. **Limit to specific hosts:**

```bash
ansible-playbook install-nginx.yml --limit web-server
```

This limited playbook execution to the specified host.

5. **List what would be affected without running:**

```bash
ansible-playbook install-nginx.yml --list-hosts
ansible-playbook install-nginx.yml --list-tasks
```

`--list-hosts` displayed the hosts affected by the playbook, while `--list-tasks` displayed the tasks that would run.

**Document:** Why is `--check --diff` the most important flag combination for production use?

`--check --diff` allows changes to be previewed before applying them. `--check` shows what would change, while `--diff` shows the exact file differences. This helps detect unexpected changes and reduces the risk of making incorrect configuration changes in production.

---

### Task 6: Multiple Plays in One Playbook

Created `multi-play.yml` with separate plays for each server group:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true

  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present

    - name: Ensure Nginx is running
      service:
        name: nginx
        state: started
        enabled: true

- name: Configure app servers
  hosts: app
  become: true

  tasks:
    - name: Create application directory
      file:
        path: /opt/myapp
        state: directory
        owner: ubuntu
        mode: '0755'

- name: Configure database servers
  hosts: db
  become: true

  tasks:
    - name: Ensure MySQL is installed
      apt:
        name: mysql-server
        state: present
```

Run it:

```bash
ansible-playbook multi-play.yml
```

The playbook completed successfully:

```text
App-server : ok=2 changed=0 failed=0
DB-server  : ok=2 changed=0 failed=0
web-server : ok=3 changed=0 failed=0
```

Each play targeted only its relevant inventory group.

**Verify:** Is Nginx only installed on web servers? Is MySQL only on db servers?

Yes. Nginx was verified on `web-server`:

```text
ii  nginx  1.28.3-2ubuntu1.10 amd64
```

MySQL was verified on `DB-server`:

```text
ii  mysql-server  8.4.11-0ubuntu0.26.04.1 amd64
```

The app server was configured separately with its application directory.

---

## 1. **Your first playbook with annotations explaining each section**

```yaml
---
- name: Install and start Nginx on web servers   # Play name
  hosts: web                                     # Inventory group targeted by the play
  become: true                                   # Runs tasks with elevated privileges

  tasks:                                         # List of tasks in the play
    - name: Install Nginx                        # Task name
      apt:                                       # Module used to install Nginx
        name: nginx                              # Package to install
        state: present                            # Ensures the package is installed
```

## 2. **All seven module examples with what each does**

* **`apt`** — Installed and managed Ubuntu packages.
* **`service`** — Started, stopped, restarted, and enabled services.
* **`copy`** — Copied files from the control node to managed nodes.
* **`file`** — Created directories and managed file/directory properties.
* **`command`** — Ran commands without shell features such as pipes and redirects.
* **`shell`** — Ran commands through a shell and supported features such as pipes and redirects.
* **`lineinfile`** — Added or modified a single line in a file.

## 3. **How handlers work with a before/after comparison**

**Before using a handler:** A service could be restarted every time a playbook was run, even when the configuration had not changed.

**After using a handler:** The configuration task used `notify: Restart Nginx`. The handler restarted Nginx only when that task reported a change.

**First run:**

```text
Copy Nginx configuration → changed
Restart Nginx → changed
```

**Second run:**

```text
Copy Nginx configuration → ok
Restart Nginx → did not run
```

## 4. **Difference between `--check`, `--diff`, and `-v`**

| Option    | Purpose                                                                          |
| --------- | -------------------------------------------------------------------------------- |
| `--check` | Previewed what would change without applying the changes.                        |
| `--diff`  | Displayed the actual differences between the current and expected file contents. |
| `-v`      | Displayed more detailed information while the playbook was running.              |

`--check --diff` was useful for production because it allowed changes to be previewed and file differences to be inspected before applying them.

