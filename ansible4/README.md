# Ansible Assignment 4 — System Manager Role

## Overview

This assignment creates a reusable Ansible role named `system_manager` for managing common resources on a Linux VM.

The role provides management for:

- Software/packages
- Users and groups
- Git repositories
- Directory structures

The configuration is kept in `defaults/main.yml`, while the actual management logic is separated into individual task files.

## Project Structure

```bash
Assignment4/
├── inventory
├── site.yml
└── roles/
    └── system_manager/
        ├── defaults/
        │   └── main.yml
        └── tasks/
            ├── main.yml
            ├── packages.yml
            ├── users.yml
            ├── directories.yml
            └── git.yml
```

Only the directories and files required for this implementation are included.

---

## 1. Configuration — `defaults/main.yml`

The `defaults/main.yml` file contains the resources that should be managed by the role.

```bash
---
system_packages:
  - git
  - curl
  - wget
  - tree
  - unzip

system_groups:
  - devops
  - developers

system_users:
  - name: devops1
    uid: 2001
    group: devops
    shell: /bin/bash

  - name: devops2
    uid: 2002
    group: devops
    shell: /bin/bash

  - name: developer1
    uid: 2003
    group: developers
    shell: /bin/bash

system_directories:
  - path: /opt/devops
    owner: root
    group: devops
    mode: "0775"

  - path: /opt/projects
    owner: root
    group: developers
    mode: "0775"

  - path: /opt/scripts
    owner: root
    group: devops
    mode: "0775"

  - path: /opt/logs
    owner: root
    group: devops
    mode: "0775"

git_repositories:
  - repo: "https://github.com/opstree/spring3hibernate.git"
    dest: /opt/projects/spring3hibernate
    version: master
```

The variables define **what** should be managed. The task files define **how** those resources are managed.

---

## 2. Package Management — `tasks/packages.yml`

The `packages.yml` file manages the installation of the required software.

```bash
---
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Install required software
  ansible.builtin.apt:
    name: "{{ system_packages }}"
    state: present
```

The package names are taken from the `system_packages` variable in `defaults/main.yml`.

---

## 3. User and Group Management — `tasks/users.yml`

The `users.yml` file creates the required groups and users.

```bash
---
- name: Create required groups
  ansible.builtin.group:
    name: "{{ item }}"
    state: present
  loop: "{{ system_groups }}"

- name: Create required users
  ansible.builtin.user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
    group: "{{ item.group }}"
    shell: "{{ item.shell }}"
    create_home: true
    state: present
  loop: "{{ system_users }}"
```

The users are created with their specified UID, primary group, shell, and home directory.

---

## 4. Directory Management — `tasks/directories.yml`

The `directories.yml` file creates the required directory structure and sets the required ownership and permissions.

```bash
---
- name: Create required directory structures
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: directory
    owner: "{{ item.owner }}"
    group: "{{ item.group }}"
    mode: "{{ item.mode }}"
  loop: "{{ system_directories }}"
```

The role creates the following directories:

```bash
/opt/devops
/opt/projects
/opt/scripts
/opt/logs
```

---

## 5. Git Repository Management — `tasks/git.yml`

The `git.yml` file manages the required Git repositories.

```bash
---
- name: Clone or update Git repositories
  ansible.builtin.git:
    repo: "{{ item.repo }}"
    dest: "{{ item.dest }}"
    version: "{{ item.version }}"
    update: true
  loop: "{{ git_repositories }}"

- name: Set Git repository ownership
  ansible.builtin.file:
    path: "{{ item.dest }}"
    owner: ubuntu
    group: ubuntu
    recurse: true
  loop: "{{ git_repositories }}"
```

The repository is cloned to the specified destination and updated when required.

The ownership task ensures that the `ubuntu` user can work with the repository without Git reporting a `dubious ownership` error.

---

## 6. Main Task File — `tasks/main.yml`

The `main.yml` file imports all the individual task files.

