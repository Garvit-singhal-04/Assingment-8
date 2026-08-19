# OTSSH – SSH Connection Manager

## 1. Introduction

**OTSSH** is a Bash-based SSH connection management utility.

It provides a simple way to store and manage SSH connection details and connect to remote servers using a short connection name instead of repeatedly entering the complete SSH command.

---

## 2. Features

| Operation               | Command           |
| ----------------------- | ----------------- |
| Add connection          | `otssh -a`        |
| List connections        | `otssh ls`        |
| List connection details | `otssh ls -d`     |
| Update connection       | `otssh -u`        |
| Delete connection       | `otssh rm <name>` |
| Connect to server       | `otssh <name>`    |
| Show usage              | `otssh`           |

---

## 3. Requirements

The utility requires:

* Linux/Unix-based operating system
* Bash shell
* OpenSSH client
* Standard Linux utilities such as:

  * `grep`
  * `cut`
  * `cat`
  * `mv`
  * `rm`

Verify Bash and SSH are available:

```bash
bash --version
ssh -V
```

---

## 4. Installation

Save the script as:

```text
otssh
```

Give it execute permission:

```bash
chmod +x otssh
```

To make the command available globally, copy it to a directory in your `PATH`.

For example:

```bash
sudo cp otssh /usr/local/bin/otssh
```

Verify the installation:

```bash
otssh
```

The utility should display its usage information.

---

## 5. Data Storage

OTSSH stores connection information in a local database file:

```text
$HOME/otssh_db
```

The database is automatically created when the utility is executed if it does not already exist.

Each connection is stored as a pipe-separated record:

```text
name|host|user|port|key
```

For example:

```text
server1|192.168.21.30|kirti|22|
server2|192.168.42.34|kirti|2022|
server3|192.168.46.34|ubuntu|2022|~/.ssh/server3.pem
```

The fields represent:

| Field  | Description                          |
| ------ | ------------------------------------ |
| `name` | Name used to identify the connection |
| `host` | Remote server IP address or hostname |
| `user` | SSH username                         |
| `port` | SSH port                             |
| `key`  | SSH private key path                 |

Port `22` is used as the default SSH port.

---

# 6. Add SSH Connection

The `-a` option adds a new SSH connection.

## 6.1 Add a connection with default SSH settings

```bash
otssh -a -n server1 -h 192.168.21.30 -u kirti
```

This creates a connection named `server1` with:

* Host: `192.168.21.30`
* User: `kirti`
* Port: `22`
* SSH key: Not specified

The equivalent SSH command is:

```bash
ssh kirti@192.168.21.30
```

### Screenshot

<img width="1093" height="489" alt="image" src="https://github.com/user-attachments/assets/87e333b9-e063-4b2a-8cf1-052d1941279b" />


---

## 6.2 Add a connection with a custom port

```bash
otssh -a -n server2 -h 192.168.42.34 -u kirti -p 2022
```

The connection uses port `2022`.

Equivalent SSH command:

```bash
ssh -p 2022 kirti@192.168.42.34
```

### Screenshot

<img width="746" height="162" alt="image" src="https://github.com/user-attachments/assets/bbe4eb31-d23c-4d8b-ad17-4682133e5700" />


---

## 6.3 Add a connection with an SSH key

```bash
otssh -a -n server3 -h 192.168.46.34 -u ubuntu -p 2022 -i ~/.ssh/server3.pem
```

This stores:

* Name: `server3`
* Host: `192.168.46.34`
* User: `ubuntu`
* Port: `2022`
* Key: `~/.ssh/server3.pem`

Equivalent SSH command:

```bash
ssh -i ~/.ssh/server3.pem -p 2022 ubuntu@192.168.46.34
```

### Screenshot

<img width="912" height="178" alt="image" src="https://github.com/user-attachments/assets/4cc580cc-6bd1-4c3b-be31-667d440eb8df" />


---

# 7. List SSH Connections

## 7.1 List connection names

The following command displays the names of all saved connections:

```bash
otssh ls
```

Example output:

```text
server1
server2
server3
```

The `ls` operation uses the first field of each database record to display the connection name.

