# Day 11 - Linux File Ownership (chown & chgrp)

## Objective
Learn how Linux file and directory ownership works using `chown` and `chgrp`.

---

# Users Created

- tokyo
- berlin
- nairobi
- professor

---

# Groups Created

- heist-team
- planners
- vault-team
- tech-team

---

# Files & Directories Created

## Files

- devops-file.txt
- project-config.yaml
- team-notes.txt
- bank-heist/access-codes.txt
- bank-heist/blueprints.pdf
- bank-heist/escape-plan.txt
- heist-project/plans/strategy.conf
- heist-project/vault/gold.txt

## Directories

- app-logs/
- bank-heist/
- heist-project/
- heist-project/plans/
- heist-project/vault/

---

# Task 1: Understanding Ownership

## Verification

Displayed files using:

```bash
ls -l
```

### Example Output

```text
drwxr-x--- 3 berlin berlin 4096 May 25 07:01 berlin
```

### Ownership Format

```text
permissions links owner group size date filename
```

### Difference Between Owner and Group

**Owner:**  
The owner is usually the user who creates the file or directory. The owner has primary control and can modify permissions and ownership.

**Group:**  
A group is a collection of users who share access to files and directories according to the assigned group permissions.

### Screenshot

![Task 1 Screenshot](screenshot/Home.png)

---

# Task 2: Basic chown Operations

## Activities Performed

- Created `devops-file.txt`
- Checked current ownership
- Changed owner from `mujakkir` to `tokyo`
- Changed owner from `tokyo` to `berlin`
- Verified ownership changes

### Result

```text
-rw-r--r-- 1 berlin mujakkir 0 May 30 16:57 devops-file.txt
```

### Screenshot

![Task 2 Screenshot](images/task2.png)

---

# Task 3: Basic chgrp Operations

## Activities Performed

- Created `team-notes.txt`
- Created group `heist-team`
- Changed file group to `heist-team`
- Verified group ownership

### Result

```text
-rw-r--r-- 1 mujakkir heist-team 0 May 31 08:50 team-notes.txt
```

### Screenshot

![Task 3 Screenshot](images/task3.png)

---

# Task 4: Combined Owner & Group Change

## File Ownership Change

Created:

```text
project-config.yaml
```

Changed:

- Owner → professor
- Group → heist-team

### Result

```text
-rw-r--r-- 1 professor heist-team 0 May 31 09:08 project-config.yaml
```

## Directory Ownership Change

Created:

```text
app-logs/
```

Changed:

- Owner → berlin
- Group → heist-team

### Result

```text
drwxr-xr-x 2 berlin heist-team 4096 May 31 14:49 app-logs
```

### Screenshot

![Task 4 Screenshot](images/task4.png)

---

# Task 5: Recursive Ownership

## Directory Structure

```text
heist-project/
├── vault/
│   └── gold.txt
└── plans/
    └── strategy.conf
```

## Activities Performed

- Created group `planners`
- Applied recursive ownership changes

Owner:

```text
professor
```

Group:

```text
planners
```

## Verification

```text
heist-project/:
drwxr-xr-x 2 professor planners plans
drwxr-xr-x 2 professor planners vault

heist-project/plans:
-rw-r--r-- 1 professor planners strategy.conf

heist-project/vault:
-rw-r--r-- 1 professor planners gold.txt
```

### Screenshot

![Task 5 Screenshot](images/task5.png)

---

# Task 6: Practice Challenge

## Created Users

- tokyo
- berlin
- nairobi

## Created Groups

- vault-team
- tech-team

## Directory Created

```text
bank-heist/
```

## Files Created

- access-codes.txt
- blueprints.pdf
- escape-plan.txt

## Ownership Assigned

| File | Owner | Group |
|------|--------|--------|
| access-codes.txt | tokyo | vault-team |
| blueprints.pdf | berlin | tech-team |
| escape-plan.txt | nairobi | vault-team |

## Verification

```text
-rw-r--r-- 1 tokyo   vault-team 0 access-codes.txt
-rw-r--r-- 1 berlin  tech-team  0 blueprints.pdf
-rw-r--r-- 1 nairobi vault-team 0 escape-plan.txt
```

### Screenshot

![Task 6 Screenshot](images/task6.png)

---

# Commands Used

## View Ownership

```bash
ls -l filename
```

## Change Owner Only

```bash
sudo chown newowner filename
```

## Change Group Only

```bash
sudo chgrp newgroup filename
```

## Change Owner and Group Together

```bash
sudo chown owner:group filename
```

## Recursive Ownership Change

```bash
sudo chown -R owner:group directory/
```

## Change Only Group Using chown

```bash
sudo chown :groupname filename
```

---

# What I Learned

- Understanding Linux file ownership.
- Difference between owner and group.
- Managing users and groups.
- Changing file ownership using `chown`.
- Changing file groups using `chgrp`.
- Changing owner and group together.
- Applying ownership recursively using the `-R` flag.
- Verifying ownership using `ls -l` and `ls -lR`.
