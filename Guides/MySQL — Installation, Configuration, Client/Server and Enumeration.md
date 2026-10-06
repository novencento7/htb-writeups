# MySQL — Installation, Configuration, Client/Server and Enumeration

## 1. What is MySQL?

**MySQL** is an open-source relational database management system (RDBMS) developed and supported by Oracle.

A database is a structured collection of data organized so that it can be stored, retrieved, and modified efficiently.

MySQL uses **SQL (Structured Query Language)** to query and modify data.

MySQL follows the **client-server model** and consists of a MySQL server and one or more clients.

```text
┌────────────────────┐
│   MySQL Client     │
│                    │
│  mysql             │
│  Application       │
│  PHP / Python      │
└─────────┬──────────┘
          │
          │ TCP/IP
          │ Port 3306
          ▼
┌────────────────────┐
│   MySQL Server     │
│                    │
│  mysqld            │
│                    │
│  ┌──────────────┐  │
│  │  Databases   │  │
│  │  Tables      │  │
│  │  Data        │  │
│  └──────────────┘  │
└────────────────────┘
```

The **MySQL server** manages the databases and their data, while the **client** sends SQL queries to the server and receives the results.

---

# 2. Databases, Tables, Rows and Columns

A MySQL server can contain multiple databases:

```text
MySQL Server
│
├── Database
│   │
│   ├── Table
│   │   ├── Columns
│   │   ├── Rows
│   │   └── Data
│   │
│   └── Table
│
└── Database
```

For example:

```text
database: company

users
+----+----------+---------------------+
| id | username | email               |
+----+----------+---------------------+
|  1 | mario    | mario@example.com   |
|  2 | luca     | luca@example.com    |
+----+----------+---------------------+
```

A database can contain many tables, and each table can contain many rows.

---

# 3. MySQL and Web Applications

MySQL is commonly used with web servers and server-side programming languages.

A traditional example is the **LAMP stack**:

```text
Linux
Apache
MySQL
PHP
```

A typical web application can be represented as:

```text
Browser
   │
   ▼
Web Server
   │
   ▼
Application
(PHP / Python / ...)
   │
   │ SQL
   ▼
MySQL Server
   │
   ▼
Database
```

The browser normally does **not** communicate directly with MySQL.

Instead:

1. The browser sends an HTTP request.
2. The web server receives it.
3. The application processes the request.
4. The application sends an SQL query to MySQL.
5. MySQL processes the query.
6. The application receives the result.
7. The web server sends the response to the browser.

This separation is important from both an architectural and security perspective.

---

# 4. Installing the MySQL Server

On Ubuntu/Debian:

```bash
sudo apt update
sudo apt install mysql-server -y
```

Check the service:

```bash
sudo systemctl status mysql
```

If it is not running:

```bash
sudo systemctl start mysql
```

Enable MySQL at boot:

```bash
sudo systemctl enable mysql
```

Check the installed version:

```bash
mysql --version
```

Example:

```text
mysql  Ver 8.0.x for Linux on x86_64
```

---

# 5. Installing the MySQL Client

The client is the program used to connect to a MySQL server.

On Ubuntu/Debian:

```bash
sudo apt install mysql-client -y
```

Check the version:

```bash
mysql --version
```

The client and server are separate components.

For example:

```text
Client Machine
┌─────────────────┐
│ MySQL Client    │
│ mysql           │
└────────┬────────┘
         │
         │ TCP 3306
         ▼
Server Machine
┌─────────────────┐
│ MySQL Server    │
│ mysqld          │
│                 │
│ Databases       │
└─────────────────┘
```

The MySQL client does not need to contain the database itself.

---

# 6. Connecting Locally

On a machine where the MySQL server is installed:

```bash
sudo mysql
```

Once connected, the prompt should look similar to:

```text
mysql>
```

Check the server version:

```sql
SELECT VERSION();
```

or:

```sql
SELECT @@version;
```

---

# 7. Creating a Database

Create a database for testing:

```sql
CREATE DATABASE lab;
```

List the available databases:

```sql
SHOW DATABASES;
```

