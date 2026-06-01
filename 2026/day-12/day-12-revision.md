# Day 12 - Revision (Days 01–11)

## Goal

Review and reinforce the Linux and DevOps fundamentals learned during Days 01–11.

---

## Revision Notes

### 1. Mindset & Learning Plan Review

- Continue following the 90 Days of DevOps roadmap consistently.
- Focus on building strong Linux fundamentals before moving to advanced DevOps tools.
- Maintain the habit of documenting daily learnings on GitHub.
- Improve command-line speed and confidence through daily practice.

---

### 2. Processes & Services Review

#### Commands Revisited

```bash
ps aux
systemctl status ssh
journalctl -u ssh
```

#### Observations

- `ps aux` displays all running processes on the system.
- `systemctl status ssh` shows whether the SSH service is active and running.
- `journalctl -u ssh` helps inspect SSH service logs and troubleshoot issues.

---

### 3. File Skills Practice

#### Commands Practiced

```bash
echo "Revision practice" >> notes.txt
chmod 755 script.sh
cp notes.txt backup-notes.txt
```

#### What I Observed

- `echo >>` appends content without overwriting existing data.
- `chmod` changes file permissions for users, groups, and others.
- `cp` creates a duplicate file while preserving the original.

![Screenshot](screenshot/t3.png)
---

### 4. Cheat Sheet Refresh

#### Top 5 Commands I Would Use During an Incident

```bash
grep
systemctl status
journalctl
ps aux
ls -l
```

#### Why

- `grep` quickly filters important information from large outputs.
- `systemctl status` checks service health instantly.
- `journalctl` helps investigate logs and failures.
- `ps aux` identifies running processes.
- `ls -l` verifies permissions and ownership.

---

### 5. User & Group Management Review

#### Scenario Recreated

```bash
sudo useradd trainee
id trainee

sudo chown trainee:trainee devops-file.txt

ls -l devops-file.txt
```

#### Verification

- Confirmed user creation using `id`.
- Verified ownership changes using `ls -l`.
- Understood the relationship between users, groups, and file ownership.

![Screenshot](screenshot/t5.png)---

## Mini Self-Check

### 1. Which 3 commands save you the most time right now, and why?

#### grep
Filters useful information quickly from large outputs.

#### systemctl status
Provides an instant health check of services.

#### ls -l
Shows permissions, ownership, and file details at a glance.

---

### 2. How do you check if a service is healthy?

```bash
systemctl status ssh
journalctl -u ssh
ps aux | grep ssh
```

These commands verify service status, logs, and running processes.

---

### 3. How do you safely change ownership and permissions without breaking access?

Example:

```bash
sudo chown mujakkir:mujakkir project-file.txt
chmod 644 project-file.txt
```

Always verify ownership and permissions using:

```bash
ls -l project-file.txt
```

---

### 4. What will you focus on improving in the next 3 days?

- Strengthen Linux command-line efficiency.
- Improve troubleshooting skills using logs and services.
- Practice more user, group, and permission management scenarios.
- Continue documenting all work professionally on GitHub.

---

## Key Takeaways

- Linux permissions and ownership are critical system administration skills.
- Service management becomes easier with `systemctl` and `journalctl`.
- User and group management helps secure systems properly.
- Consistent practice and documentation improve retention.
- Building strong Linux fundamentals is essential for a DevOps career.

---

## Progress Summary

✅ Linux Basics

✅ File Operations

✅ Permissions & Ownership

✅ Process Management

✅ Service Management

✅ User & Group Management

✅ Git & GitHub Documentation

Ready to move forward to Day 13.
