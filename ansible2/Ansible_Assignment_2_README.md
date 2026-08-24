<img width="1217" height="564" alt="image" src="https://github.com/user-attachments/assets/72c63963-88cd-49d2-83e9-0ce4664fd52c" /># Ansible Assignment 2

## Objective

The objective of this assignment is to configure multiple worker nodes using **Ansible ad-hoc commands only**.

The assignment includes:

- Installing Nginx on more than two servers.
- Limiting Nginx log storage to approximately 1 GB using log rotation.
- Creating websites for each team member.
- Displaying each website for 2 hours before switching to the next website.
- Installing Apache.
- Configuring Nginx as a reverse proxy for Apache.
- Executing Ansible operations **one worker at a time**.

For testing the website rotation, the schedule can temporarily be changed from **2 hours to 2 minutes**.

---

#  Inventory

The worker nodes are defined in the Ansible inventory:

```ini
[workers]
worker1 ansible_host=65.2.69.67
worker2 ansible_host=13.232.43.62
worker3 ansible_host=43.204.234.167

[workers:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/garvit/ansible/ansible.pem


```

Check connectivity:

```bash
ansible workers -i inventory -m ping
```
<img width="597" height="252" alt="image" src="https://github.com/user-attachments/assets/ce8b45a0-3de1-472c-a1a2-cc89b4285405" />
<img width="600" height="589" alt="image" src="https://github.com/user-attachments/assets/b5f40ef7-4e57-48a8-86e5-f110aae2a331" />


---
# Sequential Execution

The assignment requires the worker nodes to be updated **one by one**.

For this, all commands use:

```bash
--forks 1
```

Example:

```bash
ansible workers -i inventory -m ping --forks 1
```


---

#  Install Nginx

Install Nginx on all workers:

```bash
ansible workers -i inventory -m apt -a "name=nginx state=present update_cache=yes" --become --forks 1
```
<img width="1190" height="611" alt="image" src="https://github.com/user-attachments/assets/37724ead-fef8-4ec5-8705-d613733e8fc3" />

Start and enable Nginx:

```bash
ansible workers -i inventory -m service -a "name=nginx state=started enabled=yes" --become --forks 1
```
<img width="1190" height="611" alt="image" src="https://github.com/user-attachments/assets/3d129f4f-4e19-44f3-b754-0ff4e8dbabc0" />

Verify:

```bash
ansible workers -i inventory -m command -a "systemctl is-active nginx" --become --forks 1
```
<img width="1193" height="319" alt="image" src="https://github.com/user-attachments/assets/f96d7ef8-9ea1-4c2a-95d6-3dba7e06aa00" />

---

# Configure Nginx Log Rotation

Nginx logs are stored in:

```text
/var/log/nginx/
```

Install logrotate:

```bash
ansible workers -i inventory -m apt -a "name=logrotate state=present update_cache=yes" --become --forks 1
```
<img width="1191" height="643" alt="image" src="https://github.com/user-attachments/assets/d70a9de8-4b83-451e-9315-ead4dab5a10d" />

Create the Nginx logrotate configuration:

```bash
ansible workers -i inventory -m copy -a 'dest=/etc/logrotate.d/nginx content="/var/log/nginx/*.log {
    size 100M
    rotate 9
    compress
    delaycompress
}
" mode=0644' --become --forks 1
```
<img width="1184" height="635" alt="image" src="https://github.com/user-attachments/assets/68c5d9b0-30ee-4cb0-9a18-8e236697ceb3" />

### Configuration explanation

```text
size 100M
```

Rotates a log when it reaches approximately 100 MB.

```text
rotate 9
```

Keeps 9 rotated copies.

```text
compress
```

Compresses older logs.

```text
delaycompress
```

Delays compression of the most recently rotated log.

Check the current log usage:

```bash
ansible workers -i inventory -m command -a "du -sh /var/log/nginx" --become --forks 1
```
<img width="1193" height="320" alt="image" src="https://github.com/user-attachments/assets/875ecef6-ad2e-48cf-9592-a08eccbb82c5" />

