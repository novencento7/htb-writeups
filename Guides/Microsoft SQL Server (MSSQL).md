# Microsoft SQL Server (MSSQL)

## Overview

Microsoft SQL Server (`MSSQL`) is a relational database management system developed by Microsoft. It uses SQL and Microsoft's Transact-SQL (`T-SQL`) extension to manage and query relational databases.

SQL Server was originally designed for Windows, but modern versions also support Linux and containerized environments. It is widely used in enterprise applications, particularly those built with the Microsoft .NET ecosystem.

In a typical client-server architecture, applications and database clients send SQL queries to the SQL Server Database Engine, which processes the requests and returns the results.

### Client-Server Architecture

```text
+-------------------+
| Application       |
| or Database Client|
+-------------------+
          |
          | TDS over TCP
          v
+-------------------+
| Microsoft SQL     |
| Server            |
| Database Engine   |
+-------------------+
          |
          v
+-------------------+
| Databases         |
| Tables and Views  |
+-------------------+
```

The application or client does not normally communicate directly with database files. The SQL Server Database Engine manages storage, queries, transactions, permissions, and concurrent access.

## 1. MSSQL Clients

Several clients can connect to an MSSQL server.

| Client | Description |
|---|---|
| SQL Server Management Studio (SSMS) | Graphical administration tool for Windows |
| `sqlcmd` | Command-line SQL query utility |
| `mssql-cli` | Interactive command-line client |
| HeidiSQL | Graphical database management client |
| SQLPro | Database client for macOS |
| `impacket-mssqlclient` | Python client useful for authorized security assessments |

SSMS is a client application. It can run on a workstation separate from the SQL Server host.

This distinction is important when investigating a Windows environment: a workstation may contain database clients, saved connection information, or configuration files even when the database server itself is hosted elsewhere.

Official resources:

- [SQL Server](https://www.microsoft.com/en-us/sql-server)
- [SQL Server Management Studio](https://learn.microsoft.com/en-us/ssms/)
- [sqlcmd documentation](https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-utility)
- [Impacket](https://github.com/fortra/impacket)

## 2. Default System Databases

SQL Server creates several system databases during installation.

| Database | Purpose |
|---|---|
| `master` | Stores instance-level configuration and system metadata |
| `model` | Template used when creating new databases |
| `msdb` | Stores information used by SQL Server Agent, jobs, alerts, and related features |
| `tempdb` | Stores temporary objects and intermediate data |
| `Resource` | Contains system objects used by SQL Server |

The `Resource` database differs from the other system databases because it is a hidden, read-only system database.

To list databases after connecting to SQL Server, execute:

```sql
SELECT name
FROM sys.databases
ORDER BY name;
```

To select a particular database:

```sql
USE master;
GO
```

Replace `master` with the database you want to access.

## 3. Installing Microsoft SQL Server

SQL Server is available for supported Windows and Linux environments. Installation procedures depend on the operating system and SQL Server version.

### 3.1 Windows Server

On Windows, download SQL Server from the official Microsoft website:

[Download SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)

General installation procedure:

1. Download and launch the SQL Server installer.
2. Select the installation type.
3. Choose the database engine features required by your environment.
4. Configure the database instance.
5. Select the authentication mode.
6. Configure the SQL Server service account.
7. Complete the installation.
8. Install SQL Server Management Studio separately if required.
9. Configure network connectivity and firewall rules if remote access is needed.

The default instance is commonly identified as `MSSQLSERVER`.

The default TCP port is `1433`, although named instances and custom configurations may use other ports.

### 3.2 Linux

Microsoft provides SQL Server packages for supported Linux distributions, including Ubuntu.

Always follow the official installation instructions for the specific distribution and SQL Server version:

[Install SQL Server on Linux](https://learn.microsoft.com/en-us/sql/linux/quickstart-install-connect-ubuntu)

The general process is:

1. Configure Microsoft's package repository.
2. Install the SQL Server package.
3. Run the initial setup utility.
4. Select an edition and configure the system administrator password.
5. Start and enable the SQL Server service.
6. Install a client utility such as `sqlcmd`.
7. Configure network access only when necessary.

On supported systemd-based installations, service management typically uses:

```bash
sudo systemctl status mssql-server
sudo systemctl start mssql-server
sudo systemctl enable mssql-server
```

Check the service logs if the server fails to start:

```bash
sudo journalctl -u mssql-server --no-pager -n 100
```

**Important:** Package names, repository configuration, and client installation commands vary by release. Use the official Microsoft instructions rather than assuming that commands for one Ubuntu release work on another.

## 4. Configuring the Server

### 4.1 Authentication

SQL Server supports two principal authentication approaches.

**Windows Authentication**

The operating system or Active Directory validates the user's identity. SQL Server then determines which permissions that identity has within the database instance.

This approach is common in Windows domain environments.

**SQL Server Authentication**

SQL Server validates a SQL login and password directly.

The built-in `sa` login is a well-known SQL Server administrator account. Its availability and permissions depend on the installation and configuration.

Best practices include:

- Prefer Windows or integrated authentication where appropriate.
- Use dedicated accounts with the minimum required privileges.
- Disable unused accounts.
- Avoid weak or default passwords.
- Store credentials in an approved secrets-management system.
- Review administrator privileges regularly.

### 4.2 Network Configuration

SQL Server commonly accepts TCP connections on port `1433`.

Verify the listening port and network configuration using SQL Server Configuration Manager or the appropriate operating-system tools.

On Linux:

```bash
sudo ss -lntp
```

To inspect the status of the host firewall on Ubuntu systems using UFW:

```bash
sudo ufw status verbose
```

If remote access is required, allow connections only from trusted application servers or administration hosts.

Do not expose the database directly to the public internet unless there is a specific, carefully secured requirement.

### 4.3 Encryption

SQL Server supports encrypted client-server connections.

Check that:

- TLS encryption is configured correctly.
- Clients validate the server certificate.
- Certificates are trusted and renewed before expiration.
- Authentication credentials are not transmitted over untrusted connections without appropriate protection.

A self-signed certificate may be suitable for some controlled environments, but clients must not blindly trust arbitrary certificates.

## 5. Installing and Using a Client

### 5.1 Using `sqlcmd`

`sqlcmd` is Microsoft's command-line utility for executing SQL statements against SQL Server.

Install it by following the official instructions for your operating system:

[Install sqlcmd](https://learn.microsoft.com/en-us/sql/tools/sqlcmd/sqlcmd-download-install)

A typical SQL login connection looks like this:

```bash
sqlcmd -S 192.0.2.10,1433 -U dbadmin -d master
```

Here:

- `-S` specifies the server and port.
- `-U` specifies the SQL login.
- `-d` specifies the initial database.

Use an approved credential-prompt mechanism for the installed client version rather than placing passwords directly in shell commands.

For Windows Authentication, supported clients provide integrated-authentication options. The exact syntax depends on the version of `sqlcmd` and the operating system.

Once connected, execute:

```sql
SELECT @@VERSION;
GO
```

List the available databases:

```sql
SELECT name
FROM sys.databases;
GO
```

List tables in the current database:

```sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE';
GO
```

Exit the interactive client with:

```text
EXIT
```

### 5.2 Connecting with Impacket

Impacket provides `mssqlclient.py`, a Python client frequently used in authorized penetration tests and security laboratories.

On Kali Linux or another distribution with Impacket installed, locate the executable:

```bash
command -v impacket-mssqlclient
```

Alternatively:

```bash
find /usr/share/doc/python3-impacket -name 'mssqlclient.py' 2>/dev/null
```

For a SQL Server login:

```bash
impacket-mssqlclient dbadmin@192.0.2.10
```

For Windows or domain authentication:

```bash
impacket-mssqlclient DOMAIN/user@192.0.2.10 -windows-auth
```

The client will prompt for the password when appropriate.

After authentication, inspect the available client commands:

```text
help
```

List the databases:

```sql
SELECT name FROM sys.databases;
```

Depending on the client version, additional commands may be available for switching databases or inspecting server configuration.

Use only credentials and systems for which you have explicit authorization.

## 6. Basic SQL Operations

Once connected, SQL queries can be used to inspect and manage databases.

### 6.1 List Databases

```sql
SELECT name
FROM sys.databases
ORDER BY name;
```

### 6.2 Select a Database

```sql
USE master;
GO
```

### 6.3 List Tables

```sql
SELECT TABLE_SCHEMA, TABLE_NAME
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_TYPE = 'BASE TABLE';
```

### 6.4 Inspect a Table

```sql
SELECT TOP (10) *
FROM dbo.Employees;
```

Replace `dbo.Employees` with the actual schema and table name.

### 6.5 Inspect the Current Login

```sql
SELECT
    SYSTEM_USER AS system_login,
    SUSER_SNAME() AS login_name,
    USER_NAME() AS database_user;
```

### 6.6 Check Server Version

```sql
SELECT @@VERSION;
```

These queries are useful for routine administration, troubleshooting, and authorized service enumeration.

## 7. Footprinting MSSQL

Footprinting is the process of identifying a service, its version, network configuration, and potentially relevant security weaknesses.

During a Hack The Box (HTB) lab or an authorized assessment, begin by checking whether TCP port `1433` is open.

### 7.1 Basic Nmap Scan

```bash
nmap -sV -p 1433 192.0.2.10
```

Options:

- `-sV`: attempts to identify the service and version.
- `-p 1433`: scans the conventional MSSQL TCP port.

If the service uses a custom port, scan the discovered port instead.

### 7.2 MSSQL Enumeration Scripts

Nmap provides scripts for gathering additional information about SQL Server.

```bash
nmap -sV -p 1433 \
  --script ms-sql-info \
  192.0.2.10
```

To enumerate additional information where authorized:

```bash
nmap -sV -p 1433 \
  --script ms-sql-info,ms-sql-ntlm-info \
  192.0.2.10
```

Depending on the target configuration and available responses, these scripts may reveal:

- Server and computer names.
- SQL Server instance names.
- Product and version information.
- Domain-related information.
- Network configuration details.

The absence of results does not necessarily mean that SQL Server is not running. Firewalls, authentication requirements, and service configuration can limit the information returned.

### 7.3 Additional Nmap Scripts

Nmap includes other MSSQL scripts for checking configuration, database access, tables, and certain authentication conditions.

Discover the scripts available in the installed version:

```bash
ls /usr/share/nmap/scripts/ms-sql-*.nse
```

Read individual script documentation before using it:

```bash
nmap --script-help ms-sql-info
```

Use intrusive or potentially disruptive checks only when they are within the authorized assessment scope.

### 7.4 Metasploit MSSQL Ping

Metasploit includes the `auxiliary/scanner/mssql/mssql_ping` module.

Launch Metasploit:

```bash
msfconsole
```

Select the module:

```text
use auxiliary/scanner/mssql/mssql_ping
```

Set the target:

```text
set RHOSTS 192.0.2.10
```

Run the scan:

```text
run
```

Depending on the server response, the module may reveal the server name, instance name, version, TCP port, and named-pipe information.

## 8. Connecting After Enumeration

If valid credentials are available, connect using an appropriate client.

For example:

```bash
impacket-mssqlclient DOMAIN/user@192.0.2.10 -windows-auth
```

Once connected, begin with low-impact queries:

```sql
SELECT @@VERSION;
SELECT SYSTEM_USER;
SELECT name FROM sys.databases;
```

Then inspect the databases and objects that the authenticated account is authorized to access.

Authentication alone does not imply administrative access. The operations available depend on the login's server-level permissions, database roles, and applicable security policies.

## 9. Common Misconfigurations

MSSQL security assessments should consider the following potential issues.

### Weak or Reused Credentials

Weak SQL passwords, reused credentials, and poorly protected administrator accounts can permit unauthorized access.

### Excessive Privileges

Accounts with unnecessary server-level administrative privileges can create opportunities for privilege escalation.

Review server roles and database permissions regularly.

### Unnecessary Network Exposure

A database server that accepts connections from untrusted networks has a larger attack surface.

Restrict access using network segmentation, firewall rules, and private connectivity.

### Insufficient Encryption Validation

Encrypted connections are not automatically secure if clients fail to validate certificates or administrators use inappropriate trust settings.

### Dangerous Extended Procedures

SQL Server includes extended procedures that can provide functionality beyond ordinary SQL queries. Some capabilities may interact with the operating system when enabled and permitted.

For example, `xp_cmdshell` can execute operating-system commands in supported configurations. It is disabled by default on new installations and should remain disabled unless a documented requirement justifies its use.

Enabling such functionality unnecessarily can significantly increase risk.

### Named Pipes

SQL Server can support named-pipe connectivity in addition to TCP/IP.

Review whether named pipes are required and ensure that their access controls and network exposure are appropriate.

### Unpatched Software

Maintain a supported SQL Server version and apply security updates in accordance with the organization's patch-management policy.

## 10. Troubleshooting

### Connection Refused or Timeout

Check that:

- The SQL Server service is running.
- The hostname and port are correct.
- TCP/IP is enabled where required.
- The server is listening on the expected interface.
- Firewalls allow traffic from the client.
- The SQL Server instance uses the expected port.

### Login Failed

Check that:

- The username and password are correct.
- The login is enabled.
- The authentication mode supports the chosen login.
- Windows or domain authentication is configured correctly.
- The account is permitted to connect to the instance.

### Database Access Denied

A successful login does not guarantee access to every database. Check the user's database mapping, roles, and permissions.

### TLS or Certificate Errors

Check the client and server encryption configuration, certificate validity, certificate trust chain, and hostname matching.

Do not resolve certificate problems by disabling validation in production.

### Service Not Running on

---

## Interpreting MSSQL Nmap Script Scan Results

During a penetration test, running Nmap's MSSQL NSE scripts can help identify the SQL Server version, discover the underlying Windows host, test authentication, and determine which database operations may be accessible.

The following scan was performed against an authorized Hack The Box lab target.

### MSSQL Enumeration Command

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes \
--script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER \
-sV -p 1433 10.129.244.239
```

#### Understanding the Command

- `sudo nmap`: runs Nmap with elevated privileges.
- `--script`: specifies the NSE scripts to execute.
- `--script-args`: supplies arguments used by the MSSQL scripts.
- `mssql.instance-port=1433`: specifies the SQL Server TCP port.
- `mssql.username=sa`: specifies the SQL Server login to test.
- `mssql.password=`: supplies an empty password.
- `mssql.instance-name=MSSQLSERVER`: specifies the default instance name.
- `-sV`: attempts to identify the service and its version.
- `-p 1433`: scans TCP port 1433 on the target.

**Important:** This command tests the `sa` account with an empty password. It does not establish that the account is disabled, that every authentication method fails, or that the server is otherwise inaccessible.

### 1. Identify the SQL Server Version

The scan returned:

```text
PORT     STATE SERVICE  VERSION
1433/tcp open  ms-sql-s Microsoft SQL Server 2019 15.00.2000.00; RTM
```

This confirms that TCP port 1433 is open and the service has been identified as Microsoft SQL Server 2019.

The version number `15.00.2000.00` corresponds to the SQL Server 2019 RTM release.

This information is useful for documenting the target and investigating whether applicable security updates are missing. The version alone does not prove that the server is vulnerable.

### 2. Identify the Underlying Windows Host

The `ms-sql-ntlm-info` script returned:

```text
Target_Name: ILF-SQL-01
NetBIOS_Domain_Name: ILF-SQL-01
NetBIOS_Computer_Name: ILF-SQL-01
DNS_Domain_Name: ILF-SQL-01
DNS_Computer_Name: ILF-SQL-01
Product_Version: 10.0.17763
```

The results reveal the hostname `ILF-SQL-01`.

The Windows build number `10.0.17763` is associated with the Windows Server 2019 generation.

During an assessment, record this information because hostnames can help correlate findings from other services, such as SMB, LDAP, DNS, and RDP.

The returned domain-related fields do not necessarily prove that the host belongs to an Active Directory domain. Further enumeration is needed to establish the actual environment.

### 3. Understand the MSSQL Instance Information

The `ms-sql-info` script returned:

```text
Instance name: MSSQLSERVER
Version:
  name: Microsoft SQL Server 2019 RTM
  number: 15.00.2000.00
TCP port: 1433
Named pipe: \\10.129.244.239\pipe\sql\query
Clustered: false
```

The relevant fields are:

- **Instance name:** `MSSQLSERVER`, the conventional name of the default SQL Server instance.
- **TCP port:** `1433`, the port used by the instance.
- **Named pipe:** an alternative communication mechanism, if enabled and accessible.
- **Clustered:** `false`, indicating that the instance was not identified as clustered by this scan.

The named-pipe information does not, by itself, prove that remote named-pipe connections are accessible.

### 4. Check for an Empty Password

The scan included the `ms-sql-empty-password` script and supplied the `sa` username with an empty password.

The relevant output was:

```text
ms-sql-empty-password:
|_  10.129.244.239\MSSQLSERVER:
```

The output does not explicitly report a successful login. Therefore, it should not be interpreted as definitive proof that the empty password worked.

To verify the result, attempt a connection using an appropriate MSSQL client and inspect the authentication response.

### 5. Understand Authentication Errors

The `ms-sql-config` script returned:

```text
ERROR: Bad username or password
```

The `ms-sql-dump-hashes` script returned a similar error.

These messages indicate that the scripts could not authenticate using the supplied credentials.

They do not establish that all accounts are inaccessible or that the server cannot be reached using Windows Authentication or another valid SQL login.

SQL Server supports different authentication mechanisms, including:

- **Windows Authentication:** authentication through Windows security.
- **SQL Server Authentication:** authentication using SQL Server-managed logins, such as `sa`.

The appropriate authentication method depends on the target's configuration and the credentials available.

### 6. Understand the `xp_cmdshell` Script

The `ms-sql-xp-cmdshell` script returned:

```text
(Use --script-args=ms-sql-xp-cmdshell.cmd='<CMD>' to change command.)
```

This message describes an available script argument. It does not confirm that `xp_cmdshell` is enabled or that operating-system commands can be executed.

`xp_cmdshell` is an extended stored procedure that can execute operating-system commands through SQL Server when enabled and when the login has sufficient permissions.

Its availability and permissions are important security considerations because excessive privileges may allow database access to become a route toward the underlying operating system.

### 7. Understand the DAC Result

The `ms-sql-dac` script returned:

```text
port: 1434
state: closed
error: ERROR
```

The Dedicated Administrator Connection (DAC) provides a special administrative connection for troubleshooting SQL Server.

The reported failure means the script could not establish the attempted DAC connection. It does not establish that every administrative connection is unavailable.

Do not confuse the DAC with the SQL Server Browser service, which commonly uses UDP port 1434 to provide information about SQL Server instances.

### 8. Understand Table Enumeration Errors

The `ms-sql-tables` script returned:

```text
ERROR: Script execution failed (use -d to debug)
```

The `ms-sql-hasdbaccess` script also reported:

```text
Bug in ms-sql-hasdbaccess: no string output.
```

These messages do not conclusively establish whether the login has database access.

Possible explanations include authentication problems, insufficient permissions, script limitations, or execution errors. Additional testing with a valid SQL connection is required to determine the actual cause.

### 9. Connect to MSSQL Using Impacket

If valid credentials are available, Impacket's `mssqlclient.py` can be used to connect to the SQL Server instance.

For SQL Server Authentication:

```bash
impacket-mssqlclient sa@10.129.244.239
```

Enter the password when prompted. To test an empty password, submit an empty response.

For Windows Authentication, use the appropriate Windows account and the `-windows-auth` option:

```bash
impacket-mssqlclient DOMAIN/username@10.129.244.239 -windows-auth
```

Replace the example domain and username with authorized credentials.

If authentication succeeds, the client provides an interactive SQL prompt. You can begin by enumerating the databases available to the current login.

```sql
SELECT name FROM sys.databases;
```

The returned databases depend on the login's permissions and SQL Server's metadata visibility rules.

### 10. Troubleshooting Checklist

When an MSSQL enumeration scan produces errors, use the following checklist:

1. **Verify connectivity:** confirm that TCP port 1433 is reachable.
2. **Verify the credentials:** distinguish a failed login from a network or service error.
3. **Check the authentication method:** determine whether the target expects SQL Server Authentication or Windows Authentication.
4. **Check permissions:** successful authentication does not imply administrative privileges.
5. **Inspect individual NSE scripts:** run selected scripts separately to isolate errors.
6. **Use client-side verification:** confirm important findings with an MSSQL client rather than relying on ambiguous script output.
7. **Record the evidence:** document the hostname, SQL Server version, instance name, open port, authentication results, and any limitations.

### Key Takeaways

| Finding | Interpretation |
|---|---|
| `1433/tcp open` | SQL Server is reachable on the scanned TCP port. |
| `Microsoft SQL Server 2019 RTM` | The service identifies itself as SQL Server 2019 RTM. |
| `ILF-SQL-01` | Hostname information was disclosed through NTLM-related enumeration. |
| `Bad username or password` | The supplied credentials did not authenticate in that script's attempt. |
| `xp_cmdshell` usage message | Not proof that command execution is enabled. |
| `ms-sql-tables` execution failure | The table enumeration attempt failed; the underlying cause remains unconfirmed. |

The most important principle is to distinguish **service discovery, authentication, authorization, and command execution**. An open SQL Server port proves network reachability, not database access or administrative control.