Select the database:

```sql
USE lab;
```

---

# 8. Creating a Table

Create a simple `users` table:

```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL
);
```

List the tables:

```sql
SHOW TABLES;
```

---

# 9. Inserting Data

Insert a record:

```sql
INSERT INTO users (username, email)
VALUES ('mario', 'mario@example.com');
```

Insert another:

```sql
INSERT INTO users (username, email)
VALUES ('luca', 'luca@example.com');
```

Display the data:

```sql
SELECT * FROM users;
```

Example:

```text
+----+----------+-------------------+
| id | username | email             |
+----+----------+-------------------+
|  1 | mario    | mario@example.com |
|  2 | luca     | luca@example.com  |
+----+----------+-------------------+
```

---

# 10. Updating Data

Modify an existing record:

```sql
UPDATE users
SET email = 'mario2@example.com'
WHERE username = 'mario';
```

Verify the change:

```sql
SELECT * FROM users;
```

---

# 11. Deleting Data

Delete a specific record:

```sql
DELETE FROM users
WHERE username = 'luca';
```

Be careful with:

```sql
DELETE FROM users;
```

Without a `WHERE` clause, this removes **all rows** from the table.

---

# 12. Inspecting Tables

Display the columns of a table:

```sql
SHOW COLUMNS FROM users;
```

or:

```sql
DESCRIBE users;
```

Display the SQL statement used to create the table:

```sql
SHOW CREATE TABLE users;
```

---

# 13. Creating a MySQL User

It is good practice to avoid using the administrative account for normal database operations.

Create a dedicated user:

```sql
CREATE USER 'labuser'@'localhost'
IDENTIFIED BY 'Strong-Password-123!';
```

List MySQL users:

```sql
SELECT user, host
FROM mysql.user;
```

---

# 14. Granting Privileges

Give the new user access to the `lab` database:

```sql
GRANT ALL PRIVILEGES
ON lab.*
TO 'labuser'@'localhost';
```

Check the privileges:

```sql
SHOW GRANTS FOR 'labuser'@'localhost';
```

The user now has privileges on the `lab` database without necessarily having administrative privileges over the entire MySQL server.

---

# 15. Connecting with the MySQL Client

Exit the MySQL shell:

```sql
EXIT;
```

Connect using the new account:

```bash
mysql -u labuser -p
```

The client will ask for the password:

```text
Enter password:
```

It is preferable to use `-p` without specifying the password directly on the command line.

Avoid:

```bash
mysql -u labuser -pMyPassword
```

because the password can potentially become visible in shell history or process information.

---

# 16. Connecting to a Remote MySQL Server

The general syntax is:

```bash
mysql -u USER -p -h SERVER_IP
```

Example:

```bash
mysql -u labuser -p -h 192.168.1.100
```

You can also specify the database:

```bash
mysql -u labuser -p -h 192.168.1.100 lab
```

The options mean:

```text
-u    MySQL username
-p    Prompt for the password
-h    MySQL server hostname/IP
```

The connection looks like:

```text
Client
192.168.1.50
     │
     │ TCP 3306
     ▼
Server
192.168.1.100
     │
     ▼
lab database
```

---

# 17. MySQL Configuration

On Ubuntu, an important configuration file is:

```text
/etc/mysql/mysql.conf.d/mysqld.cnf
```

View the configuration:

```bash
sudo cat /etc/mysql/mysql.conf.d/mysqld.cnf
```

Or remove comments and empty lines:

```bash
sudo cat /etc/mysql/mysql.conf.d/mysqld.cnf \
| grep -v "#" \
| sed -r '/^\s*$/d'
```

A typical configuration contains:

```ini
[mysqld]

user = mysql
port = 3306
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock
```

Important configuration parameters include:

```text
user
port
datadir
socket
bind-address
secure_file_priv
```

---

# 18. Unix Socket vs TCP/IP

MySQL can communicate locally through a Unix socket:

```text
/var/run/mysqld/mysqld.sock
```

or through TCP/IP:

```text
SERVER_IP:3306
```

When running:

```bash
mysql -u labuser -p
```

