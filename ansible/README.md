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


## Part 2: Nginx Log Management

Nginx stores its logs in `/var/log/nginx`. To prevent old logs from continuously consuming disk space, `logrotate` was configured on all worker nodes.

### Check current log usage

```bash
ansible workers -i inventory -m command -a "du -sh /var/log/nginx" --become --forks 1
```
<img width="1212" height="330" alt="image" src="https://github.com/user-attachments/assets/4434cd64-0536-4143-b22c-de86a115127f" />

### If not installed Install logrotate

```bash
ansible workers -i inventory -m apt -a "name=logrotate state=present update_cache=yes" --become --forks 1
```
<img width="1212" height="637" alt="image" src="https://github.com/user-attachments/assets/43619a62-68a9-4005-a464-e4456aa32e2d" />

### Configure log rotation

```bash
ansible workers -i inventory -m copy -a 'dest=/etc/logrotate.d/nginx content="/var/log/nginx/*.log {
    size 100M
    rotate 9
    compress
    copytruncate
}
"' --become --forks 1
```
<img width="1195" height="465" alt="image" src="https://github.com/user-attachments/assets/6c50ce07-f38e-49a9-a526-e3f22922d583" />


The configuration rotates Nginx logs when they reach approximately 100 MB and retains 9 rotated copies in addition to the current log. Old logs are compressed.

This keeps the retained Nginx logs around the 1 GB range while preventing unlimited log growth.


### Test log rotation

```bash
ansible workers -i inventory -m command -a "logrotate -f /etc/logrotate.d/nginx" --become --forks 1
```
<img width="1193" height="319" alt="image" src="https://github.com/user-attachments/assets/d396b5fd-af71-4878-bf26-ad13a442bc47" />


### Verify rotated logs

```bash
ansible workers -i inventory -m command -a "ls -lh /var/log/nginx" --become --forks 1
```

<img width="1191" height="657" alt="image" src="https://github.com/user-attachments/assets/f03a5530-601f-450d-8283-b755adde2e4b" />



