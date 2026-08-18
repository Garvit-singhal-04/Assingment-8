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
<img width="668" height="534" alt="image" src="https://github.com/user-attachments/assets/135d7a0b-346a-45d3-ad6b-93deeaa09531" />

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
<img width="639" height="208" alt="image" src="https://github.com/user-attachments/assets/8c965008-7442-425a-923f-b0c9addca9ae" />

### Test connectivity

```bash
ansible managed -i inventory -m ping
```
<img width="657" height="229" alt="image" src="https://github.com/user-attachments/assets/fc1578f3-53b3-4fa9-b770-23a5121d090b" />

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
ansible managed -i inventory -m group -a "name=dev-team state=present" --become
```
<img width="1110" height="246" alt="image" src="https://github.com/user-attachments/assets/77403121-4e37-48fd-bc55-81662375d60b" />

Create the DevOps group:

```bash
ansible managed -i inventory -m group -a "name=devops-team state=present" --become
```
<img width="1106" height="260" alt="image" src="https://github.com/user-attachments/assets/bd552d90-c1f9-42fb-9d8a-8ead04ab61a8" />

Create the Admin group:

```bash
ansible managed -i inventory -m group -a "name=admin-group state=present" --become
```
<img width="1106" height="260" alt="image" src="https://github.com/user-attachments/assets/93279ec4-88ea-4017-81e2-e3c68c50d9c9" />

### Verify

```bash
ansible managed -i inventory -m command -a "getent group dev-team devops-team admin-group"
```
<img width="1104" height="175" alt="image" src="https://github.com/user-attachments/assets/40b78ec3-4335-40a0-ae8e-0e5d7ff477bf" />

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
ansible managed -i inventory -m user -a "name=dev1 uid=2000 group=dev-team" --become
```
<img width="1102" height="371" alt="image" src="https://github.com/user-attachments/assets/44592b05-a18e-49f3-82dd-46e76d8a7270" />

The same approach is used for all nine users.

### Verify

```bash
ansible managed -i inventory -m shell -a "getent passwd | grep -E '^(dev1|dev2|dev3|devops1|devops2|devops3|admin1|admin2|admin3):'"
```
<img width="1101" height="301" alt="image" src="https://github.com/user-attachments/assets/c8a94730-ca93-4afa-b3ed-9aa1edbc2b62" />

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
ansible managed -i inventory -m apt -a "name=zsh state=present" --become
```
<img width="1120" height="578" alt="image" src="https://github.com/user-attachments/assets/8490c76d-54e8-4832-a65d-2ba351132ee1" />

Configure DevOps users:

```bash
ansible managed -i inventory -m user -a "name=devops1 shell=/usr/bin/zsh" --become
```
<img width="1137" height="362" alt="image" src="https://github.com/user-attachments/assets/e1f6ee52-cfa4-4c5b-bc04-f5aa22167bf9" />

The same shell configuration is applied to `devops2` and `devops3`.

### Verify

```bash
ansible managed -i inventory -m command -a "getent passwd"
```
<img width="495" height="172" alt="image" src="https://github.com/user-attachments/assets/1d810e20-d07b-4969-ae91-521dce606e09" />

---

# 6. Password Expiry Policy

Password aging is configured as:

```text
Minimum password age: 1 day
Maximum password age: 90 days
```

Example:

```bash
ansible managed -i inventory -m user -a "name=dev1 password_expire_min=1 password_expire_max=90" --become
```
<img width="1193" height="363" alt="image" src="https://github.com/user-attachments/assets/709d1d21-af9a-4c39-ab08-c5780495054f" />

### Verify

```bash
ansible managed -i inventory -m command -a "chage -l dev1"
```
<img width="1183" height="233" alt="image" src="https://github.com/user-attachments/assets/cf63ba09-544a-472b-a7c6-be14142ddca3" />

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
```bash
ansible managed -i inventory -m copy -a "content='%admin-group ALL=(ALL) ALL\n' dest=/etc/sudoers.d/admin-group owner=root group=root mode=0440" --become
```
<img width="1202" height="407" alt="image" src="https://github.com/user-attachments/assets/5b4e62eb-f48c-4a66-9e62-b7132a7f6039" />


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
<img width="549" height="207" alt="image" src="https://github.com/user-attachments/assets/70749878-7bf3-41ed-92f6-9cb9846d634b" />

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
ansible managed -i inventory -m file -a "path=/home/dev1/workspace state=directory owner=dev1 group=dev-team mode=0750" --become
```
<img width="1193" height="353" alt="image" src="https://github.com/user-attachments/assets/da4d93f7-5377-42df-8dc5-4ed22779aa23" />

