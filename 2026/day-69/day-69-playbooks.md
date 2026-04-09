# Day 69 -- Ansible Playbooks and Modules

## Task 1: Your First Playbook
Create `install-nginx.yml`:

```yaml
---
- name: Install and start Nginx on web servers
  hosts: web
  become: true

  tasks:`
    - name: Install Nginx
      yum:
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
        dest: /usr/share/nginx/html/index.html
```

(Use `apt` instead of `yum` if your instances run Ubuntu)

Run it:
```bash
ansible-playbook install-nginx.yml
```

   ![snapshot](images/1-a.png)

Read the output carefully -- every task shows `changed`, `ok`, or `failed`.

Now run it **again**. Notice that tasks show `ok` instead of `changed`. This is **idempotency** -- Ansible only makes changes when needed.

   ![snapshot](images/1-b.png)

**Verify:** Curl the web server's public IP. Do you see your custom page?

   ![snapshot](images/1-c.png)

---

## Task 2: Understand the Playbook Structure
Open your playbook and annotate each part in your notes:

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
   - `Play` 
     - Maps a set of tasks to a specific group of hosts.
     - Defines on which hosts to run against,  what variables to use, and which tasks/roles to execute.
   - `Task`
     - Single unit of work within a play.
     - Calls an ansible module to perform a specific action (eg. copy, service)

2. Can you have multiple plays in one playbook?
   - **YES**, a playbook can contain one or more plays.

3. What does `become: true` do at the play level vs the task level?
   - It executes command as root user.
   - At `play` level it applies to all tasks under that play.
   - At `task` level it applies to only that task.

4. What happens if a task fails -- do remaining tasks still run?
   - `No` it stops executing remaining task on the failed hosts.
   - But on other hosts where task has not failed yet, it continues running.

---

## Task 3: Learn the Essential Modules
Practice each of these modules by writing a playbook called `essential-modules.yml` with multiple tasks:

1. **`yum`/`apt`** -- Install and remove packages:
```yaml
- name: Install multiple packages
  yum:
    name:
      - git
      - curl
      - wget
      - tree
    state: present
```

2. **`service`** -- Manage services:
```yaml
- name: Ensure Nginx is running
  service:
    name: nginx
    state: started
    enabled: true
```

3. **`copy`** -- Copy files from control node to managed nodes:
```yaml
- name: Copy config file
  copy:
    src: files/app.conf
    dest: /etc/app.conf
    owner: root
    group: root
    mode: '0644'
```

4. **`file`** -- Create directories and manage permissions:
```yaml
- name: Create application directory
  file:
    path: /opt/myapp
    state: directory
    owner: ec2-user
    mode: '0755'
```

5. **`command`** -- Run a command (no shell features):
```yaml
- name: Check disk space
  command: df -h
  register: disk_output

- name: Print disk space
  debug:
    var: disk_output.stdout_lines
```

6. **`shell`** -- Run a command with shell features (pipes, redirects):
```yaml
- name: Count running processes
  shell: ps aux | wc -l
  register: process_count

- name: Show process count
  debug:
    msg: "Total processes: {{ process_count.stdout }}"
```

7. **`lineinfile`** -- Add or modify a single line in a file:
```yaml
- name: Set timezone in environment
  lineinfile:
    path: /etc/environment
    line: 'TZ=Asia/Kolkata'
    create: true
```

Create a `files/` directory with a sample `app.conf` file for the copy task. Run the playbook against all servers.

   ![snapshot](images/3.png)

**Document:** What is the difference between `command` and `shell`? When should you use each?
   - **Command**
     - Runs simple commands directly without shell
     - Use command by default
   - **Shell**
     - Supports pipelines and redirects, runs through `/bin/sh` shell
     - Only use shell when you absolutely need shell features

---

## Task 4: Handlers -- Restart Services Only When Needed
Handlers are tasks that run only when triggered by a `notify`. This avoids unnecessary service restarts.

Create `nginx-config.yml`:
```yaml
---
- name: Configure Nginx with a custom config
  hosts: web
  become: true

  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present

    - name: Deploy Nginx config
      copy:
        src: files/nginx.conf
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Restart Nginx

    - name: Deploy custom index page
      copy:
        content: "<h1>Managed by Ansible</h1><p>Server: {{ inventory_hostname }}</p>"
        dest: /usr/share/nginx/html/index.html

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

Create `files/nginx.conf` with a basic Nginx config.

Run the playbook:
- First run: handler triggers because the config file is new
- Second run: handler does NOT trigger because nothing changed

   - Handler triggers for first time

   ![snapshot](images/4-a.png)

   -  Handlre does not trigger because no new chnages, config already deployed

   ![snapshot](images/4-b.png)

**Verify:** Run it twice and compare the output. Does the handler run both times?
   - Handlre runs only once when changed config.

---

## Task 5: Dry Run, Diff, and Verbosity
Before running playbooks on production, always preview changes first.

