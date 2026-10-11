# Oracle TNS Enumeration and Security Testing

## 1. Introduction

Oracle Transparent Network Substrate (TNS) is Oracle's networking technology that allows client applications to communicate with Oracle databases.

For example, a client running SQL*Plus can connect to a remote Oracle database through the network.

The main components are:

- **Client:** The application used to connect to the database, such as SQL*Plus.
- **Listener:** A server-side process that listens for incoming connection requests and directs them to the appropriate Oracle service.
- **Oracle Database:** The database system that authenticates users, processes SQL queries, and manages data.
- **SID:** The System Identifier, which identifies a specific Oracle database instance.
- **SERVICE_NAME:** The name of the Oracle service requested by the client.

### How a connection works

1. The client contacts the Oracle listener, usually on TCP port 1521.
2. The listener identifies the requested service and helps establish the connection.
3. Oracle authenticates the user.
4. The client can execute SQL queries according to the user's privileges.

The listener normally runs on the Oracle server, not on the client machine. Once a connection is established, the client may communicate directly with the server process handling its session.

**Important:** The listener handles connection requests. It does not normally authenticate database users itself or execute their SQL queries.

## 2. Default Port

Oracle TNS commonly uses:

| Port | Protocol | Service |
|---|---|---|
| 1521 | TCP | Oracle TNS Listener |

The port can be changed by the administrator.

Finding port 1521 open is an indication that an Oracle listener may be running, but service detection should be used to confirm it.

## 3. Oracle Configuration Files

Oracle Net Services uses configuration files to define connections and listener behavior.

### tnsnames.ora

The `tnsnames.ora` file is generally used on the client side. It maps a local connection alias to the network address and service information required to connect to an Oracle database.

Example:

```text
ORCL =
  (DESCRIPTION =
    (ADDRESS =
      (PROTOCOL = TCP)
      (HOST = 10.129.11.102)
      (PORT = 1521)
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
```

Explanation:

- `ORCL`: Local connection alias.
- `PROTOCOL`: Network protocol.
- `HOST`: Database server hostname or IP address.
- `PORT`: Listener port.
- `CONNECT_DATA`: Information identifying the requested connection.
- `SERVER`: Connection mode, such as a dedicated server process.
- `SERVICE_NAME`: Oracle service requested by the client.

The alias `ORCL` is not necessarily the SID or the service name. It is a local name used to reference this connection configuration.

### listener.ora

The `listener.ora` file configures the listener on the server.

A simplified example:

```text
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS =
        (PROTOCOL = TCP)
        (HOST = orcl.inlanefreight.htb)
        (PORT = 1521)
      )
    )
  )
```

This configuration defines a listener endpoint using TCP port 1521.

The file can also contain information about database services and other listener settings.

### Configuration directory

Oracle configuration files are commonly located in:

```text
$ORACLE_HOME/network/admin
```

The actual location depends on the Oracle installation and environment configuration.

### Other configuration parameters

| Parameter | Purpose |
|---|---|
| `DESCRIPTION` | Describes a connection. |
| `ADDRESS` | Defines a network endpoint. |
| `PROTOCOL` | Specifies the network protocol. |
| `HOST` | Specifies the server hostname or IP address. |
| `PORT` | Specifies the network port. |
| `CONNECT_DATA` | Contains connection information. |
| `INSTANCE_NAME` | Identifies an Oracle instance by name. |
| `SERVICE_NAME` | Identifies the requested Oracle service. |
| `USER` | Database username, where supported by the configuration. |
| `PASSWORD` | Database password, where supported by the configuration. |
| `CONNECT_TIMEOUT` | Limits the time allowed to establish a connection. |
| `SEND_TIMEOUT` | Limits the time allowed for sending data. |
| `RECEIVE_TIMEOUT` | Limits the time allowed for receiving data. |
| `TRACE_LEVEL` | Controls diagnostic tracing. |
| `TRACE_DIRECTORY` | Specifies where trace files are stored. |

Not every parameter is valid in every configuration file or supported in every Oracle version.

## 4. SID vs. SERVICE_NAME

These concepts are important when enumerating Oracle during a penetration test.

### SID: System Identifier

A SID identifies a particular Oracle database instance.

An instance consists of memory structures and background processes used to manage the database.

### SERVICE_NAME

A service name identifies an Oracle service that clients request when connecting.

