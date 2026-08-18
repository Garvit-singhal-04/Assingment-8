Yes. Since this is an **assignment submission with screenshots**, the README should work more like a **build log / implementation guide** rather than a generic project description.

The flow should be:

```text
1. Environment
   ↓
2. Ansible connectivity
   ↓
3. Groups
   ↓
4. Users + UIDs
   ↓
5. Shells
   ↓
6. Password policy
   ↓
7. Sudo
   ↓
8. Directory creation
   ↓
9. Permission matrix
   ↓
10. Testing
```

For each step, you can place **your command screenshot immediately after the command**, followed by the expected/result screenshot.

Here's a README structure I recommend using.

---

# Ansible Assignment 1 — UserManager

## Project Management System

This project implements a Linux-based User and Project Management System using **Ansible**.

The system is built incrementally on an Ubuntu EC2 instance. Each stage is verified before moving to the next stage.

---

# 1. Environment Setup

## Architecture

```text
┌──────────────────────────┐
│     Control Node         │
│                          │
│  Ubuntu / WSL            │
│  Ansible                 │
│  inventory               │
└────────────┬─────────────┘
             │
             │ SSH
             ▼
┌──────────────────────────┐
│      Managed Node        │
│                          │
│      Ubuntu EC2          │
│                          │
│  Users                   │
│  Groups                  │
│  Directories             │
│  Permissions             │
│  Sudo                    │
└──────────────────────────┘
```

### Control Node

The control node is the local Ubuntu/WSL machine from which Ansible commands are executed.

### Managed Node

An Ubuntu EC2 instance is used as the managed server.

### Screenshot

> **Screenshot 1:** EC2 instance running and SSH connection.

---

# 2. Ansible Inventory

Create the inventory file:

```ini
[managed]
server1 ansible_host=<EC2_PUBLIC_IP>

[managed:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/path/to/key.pem
```

The inventory defines the **server**, not the Linux users.

### Test connectivity

```bash
ansible managed -i inventory -m ping
```

Expected:

```text
server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Screenshot

> **Screenshot 2:** Inventory file.

> **Screenshot 3:** Successful `ansible ping`.

---

# 3. Create Team Groups

The required Linux groups are:

```text
dev-team
devops-team
admin-group
```

Create the Development group:

```bash
ansible managed -i inventory -m group \
-a "name=dev-team state=present" --become
```

Create the DevOps group:

```bash
ansible managed -i inventory -m group \
-a "name=devops-team state=present" --become
```

Create the Admin group:

```bash
ansible managed -i inventory -m group \
-a "name=admin-group state=present" --become
```

### Verify

```bash
ansible managed -i inventory -m command \
-a "getent group dev-team devops-team admin-group"
```

### Result

```text
dev-team:x:...
devops-team:x:...
admin-group:x:...
```

### Screenshots

> **Screenshot 4:** Commands creating the three groups.

> **Screenshot 5:** `getent group` showing all three groups.

---

# 4. Create Users

Nine users are required.

## User Mapping

| Team        | Users                     | UID       |
| ----------- | ------------------------- | --------- |
| dev-team    | dev1, dev2, dev3          | 2000–2002 |
| devops-team | devops1, devops2, devops3 | 2003–2005 |
| admin-group | admin1, admin2, admin3    | 2006–2008 |

Example:

```bash
ansible managed -i inventory -m user \
-a "name=dev1 uid=2000 group=dev-team" --become
```

The same approach is used for all nine users.

### Verify

```bash
ansible managed -i inventory -m command \
-a "getent passwd | grep -E '^(dev1|dev2|dev3|devops1|devops2|devops3|admin1|admin2|admin3):'"
```

### Verify individual user

```bash
ansible managed -i inventory -m command \
-a "id dev1"
```

Expected:

```text
uid=2000(dev1) gid=...(dev-team) groups=...(dev-team)
```

### Screenshots

> **Screenshot 6:** User creation commands.

> **Screenshot 7:** All nine users in `/etc/passwd`.

> **Screenshot 8:** `id dev1` showing UID and group.

---

# 5. Configure Login Shells

The shell policy is:

```text
dev-team       → /bin/bash
devops-team    → /usr/bin/zsh
admin-group    → /bin/bash
```

Zsh was installed because it was not initially available on the EC2.

```bash
ansible managed -i inventory -m apt \
-a "name=zsh state=present" --become
```

Configure DevOps users:

```bash
ansible managed -i inventory -m user \
-a "name=devops1 shell=/usr/bin/zsh" --become
```

The same shell configuration is applied to `devops2` and `devops3`.

### Verify

```bash
ansible managed -i inventory -m command \
-a "getent passwd devops1"
```

Expected final field:

```text
/usr/bin/zsh
```

### Screenshots

> **Screenshot 9:** Zsh installation.

> **Screenshot 10:** DevOps user shell configuration.

> **Screenshot 11:** `getent passwd` showing `/usr/bin/zsh`.

---

# 6. Password Expiry Policy

Password aging is configured as:

```text
Minimum password age: 1 day
Maximum password age: 90 days
```

Example:

```bash
ansible managed -i inventory -m user \
-a "name=dev1 password_expire_min=1 password_expire_max=90" \
--become
```

### Verify

```bash
ansible managed -i inventory -m command \
-a "chage -l dev1"
```

Expected:

```text
Minimum number of days between password change : 1
Maximum number of days between password change : 90
```

### Screenshots

> **Screenshot 12:** Password policy configuration.

> **Screenshot 13:** `chage -l dev1` showing the expiry policy.

---

# 7. Configure Sudo Access

Sudo access is assigned to:

```text
devops-team
admin-group
```

Development users do not receive sudo privileges.

## DevOps sudo rule

```text
%devops-team ALL=(ALL) ALL
```

## Admin sudo rule

```text
%admin-group ALL=(ALL) ALL
```

Configuration is stored in:

```text
/etc/sudoers.d/
```

### Verify

```bash
sudo -l -U devops1
```

```bash
sudo -l -U admin1
```

And verify that a development user does not have the same sudo rule:

```bash
sudo -l -U dev1
```

### Screenshots

> **Screenshot 14:** Sudo configuration files.

> **Screenshot 15:** `sudo -l -U devops1`.

> **Screenshot 16:** `sudo -l -U admin1`.

> **Screenshot 17:** `sudo -l -U dev1`.

---

# 8. Create Directory Structure

The required structure is:

```text
/home/
├── dev1/workspace
├── dev2/workspace
├── dev3/workspace
├── devops1/workspace
├── devops2/workspace
├── devops3/workspace
├── admin1/workspace
├── admin2/workspace
└── admin3/workspace

