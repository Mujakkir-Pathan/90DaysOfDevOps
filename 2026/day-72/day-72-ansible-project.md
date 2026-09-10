# Day 72 -- Ansible Project: Automate Docker and Nginx Deployment

## Task
Five days of Ansible -- inventory, ad-hoc commands, playbooks, modules, handlers, variables, facts, conditionals, loops, roles, templates, Galaxy, and Vault. Today I put it all together and built a production-style deployment.

I automated a complete deployment: installed Docker, pulled and ran a containerized application, set up Nginx as a reverse proxy in front of it, and managed everything through Ansible roles.

---
## All ansible playbooks files

[ansible playbooks files](https://github.com/Mujakkir-Pathan/ansible-playbooks/tree/main/day-5)

---

## All sceenshots

[screenshots of task](screenshots/)

---

## Challenge Tasks

### Task 1: Plan the Project Structure
The complete Ansible project layout was created:

```text
ansible-docker-project/
  ansible.cfg
  inventory.ini
  site.yml
  group_vars/
    all.yml
    web/
      vault.yml
  roles/
    common/
      tasks/main.yml
    docker/
      tasks/main.yml
      handlers/main.yml
      defaults/main.yml
    nginx/
      tasks/main.yml
      templates/
        nginx.conf.j2
        app-proxy.conf.j2
      handlers/main.yml
      defaults/main.yml
```

The role skeletons were generated with:

```bash
mkdir -p ansible-docker-project/roles
cd ansible-docker-project
ansible-galaxy init roles/common
ansible-galaxy init roles/docker
ansible-galaxy init roles/nginx
```

The `ansible.cfg` and `inventory.ini` were set up using the Day 68 configuration.

---

### Task 2: Build the Common Role
The `common` role was configured to run on every server.

`roles/common/tasks/main.yml` was configured to:
- Update the package cache with `apt`.
- Install the common packages.
- Set the hostname.
- Set the timezone.
- Create the `deploy` user with the `sudo` group.

All common tasks were tagged with `common`.

`group_vars/all.yml` was configured with:

```yaml
---
timezone: Asia/Kolkata
project_name: devops-app
app_env: development
common_packages:
  - vim
  - curl
  - wget
  - git
  - htop
  - tree
  - jq
  - unzip
```

The common role ran successfully on `App-server`, `DB-server`, and `web-server`.

---

### Task 3: Build the Docker Role
The Docker role was configured to install Docker, start the service, pull the application image, run the container, and verify the application.

`roles/docker/defaults/main.yml` was configured with:

```yaml
---
docker_app_image: nginx
docker_app_tag: latest
docker_app_name: myapp
docker_app_port: 8080
docker_container_port: 80
```

The Docker role tasks were configured to:
1. Install Docker dependencies using `apt`.
2. Add the Docker CE repository.
3. Install Docker CE.
4. Start and enable the Docker service.
5. Add the `deploy` user to the `docker` group.
6. Install Docker Compose through the Docker Compose plugin.
7. Log in to Docker Hub using Vault-encrypted credentials.
8. Pull the application image.
9. Run the application container with `8080:80`.
10. Verify the container by checking `http://localhost:8080`.

All Docker tasks were tagged with `docker`.

The `community.docker` collection was installed and available.

The Docker role was successfully applied to `web-server`.

---

### Task 4: Build the Nginx Role
The Nginx role was configured to install Nginx and use it as a reverse proxy to the Docker container.

`roles/nginx/defaults/main.yml` was configured with:

```yaml
---
nginx_http_port: 80
nginx_upstream_port: 8080
nginx_server_name: "_"
```

The Nginx role was configured to:
1. Install Nginx.
2. Remove the default Nginx site configuration.
3. Deploy the main Nginx configuration from `nginx.conf.j2`.
4. Deploy the reverse proxy configuration from `app-proxy.conf.j2`.
5. Test the Nginx configuration with `nginx -t`.
6. Start and enable Nginx.
7. Reload Nginx through a handler when configuration changes occurred.

The reverse proxy template configured Nginx port 80 to forward requests to `127.0.0.1:8080`.

The Nginx handlers were configured with `Reload Nginx` and `Restart Nginx`.

---

### Task 5: Encrypt Docker Hub Credentials with Vault
The vault file was created with:

```bash
ansible-vault create group_vars/web/vault.yml
```

The Docker Hub credentials were stored inside the encrypted file.

The Vault file was successfully opened with:

```bash
ansible-vault view group_vars/web/vault.yml
```

A `.vault_pass` file was created and secured with:

```bash
chmod 600 .vault_pass
```

`.vault_pass` was added to `.gitignore`.

The Vault password file was referenced in `ansible.cfg` with:

```ini
vault_password_file = .vault_pass
```

---

### Task 6: Write the Master Playbook and Deploy
The master `site.yml` was created:

```yaml
---
- name: Apply common configuration
  hosts: all
  become: true
  roles:
    - common
  tags: common

- name: Install Docker and run containers
  hosts: web
  become: true
  roles:
    - docker
  tags: docker

- name: Configure Nginx reverse proxy
  hosts: web
  become: true
  roles:
    - nginx
  tags: nginx
```

The full deployment was run successfully with:

```bash
ansible-playbook site.yml
```

The final successful deployment recap showed:

```text
App-server                 : ok=6    changed=1    unreachable=0    failed=0    skipped=0    ignored=0
DB-server                  : ok=6    changed=1    unreachable=0    failed=0    skipped=0    ignored=0
web-server                 : ok=26   changed=9    unreachable=0    failed=0    skipped=0    ignored=0
```

The following tag commands were also executed successfully:

```bash
ansible-playbook site.yml --tags docker
ansible-playbook site.yml --tags nginx
ansible-playbook site.yml --skip-tags common
```

**Verify:**

1. Curl the server on port 8080 -- does the Docker container respond directly?

The Docker container responded directly on port 8080 with the Nginx welcome page.

2. Curl the server on port 80 -- does Nginx reverse proxy the request to the container?

Nginx successfully reverse proxied the request on port 80 and returned the container's Nginx welcome page.

3. Check `docker ps` on the server -- is the container running with the correct port mapping?

The container was running with the correct mapping:

```text
IMAGE          nginx:latest
STATUS         Up 38 minutes
PORTS          0.0.0.0:8080->80/tcp
NAMES          myapp
```

---

### Task 7: Bonus -- Deploy a Different App and Re-Run
The Docker image was changed from Nginx to Apache HTTP Server with:

```bash
ansible-playbook site.yml --tags docker -e "docker_app_image=httpd docker_app_tag=latest docker_app_name=apache-app"
```

The old `myapp` container was removed and the `apache-app` container was recreated.

The final container verification showed:

```text
IMAGE          httpd:latest
STATUS         Up
PORTS          0.0.0.0:8080->80/tcp
NAMES          apache-app
```

Nginx continued to proxy traffic to the new container. The request on port 80 returned:

```html
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<title>It works! Apache httpd</title>
</head>
<body>
<p>It works!</p>
</body>
</html>
```

The full playbook was then run again:

```bash
ansible-playbook site.yml
```

The final recap was:

```text
App-server                 : ok=6    changed=1    unreachable=0    failed=0    skipped=0    ignored=0
DB-server                  : ok=6    changed=1    unreachable=0    failed=0    skipped=0    ignored=0
web-server                 : ok=25   changed=4    unreachable=0    failed=0    skipped=0    ignored=0
```

The playbook completed with zero failures. The run was mostly `ok`, although four tasks still reported `changed`, so the final run demonstrated successful repeatable deployment but was not completely idempotent.

**Reflect and document:**

1. How many total tasks ran?

The final full playbook run executed 37 task executions across the three managed nodes:
- `App-server`: 6
- `DB-server`: 6
- `web-server`: 25

2. Map each Ansible concept to the day you learned it:

| Day | Concept Used |
|-----|-------------|
| 68 | Inventory, ad-hoc commands, SSH setup |
| 69 | Playbooks, modules, handlers |
| 70 | Variables, facts, conditionals, loops |
| 71 | Roles, templates, Galaxy, Vault |
| 72 | Everything combined in one project |

3. What would you add for production? (SSL with certbot, monitoring, log rotation, multi-container Compose)

- SSL with Certbot -- secure Nginx with HTTPS and manage SSL certificates.
- Monitoring -- monitor servers, Docker containers, Nginx, resources, and application health.
- Log rotation -- prevent logs from growing indefinitely and consuming disk space.
- Multi-container Compose -- manage multiple application containers and their dependencies together.

4. Clean up your EC2 instances when done. If you used Terraform: `terraform destroy`. If manual: terminate from the console.

The EC2 instances were to be cleaned up after completing the project by using Terraform destroy if provisioned with Terraform, or by terminating the instances manually from the AWS console.