The total should remain well below the assignment's 1 GB requirement under normal operation.

---

#  Create Team Member Websites

For this implementation, the team members are:

```text
Garvit
Anya
Sahil
Ankur
Kuldeep
Arhan
```

Create directories:

```bash
ansible workers -i inventory -m file -a "path=/var/www/garvit state=directory owner=www-data group=www-data mode=0755" --become --forks 1
```
<img width="1193" height="320" alt="image" src="https://github.com/user-attachments/assets/49687824-176e-4e46-bbb6-4ba7878dcebc" />

The same command will be used to create directories of every team member
```

Create garvit website:

```bash
ansible workers -i inventory -m copy -a 'dest=/var/www/garvit/index.html content="<html><body><h1>garvit Website</h1><p>This is garvit website.</p></body></html>" owner=www-data group=www-data mode=0644' --become --forks 1
```
websites can be created for every user by replacing appropriate details.
---

# Create the Current Website

A symbolic link is used so that the web server can always serve one fixed location:

```text
/var/www/current
```

Initially point it to garvit:

```bash
ansible workers -i inventory -m file -a "src=/var/www/garvit dest=/var/www/current state=link force=yes" --become --forks 1
```

Check the current website:

```bash
ansible workers -i inventory -m command -a "readlink -f /var/www/current" --become --forks 1
```
<img width="1193" height="320" alt="image" src="https://github.com/user-attachments/assets/ec763054-d591-4acb-a9ff-1c014da89624" />

Expected:

```text
/var/www/garvit
```

The symlink allows the active website to be changed without modifying the web-server configuration.

---

#  Configure Nginx

Create the team Nginx configuration:

```bash
ansible workers -i inventory -m copy -a 'dest=/etc/nginx/sites-available/team.conf content="server {
    listen 80;
    server_name NCR.opstree.com;

    root /var/www/current;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
" mode=0644' --become --forks 1
```
<img width="1218" height="663" alt="image" src="https://github.com/user-attachments/assets/da2082b4-f315-4b8f-b89e-02f4000917bb" />

Enable it:

```bash
ansible workers -i inventory -m file -a "src=/etc/nginx/sites-available/team.conf dest=/etc/nginx/sites-enabled/team.conf state=link" --become --forks 1
```
<img width="1218" height="663" alt="image" src="https://github.com/user-attachments/assets/e7276754-fcec-4002-9ff1-6bc32e746df3" />

Remove the default Nginx configuration:

```bash
ansible workers -i inventory -m file -a "path=/etc/nginx/sites-enabled/default state=absent" --become --forks 1
```
<img width="1217" height="683" alt="image" src="https://github.com/user-attachments/assets/2e5ec0f7-5c45-4129-b67e-afdac8ba5c3e" />

Test Nginx:

```bash
ansible workers -i inventory -m command -a "nginx -t" --become --forks 1
```
<img width="1210" height="382" alt="image" src="https://github.com/user-attachments/assets/53b6e6b5-670f-42dc-baed-5525c85ed7bc" />

Reload:

```bash
ansible workers -i inventory -m service -a "name=nginx state=reloaded" --become --forks 1
```
<img width="1215" height="700" alt="image" src="https://github.com/user-attachments/assets/1becb6f0-c159-4742-9083-39b2443605c5" />

Test:

```bash
curl -H "Host: NCR.opstree.com" http://<WORKER_PUBLIC_IP>
```
<img width="1222" height="70" alt="image" src="https://github.com/user-attachments/assets/0c20edd7-7f02-4ebe-bb62-92d3de255017" />

The response should contain the currently active team member's website.

---

# Website Rotation Script

Create the rotation script:

```bash
ansible workers -i inventory -m copy -a 'dest=/usr/local/bin/rotate-websites.sh mode=0755 content="#!/bin/bash

CURRENT=$(readlink -f /var/www/current)