One service may be associated with one or more database instances, depending on the Oracle configuration.

### Why is the SID important?

A server can host multiple Oracle instances. Identifying the correct instance or service helps determine which database endpoint should be targeted during enumeration.

However, knowing the SID does not provide database access. Authentication and authorization are still required.

### Example

```text
Server: 10.129.204.235
Listener: TCP/1521
SID discovered: XE
```

The identifier `XE` was discovered during enumeration. In the example connection later in this guide, `XE` is also used as the service identifier in the connection string.

Do not assume that the SID and SERVICE_NAME must always be identical.

## 5. Scanning Oracle with Nmap

The first step in an HTB lab is to check whether the Oracle listener is reachable.

```bash
sudo nmap -p 1521 -sV --open 10.129.204.235
```

Options:

- `-p 1521`: Scan TCP port 1521.
- `-sV`: Attempt to identify the service and its version.
- `--open`: Display open ports.

Example output:

```text
PORT     STATE SERVICE    VERSION
1521/tcp open  oracle-tns Oracle TNS listener 11.2.0.2.0
```

This indicates that Nmap detected an Oracle TNS listener on the target.

The version shown is the version detected by Nmap; it should not automatically be treated as proof of the exact patch level installed.

## 6. Enumerating SIDs with Nmap

The Oracle SID brute-force script attempts to identify valid SID names from a list of candidates.

```bash
sudo nmap -p 1521 -sV \
  --script oracle-sid-brute \
  10.129.204.235
```

Example output:

```text
1521/tcp open oracle-tns
| oracle-sid-brute:
|_ XE
```

In this example, the script discovered the SID `XE`.

This is useful because the client may need the correct SID or service information to establish a connection.

Only run these checks against systems you are authorized to test.

## 7. ODAT: Oracle Database Attacking Tool

ODAT is an open-source Python penetration-testing tool designed to enumerate and test Oracle databases.

Depending on the target configuration and available privileges, it can help identify:

- Oracle services and database information.
- Valid credentials.
- Misconfigurations.
- Accessible database functionality.
- File-access capabilities.
- Potential privilege-related weaknesses.

ODAT is available from:

https://github.com/quentinhardy/odat

### Installing ODAT

The original lab instructions use Parrot OS and install Python dependencies and Oracle client libraries.

Start by installing common build dependencies:

```bash
sudo apt update
sudo apt install -y \
  git \
  build-essential \
  python3-dev \
  python3-pip \
  libaio1
```

Clone the repository:

```bash
git clone https://github.com/quentinhardy/odat.git
cd odat
```

Follow the installation instructions in the repository for your operating system and Oracle client version. ODAT may require additional Python dependencies and Oracle client libraries.

Older installation instructions sometimes use `cx_Oracle`, legacy Python build steps, or system-wide `pip` installations. These methods may not work with modern Python distributions, so do not assume the original commands are compatible with every current environment.

### Verify the installation

```bash
./odat.py -h
```

The help output should display the available commands and modules.

### Running ODAT

In the original HTB example:

```bash
./odat.py all -s 10.129.204.235
```

Here:

- `all`: Runs multiple ODAT checks.
- `-s`: Specifies the target server.

The command can take some time because multiple modules may be executed.

Example output:

```text
[+] Checking if target 10.129.204.235:1521
    is well configured for a connection...

[+] Valid credentials found: scott/tiger
```

The tool reports that the credentials `scott/tiger` were valid in this lab.

This does not mean that these credentials are valid on other Oracle servers.

For practical work, prefer targeted ODAT modules over running every module when you already know what you need to investigate.

## 8. Oracle Accounts and Default Passwords

Older Oracle installations and poorly maintained systems may contain weak or unchanged credentials.

The original tutorial mentions examples such as:

| Account or password | Context |
|---|---|
| `SCOTT` / `TIGER` | Legacy sample account credentials; valid in the lab example. |
| `DBSNMP` / `DBSNMP` | Historical default credentials associated with the DBSNMP account in some configurations. |
| `CHANGE_ON_INSTALL` | Historical password associated with older Oracle installations. |

These are historical examples, not universal defaults. Accounts may be locked, removed, renamed, or assigned different passwords.

An exposed listener does not necessarily mean that any default account can log in.

## 9. Installing SQL*Plus