```bash
---
- name: Manage packages
  ansible.builtin.import_tasks: packages.yml

- name: Manage users and groups
  ansible.builtin.import_tasks: users.yml

- name: Manage directories
  ansible.builtin.import_tasks: directories.yml

- name: Manage Git repositories
  ansible.builtin.import_tasks: git.yml
```

This separates the role into different areas of system management and keeps the tasks easier to maintain.

---

## 7. Inventory

The `inventory` file contains the server that will be managed by Ansible.

```bash
[managed]
server1 ansible_host=<SERVER_IP> ansible_user=ubuntu ansible_ssh_private_key_file=<PATH_TO_KEY>
```

Replace `<SERVER_IP>` with the IP address of the managed VM and `<PATH_TO_KEY>` with the path to the SSH private key.

Example:

```bash
[managed]
server1 ansible_host=3.110.118.154 ansible_user=ubuntu ansible_ssh_private_key_file=/home/garvit/ansible4.1/ansible.pem
```

The actual IP address and private key path should be adjusted according to the environment.

---

## 8. Playbook — `site.yml`

The `site.yml` file applies the `system_manager` role to the managed server.

```bash
---
- name: Configure managed servers
  hosts: managed
  become: true

  roles:
    - system_manager
```

`become: true` is used because package installation and user/group management require administrative privileges.

---

# Running the Role

## 1. Test Ansible Connectivity

First, verify that Ansible can connect to the managed server.

```bash
ansible managed -i inventory -m ping
```

Expected result:

```bash
server1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

The Python interpreter warning may appear during this command. It does not indicate a failure if the result is `SUCCESS`.

---

## 2. Check Playbook Syntax

Run a syntax check before executing the playbook.

```bash
ansible-playbook -i inventory site.yml --syntax-check
```

Expected output:

```bash
playbook: site.yml
```

---

## 3. Run the Playbook

Execute the role:

```bash
ansible-playbook -i inventory site.yml
```

Ansible will configure the managed server according to the values defined in `defaults/main.yml`.

---

# Verification

## Verify Installed Software

Check Git:

```bash
ansible managed -i inventory -m shell -a "git --version"
```

Check curl:

```bash
ansible managed -i inventory -m shell -a "curl --version | head -1"
```

---

## Verify Groups

Check the `devops` group:

```bash
ansible managed -i inventory -m shell -a "getent group devops"
```

Check the `developers` group:

```bash
ansible managed -i inventory -m shell -a "getent group developers"
```

---

## Verify Users

Check `devops1`:

```bash
ansible managed -i inventory -m shell -a "id devops1"
```

Check `devops2`:

```bash
ansible managed -i inventory -m shell -a "id devops2"
```

Check `developer1`:

```bash
ansible managed -i inventory -m shell -a "id developer1"
```

---

## Verify Directory Structure

```bash
ansible managed -i inventory -m shell -a "ls -ld /opt/devops /opt/projects /opt/scripts /opt/logs"
```

This verifies that the required directories exist and allows their ownership and permissions to be checked.

---

## Verify Git Repository

Check that the repository exists:

```bash
ansible managed -i inventory -m shell -a "ls -ld /opt/projects/spring3hibernate"
```

Check repository status:

```bash
ansible managed -i inventory -m shell -a "cd /opt/projects/spring3hibernate && git status"
```

The command should show that the directory is a Git repository and display its current branch and status.

---

# Role Workflow

The role follows a simple separation between configuration and implementation:

```text
defaults/main.yml
        |
        | Defines WHAT should be managed
        v
tasks/
        |
        | Defines HOW it should be managed
        v
Managed Linux VM
```

For example:

```text
system_packages
        |
        v
packages.yml
        |
        v
Install packages
```

The same approach is used for users, directories, and Git repositories.

# Conclusion

The `system_manager` role provides a reusable way to manage the resources required by the assignment.

The role covers:

```text
Software management
User and group management
Git repository management
Directory structure management
```

Configuration can be changed in `defaults/main.yml` without changing the underlying task logic.