/project-management/
├── teams/
│   ├── dev-team/
│   ├── devops-team/
│   └── admin-group/
│
├── projects/
│   ├── WebApp/
│   ├── API/
│   └── Mobile/
│
├── shared/
├── archive/
└── admin/
```

---

## 8.1 Personal Workspaces

Create workspaces inside each user's home directory.

Example:

```bash
ansible managed -i inventory -m file \
-a "path=/home/dev1/workspace state=directory" \
--become
```

All nine users receive a `workspace` directory.

### Verify

```bash
ansible managed -i inventory -m command \
-a "find /home -maxdepth 2 -type d -name workspace" \
--become
```

### Screenshot

> **Screenshot 18:** All nine workspace directories.

---

# 8.2 Team Directories

Create:

```text
/project-management/teams/dev-team
/project-management/teams/devops-team
/project-management/teams/admin-group
```

Example:

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/teams/dev-team state=directory" \
--become
```

### Screenshot

> **Screenshot 19:** Team directories created.

---

# 8.3 Project Directories

Create:

```text
WebApp
API
Mobile
```

under:

```text
/project-management/projects/
```

### Screenshot

> **Screenshot 20:** Project directories.

---

# 8.4 Shared, Archive and Admin Directories

Create:

```text
/project-management/shared
/project-management/archive
/project-management/admin
```

### Screenshot

> **Screenshot 21:** Complete `/project-management` directory structure.

---

# 9. Security & Permission Matrix

The required permission model is:

| Resource           | Owner | Team       | Others     |
| ------------------ | ----- | ---------- | ---------- |
| Personal Workspace | Full  | Read       | None       |
| Team Directory     | Full  | —          | Read       |
| Project Directory  | Full  | Read/Write | Read       |
| Shared Resources   | —     | Read/Write | Read/Write |
| Archive            | Full  | Read       | Read       |
| Admin Area         | Full  | Admin only | None       |

The exact Linux implementation is configured below.

---

# 10. Personal Workspace Permissions

Example:

```text
/home/dev1/workspace
```

Requirement:

```text
dev1      → Full
dev-team  → Read
others    → None
```

Configure:

```bash
ansible managed -i inventory -m file \
-a "path=/home/dev1/workspace owner=dev1 group=dev-team mode=0750" \
--become
```

Result:

```text
0750

dev1       → rwx
dev-team   → r-x
others     → ---
```

The same model is applied to all nine users.

### Screenshot

> **Screenshot 22:** `ls -ld /home/dev1/workspace`.

---

# 11. Team Directory Permissions

Example:

```text
/project-management/teams/dev-team
```

Requirement:

```text
dev-team → Full
other teams → Read-only
```