the client can use the local Unix socket.

When running:

```bash
mysql -u labuser -p -h 192.168.1.100
```

the client explicitly requests a TCP/IP connection.

This distinction is important when troubleshooting remote connections.

---

# 19. Configuring Remote Connections

Check the `bind-address` setting:

```ini
bind-address = 127.0.0.1
```

This restricts MySQL to the local machine.

For a lab environment, it could be configured to listen on a specific network interface:

```ini
bind-address = 192.168.1.100
```

Alternatively:

```ini
bind-address = 0.0.0.0
```

allows MySQL to listen on all network interfaces.

> **Warning:** Do not expose MySQL to untrusted networks without appropriate firewall rules, authentication, and access restrictions.

After changing the configuration:

```bash
sudo systemctl restart mysql
```

Check the listening socket:

```bash
sudo ss -lntp | grep 3306
```

Example:

```text
LISTEN 0 151 192.168.1.100:3306 0.0.0.0:*
```

---

# 20. MySQL User and Source Host

A MySQL account is associated with both a username and a host.

For example:

```text
'labuser'@'localhost'
```

is different from:

```text
'labuser'@'192.168.1.50'
```

and from:

```text
'labuser'@'%'
```

For example:

```sql
CREATE USER 'labuser'@'192.168.1.50'
IDENTIFIED BY 'Strong-Password-123!';
```

This restricts that account to connections originating from the specified host.

Using `%` is considerably broader:

```sql
CREATE USER 'labuser'@'%'
IDENTIFIED BY 'Strong-Password-123!';
```

For production systems, access should normally be restricted as much as possible.

---

# 21. Firewall Configuration

If remote clients need access, TCP/3306 must also be allowed through the firewall.

With UFW:

```bash
sudo ufw allow from 192.168.1.50 to any port 3306 proto tcp
```

This is preferable to opening MySQL to everyone:

```bash
sudo ufw allow 3306/tcp
```

Check the firewall:

```bash
sudo ufw status
```

---

# 22. Testing the Connection

From the client:

```bash
mysql -u labuser -p -h 192.168.1.100
```

Before attempting authentication, you can test whether TCP/3306 is reachable:

```bash
nc -vz 192.168.1.100 3306
```

or:

```bash
nmap -p3306 192.168.1.100
```

If Nmap reports:

```text
3306/tcp open mysql
```

the service is reachable.

This does **not** mean that authentication will succeed.

---

# 23. MySQL Enumeration with Nmap

MySQL normally uses:

```text
TCP/3306
```

Basic service detection:

```bash
nmap -p3306 -sV 192.168.1.100
```

More detailed enumeration:

```bash
nmap -p3306 -sV -sC --script mysql* 192.168.1.100
```

Only perform enumeration against systems you own or are explicitly authorized to test.

---

# 24. Understanding `mysql-info`

Example:

```text
3306/tcp open mysql
MySQL 8.0.27-0ubuntu0.20.04.1

| mysql-info:
|   Protocol: 10
|   Version: 8.0.27-0ubuntu0.20.04.1
|   Thread ID: 10
|   Capabilities flags: 65535
|   Status: Autocommit
|   Auth Plugin Name: caching_sha2_password
```

### Protocol

```text
Protocol: 10
```

This identifies the MySQL client/server protocol version.

### Version

```text
Version: 8.0.27-0ubuntu0.20.04.1
```

This identifies the server version reported during the handshake.

### Thread ID

```text
Thread ID: 10
```

This identifies the server thread/connection associated with the connection.

### Capabilities

```text
Capabilities flags: 65535
```

These flags describe protocol capabilities supported by the server.

Examples include:

```text
SupportsTransactions
SupportsCompression
SupportsMultipleResults
SupportsAuthPlugins
ConnectWithDatabase
```

### Authentication Plugin

```text
Auth Plugin Name: caching_sha2_password
```

This identifies the authentication plugin advertised by the server.

---

# 25. Example: MySQL 8.0.27

A scan such as:

```text
3306/tcp open mysql
MySQL 8.0.27-0ubuntu0.20.04.1
```

