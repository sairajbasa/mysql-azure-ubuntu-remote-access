# mysql-azure-ubuntu-remote-access
# MySQL Installation and Remote Access on Azure Ubuntu VM

This guide explains how to install **MySQL Server** on an **Ubuntu VM in Microsoft Azure**, configure it to accept remote connections, create a remote MySQL user, and allow MySQL traffic securely through the Ubuntu firewall and Azure Network Security Group (NSG).

## Architecture

```text
Local Machine
     |
     | TCP 3306
     v
Azure NSG
     |
     v
Ubuntu VM
     |
     | TCP 3306
     v
MySQL Server
```

> **Security recommendation:** Do not allow port `3306` from `Any` (`0.0.0.0/0`) unless you have a specific reason. Restrict the source to the public IP/CIDR of the machine or network that needs database access.

---

## Prerequisites

Before starting, make sure you have:

- An Azure Ubuntu VM
- SSH access to the VM
- `sudo` privileges
- A public IP on the VM if connecting from the Internet
- An Azure NSG associated with the VM's NIC or subnet
- The public IP address of the client machine from which you will connect

---

# 1. Update Ubuntu Package Repository

SSH into the Azure Ubuntu VM and update the local package index:

```bash
sudo apt update
```

---

# 2. Install MySQL Server

Install the MySQL server package:

```bash
sudo apt install mysql-server
```

Press `Y` when prompted.

Verify the installed MySQL version:

```bash
mysql --version
```

---

# 3. Start MySQL Service

Start MySQL:

```bash
sudo systemctl start mysql
```

---

# 4. Enable MySQL at Boot

Enable MySQL so that it starts automatically whenever the VM is rebooted:

```bash
sudo systemctl enable mysql
```

---

# 5. Verify MySQL Service

Check the MySQL service status:

```bash
sudo systemctl status mysql
```

You should see:

```text
Active: active (running)
```

Press `q` to exit the status screen.

You can also verify the service with:

```bash
sudo systemctl is-active mysql
```

Expected output:

```text
active
```

---

# 6. Secure MySQL Installation

Run the MySQL security script:

```bash
sudo mysql_secure_installation
```

This helps remove or disable insecure default settings such as:

- Anonymous MySQL users
- Remote root access
- Test databases
- Other default security risks

Follow the prompts according to your security requirements.

> **Note:** The exact prompts can vary depending on the Ubuntu/MySQL version and authentication configuration.

---

# 7. Configure MySQL for Remote Connections

By default, MySQL commonly listens only on the local loopback address:

```text
127.0.0.1
```

To allow connections from remote systems, edit the MySQL server configuration:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Find:

```ini
bind-address = 127.0.0.1
```

Change it to:

```ini
bind-address = 0.0.0.0
```

Save the file:

```text
Ctrl + O
Enter
Ctrl + X
```

### Security Note

`0.0.0.0` means MySQL can listen on all available network interfaces.

This does **not** mean that everyone on the Internet should be allowed to connect. Access should still be restricted using:

1. Azure NSG rules
2. Ubuntu firewall rules
3. MySQL user host permissions
4. Strong passwords and appropriate privileges

For a production environment, a more restrictive network design is preferred.

---

# 8. Restart MySQL

Apply the configuration changes:

```bash
sudo systemctl restart mysql
```

Verify:

```bash
sudo systemctl status mysql
```

---

# 9. Verify MySQL Listening Port

MySQL normally listens on TCP port:

```text
3306
```

Check whether MySQL is listening:

```bash
sudo ss -lntp | grep 3306
```

You may see something similar to:

```text
LISTEN 0 70 0.0.0.0:3306 0.0.0.0:*
```

This indicates that MySQL is listening on port `3306`.

---

# 10. Create a MySQL User for Remote Access

Connect to MySQL:

```bash
sudo mysql
```

> Depending on the Ubuntu/MySQL authentication configuration, `sudo mysql -u root -p` may not work as expected if the root account uses socket authentication. Using `sudo mysql` is commonly the simplest way to access the local MySQL administrative account.

Create a remote user:

```sql
CREATE USER 'your_user'@'%' IDENTIFIED BY 'strong_password';
```

Grant the required privileges.

For example, to grant access to a specific database:

```sql
GRANT ALL PRIVILEGES ON your_database.* TO 'your_user'@'%';
```

If you intentionally need full administrative access:

```sql
GRANT ALL PRIVILEGES ON *.* TO 'your_user'@'%';
```

Check the user's privileges:

```sql
SHOW GRANTS FOR 'your_user'@'%';
```

Exit MySQL:

```sql
EXIT;
```

### Important Security Recommendation

Avoid using:

```sql
'your_user'@'%'
```

when you can restrict the user to a specific source IP.

For example:

```sql
CREATE USER 'your_user'@'203.0.113.10' IDENTIFIED BY 'strong_password';
```

This allows that MySQL user to connect only from the specified source IP.

Also follow the principle of least privilege. A normal application user generally should not receive:

```sql
GRANT ALL PRIVILEGES ON *.* ...
```

---

# 11. Configure Ubuntu UFW

First check whether UFW is enabled:

```bash
sudo ufw status
```

If UFW is enabled, allow MySQL TCP port `3306`.

For a specific source IP, prefer:

```bash
sudo ufw allow from <CLIENT_PUBLIC_IP> to any port 3306 proto tcp
```

Example:

```bash
sudo ufw allow from 203.0.113.10 to any port 3306 proto tcp
```

Avoid unnecessarily opening MySQL to everyone:

```bash
sudo ufw allow 3306/tcp
```

If you use this rule, the Ubuntu VM may accept MySQL traffic from any source that can reach it at the network layer.

Verify:

```bash
sudo ufw status numbered
```

---

# 12. Allow Port 3306 in Azure NSG

Azure Network Security Groups control network traffic to and from resources such as VM NICs and subnets.

To allow MySQL remote access:

1. Open the **Azure Portal**.
2. Navigate to your **Virtual Machine**.
3. Select **Networking**.
4. Select the relevant **Network Security Group**.
5. Go to **Inbound security rules**.
6. Select **Add**.
7. Configure the rule.

Recommended configuration:

| Setting | Value |
|---|---|
| Source | IP Addresses |
| Source IP addresses/CIDR ranges | Your client public IP/CIDR |
| Source port ranges | `*` |
| Destination | Any / VM |
| Destination port ranges | `3306` |
| Protocol | TCP |
| Action | Allow |
| Priority | Choose an appropriate unused priority |
| Name | `Allow-MySQL-3306` |

Save the rule.

### Example

If your client public IP is:

```text
203.0.113.10
```

Allow:

```text
Source: 203.0.113.10
Destination port: 3306
Protocol: TCP
Action: Allow
```

> **Do not use `Any` as the source for a production MySQL server unless you fully understand the security implications.**

---

# 13. Verify the Network Path

The connection must pass through multiple layers:

```text
Client
  |
  | TCP 3306
  v
Azure NSG
  |
  v
Azure VM NIC
  |
  v
Ubuntu UFW
  |
  v
MySQL
```

All required layers must allow the connection.

---

# 14. Test MySQL from the Remote Machine

From your local machine, use the Azure VM's public IP:

```bash
mysql -h <AZURE_VM_PUBLIC_IP> -P 3306 -u your_user -p
```

Example:

```bash
mysql -h 20.x.x.x -P 3306 -u your_user -p
```

Enter the password when prompted.

If the connection succeeds, you should get the MySQL prompt:

```text
mysql>
```

---

# 15. Troubleshooting

## Check MySQL Service

```bash
sudo systemctl status mysql
```

## Check Port 3306

```bash
sudo ss -lntp | grep 3306
```

## Check MySQL Configuration

```bash
sudo grep -E '^[[:space:]]*bind-address' /etc/mysql/mysql.conf.d/mysqld.cnf
```

Expected:

```text
bind-address = 0.0.0.0
```

## Check UFW

```bash
sudo ufw status
```

## Check Azure NSG

Verify that:

- TCP port `3306` is allowed
- Source IP is correct
- NSG is associated with the VM NIC or subnet
- The rule priority is not overridden by a higher-priority deny rule

## Test Port Connectivity from Client

Linux/macOS:

```bash
nc -vz <AZURE_VM_PUBLIC_IP> 3306
```

Windows PowerShell:

```powershell
Test-NetConnection <AZURE_VM_PUBLIC_IP> -Port 3306
```

If the port test fails, troubleshoot the network/firewall path before troubleshooting MySQL authentication.

---

# 16. Common Problems

### Problem 1: Connection timed out

Possible causes:

- Azure NSG does not allow TCP `3306`
- UFW blocks TCP `3306`
- VM has no reachable public IP
- Network routing issue
- MySQL is not listening on the expected interface/port

### Problem 2: Connection refused

Possible causes:

- MySQL service is stopped
- MySQL is not listening on port `3306`
- Incorrect `bind-address`
- MySQL configuration contains an error

Check:

```bash
sudo systemctl status mysql
sudo ss -lntp | grep 3306
```

### Problem 3: Access denied for user

Example:

```text
ERROR 1045 (28000): Access denied for user
```

Possible causes:

- Incorrect username/password
- User was not created for the connecting host
- Insufficient privileges
- MySQL authentication configuration

Check:

```sql
SELECT user, host FROM mysql.user;
```

---

# 17. Recommended Production Architecture

For production environments, avoid exposing MySQL directly to the public Internet whenever possible.

A better architecture is:

```text
Internet
   |
   v
Application / Web Tier
   |
   | Private Network
   v
MySQL Server
```

For Azure, consider placing the database VM in a private subnet and allowing MySQL traffic only from the application tier.

Example:

```text
                 Azure VNet
------------------------------------------------
|                                              |
|  Web/App Subnet          DB Subnet           |
|  ---------------         ---------           |
|  Application VM  ------> MySQL VM            |
|                    TCP 3306                   |
|                                              |
------------------------------------------------
```

The NSG can then restrict:

```text
Source: Application subnet / application workload
Destination: MySQL VM
Port: 3306
Protocol: TCP
Action: Allow
```

This is significantly safer than exposing port `3306` to the public Internet.

---

# 18. Quick Command Summary

```bash
# Update packages
sudo apt update

# Install MySQL
sudo apt install mysql-server

# Start MySQL
sudo systemctl start mysql

# Enable at boot
sudo systemctl enable mysql

# Check status
sudo systemctl status mysql

# Secure installation
sudo mysql_secure_installation

# Edit MySQL configuration
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

# Restart MySQL
sudo systemctl restart mysql

# Check listening port
sudo ss -lntp | grep 3306

# Access MySQL locally
sudo mysql

# Check UFW
sudo ufw status

# Allow MySQL from a specific source IP
sudo ufw allow from <CLIENT_PUBLIC_IP> to any port 3306 proto tcp
```

---

# 19. MySQL SQL Commands Summary

```sql
-- Create a remote user
CREATE USER 'your_user'@'%' IDENTIFIED BY 'strong_password';

-- Grant access to a specific database
GRANT ALL PRIVILEGES ON your_database.* TO 'your_user'@'%';

-- Check privileges
SHOW GRANTS FOR 'your_user'@'%';

-- View MySQL users and allowed hosts
SELECT user, host FROM mysql.user;

-- Exit
EXIT;
```

---

## Key Takeaways

1. **MySQL Server** runs inside the Azure Ubuntu VM.
2. MySQL normally uses TCP port **3306**.
3. `bind-address` controls which network interfaces MySQL listens on.
4. `0.0.0.0` makes MySQL listen on all IPv4 interfaces.
5. **Azure NSG** controls whether network traffic can reach the VM.
6. **UFW** controls traffic at the Ubuntu operating-system level.
7. MySQL users also have host restrictions such as `'user'@'%'`.
8. Do not expose port `3306` to the entire Internet unnecessarily.
9. Prefer restricting access to specific IPs or application workloads.
10. For production, prefer a **private database network** rather than exposing MySQL through a public IP.

---

## License

This documentation is provided for learning and educational purposes.