if [ \"$CURRENT\" = \"/var/www/garvit\" ]; then
    ln -sfn /var/www/tanys /var/www/current
elif [ \"$CURRENT\" = \"/var/www/anya\" ]; then
    ln -sfn /var/www/sahil /var/www/current
elif [ \"$CURRENT\" = \"/var/www/sahil\" ]; then
    ln -sfn /var/www/ankur /var/www/current
elif [ \"$CURRENT\" = \"/var/www/ankur\" ]; then
    ln -sfn /var/www/kuldeep /var/www/current
elif [ \"$CURRENT\" = \"/var/www/kuldeep\" ]; then
    ln -sfn /var/www/arhan /var/www/current
else
    ln -sfn /var/www/garvit /var/www/current
fi
"' --become --forks 1
```
<img width="1213" height="612" alt="image" src="https://github.com/user-attachments/assets/99b599f4-7515-4d84-b36d-7034b433bbad" />

```

Test it manually:

```bash
ansible workers -i inventory -m command -a "/usr/local/bin/rotate-websites.sh" --become --forks 1
```

Check:

```bash
ansible workers -i inventory -m command -a "readlink -f /var/www/current" --become --forks 1
```
<img width="1217" height="470" alt="image" src="https://github.com/user-attachments/assets/66c6e98e-b8eb-470d-83d9-2c6162613dad" />

Running the script repeatedly should cycle through the websites.

---

#  Configure Cron

## For Assignment

The actual assignment requires each website to be displayed for **2 hours**.

Therefore the production cron schedule is:

```bash
ansible workers -i inventory -m cron -a "name='Rotate team website' minute=0 hour='*/2' job='/usr/local/bin/rotate-websites.sh'" --become --forks 1
```
<img width="1207" height="613" alt="image" src="https://github.com/user-attachments/assets/3e04e0af-f8b9-4ebd-a72a-74e0e8da6c92" />

Verify:

```bash
ansible workers -i inventory -m command -a "crontab -l" --become --forks 1
```
<img width="1213" height="468" alt="image" src="https://github.com/user-attachments/assets/6d0a24d4-ee49-410e-b0a7-f6e982145206" />

Expected:

```text
#Ansible: Rotate team website
0 */2 * * * /usr/local/bin/rotate-websites.sh
```


## For Testing

Waiting 2 hours is inconvenient during testing, so the schedule can temporarily be changed to every 2 minutes:

```bash
ansible workers -i inventory -m cron -a "name='Rotate team website' minute='*/2' job='/usr/local/bin/rotate-websites.sh'" --become --forks 1
```
<img width="1216" height="705" alt="image" src="https://github.com/user-attachments/assets/b8eaee0b-b951-4d52-961d-6493d6bd563a" />


After testing, change it back to the required 2-hour schedule.

---

#  Install Apache

Install Apache:

```bash
ansible workers -i inventory -m apt -a "name=apache2 state=present update_cache=yes" --become --forks 1
```
<img width="1216" height="533" alt="image" src="https://github.com/user-attachments/assets/fd41efc2-70c1-4c7d-8300-9048f20ec4c8" />

Apache initially tries to use port 80, but Nginx is already using port 80.

Therefore Apache is moved to port 8080.

---

#  Configure Apache on Port 8080

Change Apache's listening port:

```bash
ansible workers -i inventory -m lineinfile -a "path=/etc/apache2/ports.conf regexp='^Listen ' line='Listen 8080'" --become --forks 1
```
<img width="1216" height="649" alt="image" src="https://github.com/user-attachments/assets/e417b05c-edca-44b4-9134-ede86109e9c5" />

Change the Apache virtual host:

```bash
ansible workers -i inventory -m replace -a "path=/etc/apache2/sites-available/000-default.conf regexp='<VirtualHost \*:80>' replace='<VirtualHost *:8080>'" --become --forks 1
```
<img width="1216" height="649" alt="image" src="https://github.com/user-attachments/assets/1a0db3c5-87af-4db3-94b8-673155d91396" />

Test Apache:

```bash
ansible workers -i inventory -m command -a "apache2ctl configtest" --become --forks 1
```

<img width="1212" height="293" alt="image" src="https://github.com/user-attachments/assets/285de552-f69f-4e66-a360-1826a47672fd" />


Start and enable Apache:

```bash
ansible workers -i inventory -m service -a "name=apache2 state=started enabled=yes" --become --forks 1
```
<img width="1212" height="445" alt="image" src="https://github.com/user-attachments/assets/a69c99fb-df4a-48f6-afa0-3bd67a2aace2" />

Verify:

```bash
ansible workers -i inventory -m command -a "ss -lntp" --become --forks 1
```
<img width="1217" height="564" alt="image" src="https://github.com/user-attachments/assets/9e643b8e-3f1e-4e2b-8209-c5ae7acfb6af" />

---

# Configure Apache to Serve the Rotating Website

Apache must serve:

```text
/var/www/current
```

instead of its default:

```text
/var/www/html
```

Change the DocumentRoot:

```bash
ansible workers -i inventory -m replace -a "path=/etc/apache2/sites-available/000-default.conf regexp='DocumentRoot /var/www/html' replace='DocumentRoot /var/www/current'" --become --forks 1
```
<img width="1217" height="564" alt="image" src="https://github.com/user-attachments/assets/4dcc2015-361a-4700-95bd-b48d32035791" />

Restart Apache:

```bash
ansible workers -i inventory -m service -a "name=apache2 state=restarted" --become --forks 1
```
<img width="1215" height="604" alt="image" src="https://github.com/user-attachments/assets/b24f2766-4901-416f-aa20-e0e1fe2fc36c" />

Test Apache:

```bash
ansible workers -i inventory -m command -a "curl -s http://127.0.0.1:8080" --become --forks 1
```
<img width="1210" height="399" alt="image" src="https://github.com/user-attachments/assets/5f0e4173-7954-476e-9be4-51c15af5e064" />

Apache should now return the currently active team member's website.

---

# Configure Nginx as Reverse Proxy

Replace the Nginx configuration:

```bash
ansible workers -i inventory -m copy -a 'dest=/etc/nginx/sites-available/team.conf content="server {
    listen 80;
    server_name NCR.opstree.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
" mode=0644' --become --forks 1
```
<img width="1210" height="674" alt="image" src="https://github.com/user-attachments/assets/660c790c-3b4a-441d-8de6-3f07e99455ce" />

Test Nginx:

```bash
ansible workers -i inventory -m command -a "nginx -t" --become --forks 1
```
<img width="1212" height="446" alt="image" src="https://github.com/user-attachments/assets/ca0de65d-0eb5-4f17-b59c-31dd733c5a56" />

Reload Nginx:

```bash
ansible workers -i inventory -m service -a "name=nginx state=reloaded" --become --forks 1
```

---

# Final Request Flow

The final architecture is:

```text
                         Client
                           |
                           | HTTP :80
                           v
                    +--------------+
                    |    Nginx     |
                    |     :80      |
                    +--------------+
                           |
                           | proxy_pass
                           v
                    +--------------+
                    |    Apache    |
                    |    :8080     |
                    +--------------+
                           |
                           v
                    /var/www/current
```

Configure /etc/hosts

Because Nginx uses:

server_name NCR.opstree.com;

add the hostname to the /etc/hosts file on the computer where the browser is running:

sudo nano /etc/hosts

Add:

65.2.69.67 NCR.opstree.com

Save the file.

Then open the browser and visit:

http://NCR.opstree.com

<img width="1205" height="701" alt="image" src="https://github.com/user-attachments/assets/eb324def-e569-4d18-a9f9-a17641f39b5e" />
<img width="1217" height="798" alt="image" src="https://github.com/user-attachments/assets/9ffec729-3780-4e85-951e-50e269927d33" />
<img width="1217" height="798" alt="image" src="https://github.com/user-attachments/assets/5f7e24cf-d45e-4424-87e5-3a70bf04e47e" />

