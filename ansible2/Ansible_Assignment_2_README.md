# Ansible Assignment 2

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

# 1. Inventory

The worker nodes are defined in the Ansible inventory:

```ini
[workers]
worker1
worker2
worker3
```

Check connectivity:

```bash
ansible workers -i inventory -m ping
```

All workers should return:

```text
SUCCESS
"ping": "pong"
```

---

# 2. Sequential Execution

The assignment requires the worker nodes to be updated **one by one**.

For this, all commands use:

```bash
--forks 1
```

Example:

```bash
ansible workers -i inventory -m ping --forks 1
```

This makes Ansible process:

```text
worker1
   ↓
worker2
   ↓
worker3
```

instead of executing against all workers concurrently.

---

# 3. Install Nginx

Install Nginx on all workers:

```bash
ansible workers -i inventory -m apt -a "name=nginx state=present update_cache=yes" --become --forks 1
```

Start and enable Nginx:

```bash
ansible workers -i inventory -m service -a "name=nginx state=started enabled=yes" --become --forks 1
```

Verify:

```bash
ansible workers -i inventory -m command -a "systemctl is-active nginx" --become --forks 1
```

Expected:

```text
active
```

---

# 4. Configure Nginx Log Rotation

Nginx logs are stored in:

```text
/var/log/nginx/
```

Install logrotate:

```bash
ansible workers -i inventory -m apt -a "name=logrotate state=present update_cache=yes" --become --forks 1
```

Create the Nginx logrotate configuration:

```bash
ansible workers -i inventory -m copy -a 'dest=/etc/logrotate.d/nginx content="/var/log/nginx/*.log {
    size 100M
    rotate 9
    compress
    delaycompress
    missingok
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
    endscript
}
" mode=0644' --become --forks 1
```

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

The total should remain well below the assignment's 1 GB requirement under normal operation.

Test logrotate:

```bash
ansible workers -i inventory -m command -a "logrotate -d /etc/logrotate.d/nginx" --become --forks 1
```

---

# 5. Create Team Member Websites

For this implementation, the team members are:

```text
Tanya
Heena
Ankur
```

Create directories:

```bash
ansible workers -i inventory -m file -a "path=/var/www/tanya state=directory owner=www-data group=www-data mode=0755" --become --forks 1
```

```bash
ansible workers -i inventory -m file -a "path=/var/www/heena state=directory owner=www-data group=www-data mode=0755" --become --forks 1
```

```bash
ansible workers -i inventory -m file -a "path=/var/www/ankur state=directory owner=www-data group=www-data mode=0755" --become --forks 1
```

Create Tanya's website:

```bash
ansible workers -i inventory -m copy -a 'dest=/var/www/tanya/index.html content="<html><body><h1>Tanya Website</h1><p>This is Tanya website.</p></body></html>" owner=www-data group=www-data mode=0644' --become --forks 1
```

Create Heena's website:

```bash
ansible workers -i inventory -m copy -a 'dest=/var/www/heena/index.html content="<html><body><h1>Heena Website</h1><p>This is Heena website.</p></body></html>" owner=www-data group=www-data mode=0644' --become --forks 1
```

Create Ankur's website:

```bash
ansible workers -i inventory -m copy -a 'dest=/var/www/ankur/index.html content="<html><body><h1>Ankur Website</h1><p>This is Ankur website.</p></body></html>" owner=www-data group=www-data mode=0644' --become --forks 1
```

---

# 6. Create the Current Website

A symbolic link is used so that the web server can always serve one fixed location:

```text
/var/www/current
```

Initially point it to Tanya:

```bash
ansible workers -i inventory -m file -a "src=/var/www/tanya dest=/var/www/current state=link force=yes" --become --forks 1
```

Check the current website:

```bash
ansible workers -i inventory -m command -a "readlink -f /var/www/current" --become --forks 1
```

Expected:

```text
/var/www/tanya
```

The symlink allows the active website to be changed without modifying the web-server configuration.

---

# 7. Configure Nginx

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

Enable it:

```bash
ansible workers -i inventory -m file -a "src=/etc/nginx/sites-available/team.conf dest=/etc/nginx/sites-enabled/team.conf state=link" --become --forks 1
```

Remove the default Nginx configuration:

```bash
ansible workers -i inventory -m file -a "path=/etc/nginx/sites-enabled/default state=absent" --become --forks 1
```