All nine users receive a `workspace` directory.

### Verify

```bash
ansible managed -i inventory -m command -a "find /home -maxdepth 2 -type d -name workspace" --become
```
<img width="1196" height="285" alt="image" src="https://github.com/user-attachments/assets/89e775fb-2c83-4e73-86bc-f5b2822e92fe" />

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
ansible managed -i inventory -m file -a "path=/project-management/teams/dev-team state=directory" --become
```
<img width="1198" height="336" alt="image" src="https://github.com/user-attachments/assets/8253539a-1e35-48c9-a463-9b18be530276" />


# 8.3 Project Directories
```bash
ansible managed -i inventory -m file -a "path=/project-management/projects state=directory" --become
```
<img width="1198" height="336" alt="image" src="https://github.com/user-attachments/assets/441ff1fc-9b57-4f62-9bd1-1450a77133fc" />

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

<img width="1188" height="569" alt="image" src="https://github.com/user-attachments/assets/28c1f0fd-a132-4f41-a9d6-be01fa105c07" />


---

# 8.4 Shared, Archive and Admin Directories

Create:

```text
/project-management/shared
/project-management/archive
/project-management/admin
```
```bash
ansible managed -i inventory -m file -a "path=/project-management/shared state=directory" --become
```
<img width="1198" height="336" alt="image" src="https://github.com/user-attachments/assets/2b4c60df-9913-45ce-9edc-a7e26b19b98e" />


```bash
ansible managed -i inventory -m file -a "path=/project-management/archive state=directory" --become
```
<img width="1198" height="336" alt="image" src="https://github.com/user-attachments/assets/f83c4f86-695e-42bf-98ee-cef831096ea4" />

```bash
ansible managed -i inventory -m file -a "path=/project-management/admin state=directory" --become
```
<img width="1183" height="334" alt="image" src="https://github.com/user-attachments/assets/92f28250-b299-448e-965d-30c84e585b3b" />


---

# 9. Security & Permission Matrix


The exact Linux implementation is configured below.
---

# 10. Personal Workspace Permissions

Requirement:

```text
dev1      → Full
dev-team  → Read
others    → None
```

Configure:

```bash
ansible managed -i inventory -m file \-a "path=/home/dev1/workspace owner=dev1 group=dev-team mode=0750" --become
```
## Given during the creation of workspaces

The same model is applied to all nine users.
<img width="604" height="136" alt="image" src="https://github.com/user-attachments/assets/fc5b8835-e865-4e2c-985e-6a504eb59cd5" />

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
ansible managed -i inventory -m file -a "path=/project-management/teams/dev-team owner=root group=dev-team mode=0775" --become
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
<img width="1203" height="171" alt="image" src="https://github.com/user-attachments/assets/ee478ce9-8b05-4017-9017-64c92877f05e" />


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
ansible managed -i inventory -m file -a "path=/project-management/projects/WebApp owner=dev1 group=dev-team mode=0775" --become
```

### API

```bash
ansible managed -i inventory -m file -a "path=/project-management/projects/API owner=devops1 group=devops-team mode=0775" --become
```

### Mobile

```bash
ansible managed -i inventory -m file -a "path=/project-management/projects/Mobile owner=dev2 group=dev-team mode=0775" --become
```

<img width="1203" height="171" alt="image" src="https://github.com/user-attachments/assets/c81686f9-9c75-4c73-b41c-1fd01997b37d" />


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

<img width="1203" height="134" alt="image" src="https://github.com/user-attachments/assets/08824238-3f89-4c6c-a4be-02b218882332" />


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

<img width="1203" height="134" alt="image" src="https://github.com/user-attachments/assets/98a385d6-d113-409c-aaae-251e1e28d095" />


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

<img width="1203" height="134" alt="image" src="https://github.com/user-attachments/assets/c55d7fb0-f608-4c09-99d6-d7c74a72850a" />


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