1. **Dry run (check mode)** -- shows what would change without changing anything:
```bash
ansible-playbook install-nginx.yml --check
```

   ![snapshot](images/5-b.png)

2. **Diff mode** -- shows the actual file differences:
```bash
ansible-playbook nginx-config.yml --check --diff
```

   ![snapshot](images/5-b.png)

3. **Verbosity** -- increase output detail for debugging:
```bash
ansible-playbook install-nginx.yml -v       # verbose
ansible-playbook install-nginx.yml -vv      # more verbose
ansible-playbook install-nginx.yml -vvv     # connection debugging
```

4. **Limit to specific hosts:**
```bash
ansible-playbook install-nginx.yml --limit web-server
```

   ![snapshot](images/5-c.png)

5. **List what would be affected without running:**
```bash
ansible-playbook install-nginx.yml --list-hosts
ansible-playbook install-nginx.yml --list-tasks
```

   ![snapshot](images/5-d.png)


**Document:** Why is `--check --diff` the most important flag combination for production use?
   - Because it shows exactly what changes are going to be made.
   - What would be removed and what would be added.

---

## Task 6: Multiple Plays in One Playbook
Write `multi-play.yml` with separate plays for each server group:

```yaml
---
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      yum:
        name: nginx
        state: present
    - name: Start Nginx
      service:
        name: nginx
        state: started
        enabled: true

- name: Configure app servers
  hosts: app
  become: true
  tasks:
    - name: Install Node.js dependencies
      yum:
        name:
          - gcc
          - make
        state: present
    - name: Create app directory
      file:
        path: /opt/app
        state: directory
        mode: '0755'

- name: Configure database servers
  hosts: db
  become: true
  tasks:
    - name: Install MySQL client
      yum:
        name: mysql
        state: present
    - name: Create data directory
      file:
        path: /var/lib/appdata
        state: directory
        mode: '0700'
```

Run it:
```bash
ansible-playbook multi-play.yml
```

   ![snapshot](images/6-a.png)

Watch the output -- each play targets a different group, and tasks run only on the relevant hosts.

**Verify:** Is Nginx only installed on web servers? Is MySQL only on db servers?
* **YES**

---

- Your first playbook with annotations explaining each section

```sh
- name: Install and start Nginx on web servers  # Play Name - targets a group of hosts
  hosts: web                                    # Which inventory group to run on
  become: true                                  # Execute tasks as root user

  tasks:                                        # List of tasks
    - name: Install Nginx                       # Task 1: Install Nginx package
      apt:                                      # Module: apt package manager
        name: nginx                             # Package name
        state: present                          # Desired state (present/absent)

    - name: Start and enable Nginx              # Task 2: Ensure service is running
      service:                                  # Module: systemd service 
        name: nginx                             # Service name
        state: started                          # Ensure service is running
        enabled: true                           # Start on boot (persist across reboots)

    - name: Create a custom index page          # Task 3: Deploy static content
      copy:                                     # Module: copy files to remote host
        content: "<h1>Deployed by Ansible - TerraWeek Server</h1><br>"  # File Content
        dest: /var/www/html/index.html          # Destination on remote host
```

- All seven module examples with what each does
   1. **`yum`/`apt`** -- Install and remove packages:
    ```yaml
    - name: Install multiple packages
    yum:
      name:
        - git
        - curl
        - wget
        - tree
      state: present
    ```
   2. **`service`** -- Manage services:
    ```yaml
    - name: Ensure Nginx is running
    service:
      name: nginx
      state: started
      enabled: true
    ```

   3. **`copy`** -- Copy files from control node to managed nodes:
    ```yaml
    - name: Copy config file
    copy:
      src: files/app.conf
      dest: /etc/app.conf
      owner: root
      group: root
      mode: '0644'
    ```

   4. **`file`** -- Create directories and manage permissions:
    ```yaml
    - name: Create application directory
    file:
      path: /opt/myapp
      state: directory
      owner: ec2-user
      mode: '0755'
    ```

    5. **`command`** -- Run a command (no shell features):
    ```yaml
    - name: Check disk space
    command: df -h
    register: disk_output

    - name: Print disk space
    debug:
      var: disk_output.stdout_lines
    ```

    6. **`shell`** -- Run a command with shell features (pipes, redirects):
    ```yaml
    - name: Count running processes
    shell: ps aux | wc -l
    register: process_count

    - name: Show process count
    debug:
      msg: "Total processes: {{ process_count.stdout }}"
    ```

    7. **`lineinfile`** -- Add or modify a single line in a file:
    ```yaml
    - name: Set timezone in environment
    lineinfile:
      path: /etc/environment
      line: 'TZ=Asia/Kolkata'
      create: true
    ```

- Difference between `--check`, `--diff`, and `-v`
  - `--check` : Dry run (Shows what would change without actually making any changes.)
  - `--diff` : Shows actual differences in files/templates before and after changes.
  - `-v` : Verbose output (Shows detailed execution logs)

---