Test Nginx:

```bash
ansible workers -i inventory -m command -a "nginx -t" --become --forks 1
```

Reload:

```bash
ansible workers -i inventory -m service -a "name=nginx state=reloaded" --become --forks 1
```

Test:

```bash
curl -H "Host: NCR.opstree.com" http://<WORKER_PUBLIC_IP>
```

The response should contain the currently active team member's website.

---

# 8. Website Rotation Script

Create the rotation script:

```bash
ansible workers -i inventory -m copy -a 'dest=/usr/local/bin/rotate-websites.sh mode=0755 content="#!/bin/bash

CURRENT=$(readlink -f /var/www/current)

if [ \"$CURRENT\" = \"/var/www/tanya\" ]; then
    ln -sfn /var/www/heena /var/www/current
elif [ \"$CURRENT\" = \"/var/www/heena\" ]; then
    ln -sfn /var/www/ankur /var/www/current
else
    ln -sfn /var/www/tanya /var/www/current
fi
"' --become --forks 1
```

The script performs:

```text
Tanya
  ↓
Heena
  ↓
Ankur
  ↓
Tanya
  ↓
...
```

Test it manually:

```bash
ansible workers -i inventory -m command -a "/usr/local/bin/rotate-websites.sh" --become --forks 1
```

Check:

```bash
ansible workers -i inventory -m command -a "readlink -f /var/www/current" --become --forks 1
```

Running the script repeatedly should cycle through the websites.

---

# 9. Configure Cron

## For Assignment

The actual assignment requires each website to be displayed for **2 hours**.

Therefore the production cron schedule is:

```bash
ansible workers -i inventory -m cron -a "name='Rotate team website' minute=0 hour='*/2' job='/usr/local/bin/rotate-websites.sh'" --become --forks 1
```

Verify:

```bash
ansible workers -i inventory -m command -a "crontab -l" --become --forks 1
```

Expected:

```text
#Ansible: Rotate team website
0 */2 * * * /usr/local/bin/rotate-websites.sh
```

This runs at:

```text
00:00
02:00
04:00
06:00
08:00
...
```

Therefore:

```text
0–2 hours    → Tanya
2–4 hours    → Heena
4–6 hours    → Ankur
6–8 hours    → Tanya
...
```

## For Testing

Waiting 2 hours is inconvenient during testing, so the schedule can temporarily be changed to every 2 minutes:

```bash
ansible workers -i inventory -m cron -a "name='Rotate team website' minute='*/2' job='/usr/local/bin/rotate-websites.sh'" --become --forks 1
```

Verify:

```bash
ansible workers -i inventory -m command -a "crontab -l" --become --forks 1
```

Expected:

```text
#Ansible: Rotate team website
*/2 * * * * /usr/local/bin/rotate-websites.sh
```

After testing, change it back to the required 2-hour schedule.

---

# 10. Install Apache

Install Apache:

```bash
ansible workers -i inventory -m apt -a "name=apache2 state=present update_cache=yes" --become --forks 1
```

Apache initially tries to use port 80, but Nginx is already using port 80.

Therefore Apache is moved to port 8080.

---

# 11. Configure Apache on Port 8080

Change Apache's listening port:

```bash
ansible workers -i inventory -m lineinfile -a "path=/etc/apache2/ports.conf regexp='^Listen ' line='Listen 8080'" --become --forks 1
```

Change the Apache virtual host:

```bash
ansible workers -i inventory -m replace -a "path=/etc/apache2/sites-available/000-default.conf regexp='<VirtualHost \*:80>' replace='<VirtualHost *:8080>'" --become --forks 1
```

Test Apache:

```bash
ansible workers -i inventory -m command -a "apache2ctl configtest" --become --forks 1
```

Expected:

```text
Syntax OK
```

Start and enable Apache:

```bash
ansible workers -i inventory -m service -a "name=apache2 state=started enabled=yes" --become --forks 1
```

Verify:

```bash
ansible workers -i inventory -m command -a "ss -lntp" --become --forks 1
```

The expected architecture is:

```text
Nginx   → :80
Apache  → :8080
```

---

# 12. Configure Apache to Serve the Rotating Website

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

Restart Apache:

```bash
ansible workers -i inventory -m service -a "name=apache2 state=restarted" --become --forks 1
```

Test Apache:

```bash
ansible workers -i inventory -m command -a "curl -s http://127.0.0.1:8080" --become --forks 1
```

Apache should now return the currently active team member's website.

---

# 13. Configure Nginx as Reverse Proxy

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

Test Nginx:

```bash
ansible workers -i inventory -m command -a "nginx -t" --become --forks 1
```

Reload Nginx:

```bash
ansible workers -i inventory -m service -a "name=nginx state=reloaded" --become --forks 1
```

---

# 14. Final Request Flow

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
                           |
              +------------+------------+
              |            |            |
              v            v            v
            Tanya        Heena        Ankur
```

The cron job changes:

```text
/var/www/current
```

every 2 hours.

Therefore the complete request path is:

```text
Browser
   ↓
Nginx :80
   ↓
Apache :8080
   ↓
/var/www/current
   ↓
Current team member website
```

---

# 15. Verification

## Check Nginx

```bash
ansible workers -i inventory -m command -a "systemctl is-active nginx" --become --forks 1
```

Expected:

```text
active
```

## Check Apache

```bash
ansible workers -i inventory -m command -a "systemctl is-active apache2" --become --forks 1
```

Expected:

```text
active
```

## Check Ports

```bash
ansible workers -i inventory -m command -a "ss -lntp" --become --forks 1
```

Expected:

```text
Nginx   → 0.0.0.0:80
Apache  → *:8080
```

## Check Current Website

```bash
ansible workers -i inventory -m command -a "readlink -f /var/www/current" --become --forks 1
```

Example:

```text
/var/www/ankur
```

## Check Log Usage

```bash
ansible workers -i inventory -m command -a "du -sh /var/log/nginx" --become --forks 1
```

The current usage should remain well below 1 GB.

## Check Cron

```bash
ansible workers -i inventory -m command -a "crontab -l" --become --forks 1
```

For the actual assignment:

```text
0 */2 * * * /usr/local/bin/rotate-websites.sh
```

## Test Through Nginx

```bash
curl -H "Host: NCR.opstree.com" http://<WORKER_PUBLIC_IP>
```

The response should be the currently active team member's webpage.

---

# 16. AWS Security Group

For external HTTP access, the worker Security Group must allow:

```text
TCP 80
```

SSH requires:

```text
TCP 22
```

Apache's port `8080` does **not** need to be publicly exposed because Nginx communicates with Apache locally:

```text
Nginx → 127.0.0.1:8080 → Apache
```

---

# 17. Ansible Ad-Hoc Modules Used

This assignment was implemented using ad-hoc commands only.

### `apt`

Used to install:

```text
nginx
apache2
logrotate
```

### `service`

Used to:

```text
start
restart
reload
enable
```

Nginx and Apache.

### `file`

Used to create:

- Website directories
- Symbolic links
- Nginx configuration links

### `copy`

Used to create:

- HTML files
- Nginx configuration
- Website rotation script
- Logrotate configuration

### `command`

Used for:

- Verification
- `nginx -t`
- `systemctl`
- `curl`
- `ss`
- `readlink`
- `du`

### `lineinfile`

Used to change Apache's listening port.

### `replace`

Used to modify Apache's virtual host and DocumentRoot.

### `cron`

Used to schedule automatic website rotation.

---

# 18. Why `--forks 1` Was Used

By default, Ansible can execute operations on multiple hosts concurrently.

For this assignment, the requirement was to update worker nodes one by one.

Therefore commands were executed with:

```bash
--forks 1
```

This ensures sequential execution:

```text
Controller
    |
    +----> Worker 1
    |
    +----> Worker 2
    |
    +----> Worker 3
```

No playbook was used; the entire configuration was performed through **Ansible ad-hoc commands**.

---

# Final Result

The completed environment provides:

```text
3 Worker Nodes
      |
      +---- Nginx :80
      |         |
      |         +---- Reverse Proxy
      |                    |
      |                    v
      |               Apache :8080
      |                    |
      |                    v
      |              /var/www/current
      |                    |
      |          +---------+---------+
      |          |         |         |
      |        Tanya     Heena     Ankur
      |
      +---- Nginx Log Rotation
      |         |
      |         +---- Logs rotated/compressed
      |              to control disk usage
      |
      +---- Cron
                |
                +---- Website changes every 2 hours
```

The assignment is implemented using **Ansible ad-hoc commands**, with workers processed sequentially using `--forks 1`.