Configure:

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/teams/dev-team owner=root group=dev-team mode=0775" \
--become
```

Result:

```text
root       → rwx
dev-team   → rwx
others     → r-x
```

Repeat for:

```text
devops-team
admin-group
```

### Screenshot

> **Screenshot 23:** Team directory permissions.

---

# 12. Project Directory Permissions

Project assignment used:

```text
WebApp
Lead: dev1
Team: dev-team

API
Lead: devops1
Team: devops-team

Mobile
Lead: dev2
Team: dev-team
```

### WebApp

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/projects/WebApp owner=dev1 group=dev-team mode=0775" \
--become
```

### API

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/projects/API owner=devops1 group=devops-team mode=0775" \
--become
```

### Mobile

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/projects/Mobile owner=dev2 group=dev-team mode=0775" \
--become
```

### Screenshot

> **Screenshot 24:** Project directory ownership and permissions.

---

# 13. Shared Resources Permissions

The shared directory must be writable by all teams.

For the simplified lab implementation:

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/shared owner=root group=root mode=0777" \
--become
```

This gives all users:

```text
rwx
```

### Important

`0777` is used here for simplicity in the controlled assignment environment.

In production, a dedicated shared group or ACL would be preferable because `0777` gives write access to all users on the system.

### Screenshot

> **Screenshot 25:** Shared directory permissions.

---

# 14. Archive Permissions

Archive requirement:

```text
All users → Read-only
```

Configure:

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/archive owner=root group=root mode=0755" \
--become
```

Users can:

* enter the directory
* list files
* read accessible files

Users cannot create or delete files in the archive directory.

### Screenshot

> **Screenshot 26:** Archive permissions.

---

# 15. Admin Area Permissions

Requirement:

```text
Only admin-group → Full access
```

Configure:

```bash
ansible managed -i inventory -m file \
-a "path=/project-management/admin owner=root group=admin-group mode=0770" \
--become
```

Result:

```text
root         → rwx
admin-group  → rwx
others       → ---
```

### Screenshot

> **Screenshot 27:** Admin directory permissions.

---

# 16. Permission Testing

The configuration is tested using different users.

## Development user

Test:

```bash
sudo -u dev1 ls /project-management/teams/dev-team
```

Development user should have full team-directory access.

---

## DevOps user

Test:

```bash
sudo -u devops1 ls /project-management/teams/dev-team
```

DevOps users should have read/traverse access but should not be able to create files there.

---

## Admin user

Test:

```bash
sudo -u admin1 ls /project-management/admin
```

Admin users should have full access.

---

## Non-admin user

Test:

```bash
sudo -u dev1 ls /project-management/admin
```

Expected:

```text
Permission denied
```

### Screenshots

> **Screenshot 28:** Development user permission test.

> **Screenshot 29:** DevOps user permission test.

> **Screenshot 30:** Admin user permission test.

> **Screenshot 31:** Non-admin denied from admin area.

---

# 17. Final Verification

Check all users:

```bash
ansible managed -i inventory -m command \
-a "getent passwd | grep -E '^(dev1|dev2|dev3|devops1|devops2|devops3|admin1|admin2|admin3):'"
```

Check all groups:

```bash
ansible managed -i inventory -m command \
-a "getent group dev-team devops-team admin-group"
```

Check complete project structure:

```bash
ansible managed -i inventory -m command \
-a "find /project-management -type d" --become
```

Check all workspaces:

```bash
ansible managed -i inventory -m command \
-a "find /home -maxdepth 2 -type d -name workspace" --become
```

Check permissions:

```bash
ansible managed -i inventory -m command \
-a "ls -ld /project-management/*" --become
```

---

# 18. Implementation Flow

The complete implementation was performed in the following order:

```text
EC2 Setup
    │
    ▼
Ansible Inventory
    │
    ▼
Ansible Ping
    │
    ▼
Create Groups
    │
    ▼
Create 9 Users
    │
    ▼
Assign Custom UIDs
    │
    ▼
Configure Login Shells
    │
    ▼
Configure Password Expiry
    │
    ▼
Configure Sudo
    │
    ▼
Create Personal Workspaces
    │
    ▼
Create Team Directories
    │
    ▼
Create Project Directories
    │
    ▼
Create Shared / Archive / Admin
    │
    ▼
Apply Permission Matrix
    │
    ▼
Test Access
    │
    ▼
Final Verification
```

---

# 19. Conclusion

The UserManager project demonstrates how Ansible can automate Linux system administration tasks including:

* User and group management
* UID assignment
* Login shell configuration
* Password aging
* Sudo configuration
* Directory creation
* Ownership management
* Linux permissions
* Team collaboration
* Project access
* Shared resources
* Administrative access control

The implementation was built incrementally and verified at each stage using Ansible commands and Linux verification utilities such as `id`, `getent`, `chage`, `ls`, and `find`.