SQL*Plus is a command-line client for connecting to Oracle and executing SQL statements.

On the Parrot OS environment used in the original tutorial, the package installation is shown as:

```bash
sudo apt update
sudo apt upgrade parrot-core
sudo apt install oracle-instantclient-sqlplus
```

Package names and availability vary by distribution and repository configuration. On other systems, install SQL*Plus using the appropriate Oracle Instant Client packages.

### Verify SQL*Plus

```bash
sqlplus -v
```

Example output:

```text
SQL*Plus: Release 19.0.0.0.0 - Production
Version 19.6.0.0.0
```

### Troubleshooting a missing shared library

If you see:

```text
sqlplus: error while loading shared libraries:
libsqlplus.so: cannot open shared object file
```

the operating system may not be able to find the Oracle client library.

Check the actual installation directory first. If the library is installed in a directory such as `/usr/lib/oracle/12.2/client64/lib`, you can register that directory with the dynamic linker:

```bash
echo /usr/lib/oracle/12.2/client64/lib \
  | sudo tee /etc/ld.so.conf.d/oracle-instantclient.conf

sudo ldconfig
```

Use the path matching your installed client version. The path above is an example from the original troubleshooting instructions, not a universal location.

## 10. Connecting to Oracle with SQL*Plus

The connection syntax used in the lab is:

```bash
sqlplus USERNAME/PASSWORD@HOST/SERVICE_NAME
```

Example:

```bash
sqlplus scott/tiger@10.129.204.235/XE
```

The parameters are:

- `scott`: Username.
- `tiger`: Password.
- `10.129.204.235`: Target IP address.
- `XE`: Service name used in this connection string.

Example output:

```text
Connected to:
Oracle Database 11g Express Edition
Release 11.2.0.2.0

SQL>
```

The `SQL>` prompt means that the client has established a session and is ready to accept SQL commands.

You might see a warning such as:

```text
ORA-28002: the password will expire within 7 days
```

This is a password-expiration warning. It does not necessarily mean that the connection failed.

### What happens during the connection?

1. SQL*Plus connects to the remote listener.
2. The connection requests the specified Oracle service.
3. The listener helps establish the connection to the appropriate Oracle instance.
4. Oracle authenticates the user.
5. SQL*Plus can execute queries permitted by that user's privileges.

The listener runs on the server. You do not need to run your own listener on the client machine just to connect to a remote database.

## 11. Enumerating the Database with SQL

After connecting, you can examine the database using SQL queries.

### List accessible tables

```sql
SELECT table_name FROM all_tables;
```

`ALL_TABLES` lists tables accessible to the current user, subject to Oracle's visibility rules.

It does not necessarily list every table in the entire database.

### Check the current user's roles

```sql
SELECT * FROM user_role_privs;
```

The original example returns roles such as:

```text
USERNAME   GRANTED_ROLE
SCOTT      CONNECT
SCOTT      RESOURCE
```

This shows that the user has those roles.

The presence of a role does not automatically imply administrative access. The privileges granted by each role determine what the user can do.

## 12. Understanding SYSDBA

`SYSDBA` is a highly privileged Oracle administrative privilege.

The original tutorial demonstrates the following connection:

```bash
sqlplus scott/tiger@10.129.204.235/XE as sysdba
```

If this succeeds, the resulting session has administrative privileges and typically operates as `SYS`.

However, adding `as sysdba` does not automatically elevate an ordinary database user. The connection must be authorized through the relevant Oracle authentication and privilege mechanisms.

The original lab output shows a session with extensive roles, including `DBA`.

This is an important distinction:

**A valid database account is not necessarily an administrator, and an ordinary account cannot automatically become SYSDBA.**

## 13. Understanding Database Password Hashes

The original tutorial demonstrates a query against Oracle's internal user table:

```sql
SELECT name, password FROM sys.user$;
```

Depending on the Oracle version, privileges, and authentication configuration, internal tables may contain password-verifier information.

The tutorial shows output with account names and hexadecimal-looking values.

Important considerations:

- Access to internal tables is restricted.
- Oracle password storage differs across versions.
- The legacy `PASSWORD` column does not necessarily contain the complete or current password-verifier information for every account.
- Extracting a stored verifier does not reveal the original password directly.

In an authorized lab, password-verifier exposure can indicate a security weakness. In a production environment, access to this information should be tightly restricted.