tells us that:

* TCP port `3306` is open.
* Nmap identifies the service as MySQL.
* The server reports version `8.0.27`.
* The package is associated with Ubuntu 20.04.
* Additional protocol information can be obtained through `mysql-info`.

For example:

```text
Protocol: 10
Thread ID: 10
Status: Autocommit
Auth Plugin Name: caching_sha2_password
```

Nmap results should always be manually verified where possible. Service detection and NSE scripts can produce false positives.

---

# 26. Connecting to an Enumerated Server

If valid credentials are available:

```bash
mysql -u USER -p -h SERVER_IP
```

For example:

```bash
mysql -u root -p -h 10.129.14.128
```

If authentication fails, MySQL may return:

```text
ERROR 1045 (28000):
Access denied for user ...
```

The important distinction is:

```text
Service reachable
        ≠
Successful authentication
        ≠
Administrative privileges
```

---

# 27. Useful SQL Commands

Once authenticated:

### List databases

```sql
SHOW DATABASES;
```

### Select a database

```sql
USE database_name;
```

### List tables

```sql
SHOW TABLES;
```

### Describe a table

```sql
DESCRIBE table_name;
```

### Display all rows

```sql
SELECT * FROM table_name;
```

### Search for a specific value

```sql
SELECT *
FROM table_name
WHERE column_name = 'value';
```

### Display the server version

```sql
SELECT VERSION();
```

These commands form the basic workflow for interacting with a MySQL database.

---

# 28. System Databases

A standard MySQL installation includes several important system databases:

```text
information_schema
mysql
performance_schema
sys
```

### `information_schema`

Provides metadata about databases, tables, columns, privileges, and other objects.

### `mysql`

Contains information used by the MySQL server for administration and account management.

### `performance_schema`

Provides performance and runtime instrumentation.

### `sys`

Provides convenient views for analyzing and administering MySQL.

For example:

```sql
USE sys;
SHOW TABLES;
```

---

# 29. Listing MySQL Users

With sufficient privileges:

```sql
SELECT user, host
FROM mysql.user;
```

Example:

```text
+----------+-----------+
| user     | host      |
+----------+-----------+
| root     | localhost |
| labuser  | localhost |
+----------+-----------+
```

Remember that:

```text
'user'@'localhost'
```

and:

```text
'user'@'192.168.1.50'
```

are different MySQL accounts.

---

# 30. Checking Privileges

Check the privileges of an account:

```sql
SHOW GRANTS FOR 'labuser'@'localhost';
```

For example:

```text
GRANT ALL PRIVILEGES
ON lab.*
TO 'labuser'@'localhost'
```

This grants privileges on the `lab` database rather than automatically granting administrative access to the entire server.

---

# 31. Security-Relevant Configuration

Several MySQL configuration options are particularly relevant to security:

```text
user
password
admin_address
debug
sql_warnings
secure_file_priv
```

Configuration files should have appropriate permissions and should not expose credentials unnecessarily.

Check the permissions of the MySQL configuration:

```bash
ls -l /etc/mysql/mysql.conf.d/mysqld.cnf
```

---

# 32. `secure_file_priv`

Check the current value:

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

or:

```sql
SELECT @@secure_file_priv;
```

`secure_file_priv` can restrict certain file import/export operations.

---

# 33. Checking Server Variables

Display all variables:

```sql
SHOW VARIABLES;
```

Check a specific variable:

```sql
SHOW VARIABLES LIKE 'port';
```

```sql
SHOW VARIABLES LIKE 'bind_address';
```

```sql
SHOW VARIABLES LIKE 'secure_file_priv';
```

You can also query individual values:

```sql
SELECT @@port;
```

---

# 34. Restarting MySQL

After modifying the configuration:

```bash
sudo systemctl restart mysql
```

Check the service:

```bash
sudo systemctl status mysql
```

Check the listening port:

```bash
sudo ss -lntp | grep 3306
```

---

# 35. Troubleshooting

## MySQL does not start

Check:

```bash
sudo systemctl status mysql
```

and:

```bash
sudo journalctl -u mysql
```

## Port 3306 is not listening

Check:

```bash
sudo ss -lntp | grep 3306
```

## Local connections work, but remote connections fail

Check:

```text
bind-address
```

Then check the firewall:

```bash
sudo ufw status
```

Finally, check the MySQL account and its host restriction:

```sql
SHOW GRANTS FOR 'labuser'@'192.168.1.50';
```

## `Access denied`

Check:

```text
Username
Password
Host
Privileges
Authentication plugin
```

An authentication failure does not necessarily mean that the MySQL server is unreachable.

---

# 36. Complete Lab Architecture

A useful laboratory setup looks like this:

```text
                         LAN
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
     ┌──────────────┐          ┌────────────────┐
     │ MySQL Client │          │ MySQL Server   │
     │              │          │                │
     │ mysql        │          │ mysqld         │
     │              │          │                │
     │ 192.168.1.50 │          │ 192.168.1.100  │
     └──────┬───────┘          │ TCP 3306       │
            │                  │                │
            │                  │ lab            │
            └─────────────────►│ ├── users      │
                               │ └── ...        │
                               └────────────────┘
```

The complete flow is:

```text
MySQL Client
     │
     │ TCP/IP
     ▼
MySQL Server
     │
     │ Authentication
     ▼
MySQL Account
     │
     │ Authorization
     ▼
Database
     │
     ▼
Tables
     │
     ▼
Rows
```

The key distinction is:

```text
Authentication
      │
      ▼
"Who are you?"

      │
      ▼

Authorization
      │
      ▼
"What are you allowed to do?"
```

---

# 37. MySQL Enumeration Workflow

When discovering a MySQL service during an authorized security assessment, a useful workflow is:

```text
1. Identify the port
        │
        ▼
   TCP/3306?
        │
        ▼
2. Identify the service
        │
        ▼
   MySQL?
        │
        ▼
3. Identify the version
        │
        ▼
   MySQL 8.0.x?
        │
        ▼
4. Examine configuration
        │
        ├── bind-address
        ├── port
        ├── authentication
        └── secure_file_priv
        │
        ▼
5. Authentication
        │
        ├── Username
        ├── Password
        └── Authentication plugin
        │
        ▼
6. Authorization
        │
        ├── Databases
        ├── Tables
        └── Privileges
        │
        ▼
7. Data
```

---

# 38. Essential Commands Cheat Sheet

## Installation

```bash
sudo apt update
sudo apt install mysql-server mysql-client -y
```

## Service

```bash
sudo systemctl status mysql
sudo systemctl start mysql
sudo systemctl enable mysql
```

## Local connection

```bash
sudo mysql
```

## Remote connection

```bash
mysql -u USER -p -h SERVER_IP
```

## Databases

```sql
SHOW DATABASES;
USE database_name;
```

## Tables

```sql
SHOW TABLES;
DESCRIBE table_name;
```

## Data

```sql
SELECT * FROM table_name;
```

## Users

```sql
SELECT user, host FROM mysql.user;
```

## Privileges

```sql
SHOW GRANTS FOR 'user'@'host';
```

## Configuration

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

## Listening port

```bash
sudo ss -lntp | grep 3306
```

## Restart

```bash
sudo systemctl restart mysql
```

## Nmap

```bash
nmap -p3306 -sV SERVER_IP
```

More detailed:

```bash
nmap -p3306 -sV -sC --script mysql* SERVER_IP
```

---

# 39. Key Concepts to Remember

The most important concepts are:

```text
MySQL Server
    │
    ├── mysqld
    ├── Databases
    ├── Tables
    └── Data

MySQL Client
    │
    └── Sends SQL queries to the server
```

MySQL normally listens on:

```text
TCP/3306
```

The client connects using:

```bash
mysql -u USER -p -h SERVER_IP
```

A successful TCP connection does not imply successful authentication.

A successful authentication does not imply administrative privileges.

And having access to a database does not necessarily mean having access to every database on the server.

This distinction between **network access, authentication, authorization, and database access** is fundamental when working with MySQL.
