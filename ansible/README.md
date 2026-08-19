# Ansible Assignment 2

##Inventory File
<img width="578" height="271" alt="image" src="https://github.com/user-attachments/assets/0524b538-2979-43c2-ad35-a7fcd98cf7df" />

### Verify connectivity

```bash
ansible workers -i inventory -m ping
```
<img width="1220" height="578" alt="image" src="https://github.com/user-attachments/assets/81f40e53-654b-4dec-95d3-c1b2f4a09875" />


## Part 1: Install Nginx

The first step was to install Nginx on all worker nodes u
```bash
ansible worker -i inventory -m apt -a "name=nginx state=present update_cache=yes" --become --forks 1
```
<img width="1220" height="627" alt="image" src="https://github.com/user-attachments/assets/fb2b4980-7d5b-4055-a444-fda5a0f8a597" />


### Start and enable Nginx

```bash
ansible worker -i inventory -m service -a "name=nginx state=started enabled=yes" --become --forks 1
```
<img width="1220" height="578" alt="image" src="https://github.com/user-attachments/assets/12984a7c-f889-43a4-8e3c-c134da6bf1ae" />

### Verification

```bash
ansible workers -i inventory -m command -a "systemctl is-active nginx" --become --forks 1
```
<img width="1220" height="298" alt="image" src="https://github.com/user-attachments/assets/5451647b-0c0c-47fd-bb8d-7ce86f891b36" />

### Why `--forks 1`?

`--forks 1` ensures that Ansible processes one worker node at a time instead of executing the command on multiple worker nodes simultaneously.

The execution order is:

```text
server1 → server2 → server3
```

This satisfies the requirement that worker nodes should be updated one by one.