### Screenshot
<img width="912" height="178" alt="image" src="https://github.com/user-attachments/assets/309aa26c-bdf5-4ceb-9e94-62692af8e3b3" />

---

## 7.2 List connections with details

Use the `-d` option:

```bash
otssh ls -d
```

Example output:

```text
server1: ssh kirti@192.168.21.30
server2: ssh -p 2022 kirti@192.168.42.34
server3: ssh -i ~/.ssh/server3.pem -p 2022 ubuntu@192.168.46.34
```

The command is constructed dynamically based on the stored connection settings.

For example:

* If the port is `22`, `-p` is omitted.
* If no SSH key is configured, `-i` is omitted.
* If a custom port and key are configured, both options are included.

### Screenshot

<img width="912" height="178" alt="image" src="https://github.com/user-attachments/assets/7998b4c7-a48c-4a82-aa50-cd943d5a1db6" />


---

# 8. Update SSH Connection

The `-u` option updates an existing connection.

The connection name identifies which existing record should be modified.

## 8.1 Update a connection

```bash
otssh -u -n server1 -h server1 -u user1
```

This changes the connection to:

```text
server1|server1|user1|22|
```

The resulting SSH command is:

```bash
ssh user1@server1
```

### Screenshot
<img width="912" height="178" alt="image" src="https://github.com/user-attachments/assets/3c02c6b9-46b6-49cc-82cc-4bbed8a4efb3" />


---

## 8.2 Update a connection with a custom port

```bash
otssh -u -n server2 -h server2 -u user2 -p 2022
```

The resulting SSH command becomes:

```bash
ssh -p 2022 user2@server2
```

Verify the changes:

```bash
otssh ls -d
```

Example:

```text
server1: ssh user1@server1
server2: ssh -p 2022 user2@server2
server3: ssh -i ~/.ssh/server3.pem -p 2022 ubuntu@192.168.46.34
```

### Screenshot

<img width="912" height="178" alt="image" src="https://github.com/user-attachments/assets/df788861-32bf-41a9-a5e7-3e10cd0c8e7f" />


---

# 9. Delete SSH Connection

The `rm` command removes a saved connection.

For example:

```bash
otssh rm server1
```

The connection named `server1` is removed from the database.

Another example:

```bash
otssh rm server2
```

After deleting the connections, verify the database:

```bash
otssh ls -d
```

### Screenshot

<img width="912" height="178" alt="image" src="https://github.com/user-attachments/assets/9285cd84-d48c-45f9-8699-65194afedced" />


---

# 10. Connect to a Server

Once a connection has been added, the connection name can be passed directly to `otssh`.

For example:

```bash
otssh server3
```

OTSSH searches the database for `server3`, retrieves its stored configuration, constructs the appropriate SSH command, and executes it.

For a connection configured with:

```text
server3|192.168.46.34|ubuntu|2022|~/.ssh/server3.pem
```

OTSSH executes the equivalent of:

```bash
ssh -i ~/.ssh/server3.pem -p 2022 ubuntu@192.168.46.34
```

The user is then connected directly to the remote server.

### Screenshot
<img width="1029" height="653" alt="image" src="https://github.com/user-attachments/assets/1d941f6f-f8e2-4e7c-80ae-4b6909e285c0" />


---



# 11. Duplicate Connection Handling

OTSSH prevents multiple connections from being created with the same name.

For example, if `server1` already exists:

```bash
otssh -a -n server1 -h 192.168.21.30 -u kirti
```

The utility displays:

```text
Server with this Name: server1 already exist
```

This prevents duplicate connection names from being stored in the database.

### Screenshot

<img width="1044" height="108" alt="image" src="https://github.com/user-attachments/assets/cb4964ea-ec92-47d5-83d9-0d691c10e581" />


---


---

# 14. Command Summary

```text
Add connection:

otssh -a -n <name> -h <host> -u <user> [-p <port>] [-i <key>]


List connections:

otssh ls


List connections with details:

otssh ls -d


Update connection:

otssh -u -n <name> -h <host> -u <user> [-p <port>] [-i <key>]


Delete connection:

otssh rm <name>


Connect:

otssh <name>
```

---