## 14. Oracle File Access and ODAT

Oracle includes functionality that may allow database operations to interact with files on the server.

Whether this is possible depends on the Oracle feature being used, database privileges, directory permissions, and operating-system permissions.

The original tutorial uses ODAT's `utlfile` module to demonstrate writing a harmless test file.

### Example: Create a local test file

```bash
echo "Oracle File Upload Test" > testing.txt
```

This creates `testing.txt` on the client machine.

### Example: Write the file on the lab target

The original Windows-target example is:

```bash
./odat.py utlfile -s 10.129.204.235 \
  -d XE -U scott -P tiger --sysdba \
  --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt
```

The options specify:

- `utlfile`: The ODAT module.
- `-s`: Target server.
- `-d`: Database identifier used by the tool.
- `-U`: Username.
- `-P`: Password.
- `--sysdba`: Request a SYSDBA connection.
- `--putFile`: Request that the local file be written to the specified remote directory.

This operation requires the appropriate Oracle access and file-system permissions. The exact command may also depend on the installed ODAT version.

The original output reports that the test file was created in:

```text
C:\inetpub\wwwroot\testing.txt
```

### Verify a test file through a web server

If the target has a web server configured to serve that directory, the original tutorial verifies the file using:

```bash
curl http://10.129.204.235/testing.txt
```

Expected content:

```text
Oracle File Upload Test
```

This works only if the web server is running, the directory is mapped to its document root, and the file is accessible over HTTP.

A successful database connection alone does not guarantee the ability to write files or execute commands on the operating system.

## 15. Common Oracle Security Misconfigurations

During an authorized security assessment, useful findings may include:

- Exposed Oracle listeners that should not be reachable from untrusted networks.
- Weak or unchanged database credentials.
- Unnecessary accounts that remain enabled.
- Excessive database privileges.
- Inappropriate SYSDBA access.
- Unnecessary file-access capabilities.
- Outdated Oracle software.
- Inadequate network segmentation or transport protection.

Oracle supports network security mechanisms, including encryption and TLS, when appropriately configured. Do not assume that all Oracle connections are encrypted merely because they use TNS.

The original tutorial also mentions the PL/SQL Exclusion List (`PlsqlExclusionList`). This is a specialized Oracle Application Server security mechanism used to exclude selected PL/SQL packages or types from execution; it is not a general replacement for database permissions or network access controls.

## 16. Quick HTB Workflow

Use this sequence as a practical reference during an authorized Oracle lab.

### Step 1: Find the listener

```bash
sudo nmap -p 1521 -sV --open TARGET_IP
```

### Step 2: Enumerate SIDs

```bash
sudo nmap -p 1521 \
  --script oracle-sid-brute TARGET_IP
```

### Step 3: Run targeted ODAT checks

```bash
./odat.py -h
```

Choose the appropriate module for the information you need to verify.

### Step 4: Connect with SQL*Plus

```bash
sqlplus USERNAME/PASSWORD@TARGET_IP/SERVICE_NAME
```

### Step 5: Examine accessible tables

```sql
SELECT table_name FROM all_tables;
```

### Step 6: Check roles

```sql
SELECT * FROM user_role_privs;
```

### Step 7: Assess additional privileges

Determine whether the account has administrative privileges or access to sensitive Oracle features. Do not assume that a normal login grants SYSDBA access or operating-system access.

---

## Key Takeaways

| Concept | What to remember |
|---|---|
| TNS | Oracle networking technology. |
| Listener | Server-side process that receives connection requests and helps direct them to the requested service. |
| Port 1521 | Default Oracle listener port in many installations. |
| SID | Identifies a specific Oracle instance. |
| SERVICE_NAME | Identifies the Oracle service requested by a client. |
| SQL*Plus | Command-line client used to connect and execute SQL. |
| ODAT | Tool for Oracle database enumeration and security testing. |
| Credentials | Determine whether a user can authenticate. |
| Privileges | Determine what the authenticated user can do. |
| SYSDBA | Highly privileged Oracle administrative access. |

**The essential idea:** Discover the listener, identify the relevant Oracle service, establish an authorized connection, and then investigate the data and operations available to your account.

An open listener is only the beginning of the assessment. The actual security impact depends on authentication, privileges, configuration, and the capabilities exposed by the database.
