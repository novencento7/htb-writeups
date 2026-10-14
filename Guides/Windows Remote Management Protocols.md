# Windows Remote Management Protocols

---

## Overview

Windows systems can be managed remotely using several protocols and services. These mechanisms allow administrators to execute commands, access graphical interfaces, retrieve system information, and manage remote computers.

The three main Windows remote management protocols are:

- **Remote Desktop Protocol (RDP)** — Provides remote access to the Windows graphical desktop.
- **Windows Remote Management (WinRM)** — Enables remote command execution and administration through Windows management interfaces.
- **Windows Management Instrumentation (WMI)** — Provides access to system configuration, hardware information, and administrative functions.

Understanding these services is essential during penetration testing because they can reveal valuable information about a target and provide remote access when valid credentials are available.

## 1. Remote Desktop Protocol (RDP)

### Overview

Remote Desktop Protocol (RDP) is a Microsoft protocol that allows users to connect to and control a remote Windows computer through a graphical interface.

RDP operates at the application layer and typically uses:

| Port | Protocol | Description |
|---|---|---|
| 3389/TCP | TCP | Standard RDP communication |
| 3389/UDP | UDP | Supports additional RDP transport functionality |

For RDP to work, the service must be enabled and the relevant network and host firewall rules must allow connections. When a server is behind a NAT device, appropriate port forwarding may also be necessary.

RDP supports encryption through TLS and Network Level Authentication (NLA). NLA requires authentication before a complete remote desktop session is established, reducing exposure to unauthenticated connections.

However, certificate validation remains important. A self-signed certificate or a certificate whose hostname does not match the target address can trigger a warning. Such warnings should not be ignored without verifying the server's identity.

### 1.1 Footprinting RDP

Nmap can identify the RDP service, enumerate its security configuration, and retrieve information about the remote Windows host.

**Command:**

```bash
nmap -sV -sC -p3389 --script rdp* 10.129.201.248
```

Useful information may include:

- Whether Network Level Authentication (NLA) is supported.
- The supported RDP security mechanisms.
- The Windows product version.
- The target hostname and domain information.
- The system time reported by the server.

Example output:

```text
PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-enum-encryption:
|   Security layer
|     CredSSP (NLA): SUCCESS
|     CredSSP with Early User Auth: SUCCESS
|_    RDSTLS: SUCCESS
| rdp-ntlm-info:
|   Target_Name: ILF-SQL-01
|   NetBIOS_Domain_Name: ILF-SQL-01
|   NetBIOS_Computer_Name: ILF-SQL-01
|   DNS_Computer_Name: ILF-SQL-01
|   Product_Version: 10.0.17763
```

The reported version `10.0.17763` corresponds to the Windows 10 / Windows Server 2019 generation. Additional evidence is needed to distinguish the exact operating system edition.

### 1.2 Inspecting RDP Network Traffic

The `--packet-trace` option displays packets sent and received by Nmap, making it useful for understanding how the service responds during enumeration.

**Command:**

```bash
nmap -sV -sC -p3389 --packet-trace --disable-arp-ping -n 10.129.201.248
```

Nmap may send an RDP cookie containing a value such as:

```text
Cookie: mstshash=nmap
```

This value can be visible to network monitoring and security tools. Network defenders may use protocol fingerprints and connection patterns to identify scanning activity.

### 1.3 Checking RDP Security Configuration

The `rdp-sec-check` script can test the security protocols and encryption mechanisms supported by an RDP server.

Install the Perl dependency:

```bash
sudo cpan install Encoding::BER
```

Clone the tool:

```bash
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git && cd rdp-sec-check
```

Run the check:

```bash
./rdp-sec-check.pl 10.129.201.248
```

The output can indicate whether the server supports:

- Standard RDP security.
- TLS security.
- CredSSP-based security using NLA.
- Various legacy RDP encryption modes.

For example, a server that requires the hybrid CredSSP protocol and rejects standard RDP security is enforcing a more restrictive authentication configuration.

**Note:** The tool is an older project. Verify its compatibility with the installed Perl environment and interpret its results in the context of the server's current configuration.

### 1.4 Connecting to an RDP Server

Linux systems can connect to RDP servers using clients such as FreeRDP, Remmina, or rdesktop.

Example using FreeRDP:

```bash
xfreerdp /v:10.129.201.248 /u:cry0l1t3
```

FreeRDP prompts for a password when one has not been supplied through the command line.

A connection can also produce a certificate warning if the certificate is self-signed or its name does not match the hostname used for the connection.

Before trusting a certificate, verify that it belongs to the intended server. In particular, avoid placing real passwords directly in shell commands because they can be exposed through shell history or process information.

Once authentication succeeds, the client opens a graphical session on the remote Windows machine.

---

## 2. Windows Remote Management (WinRM)

### Overview

Windows Remote Management (WinRM) is Microsoft's implementation of the WS-Management protocol. It allows administrators to manage remote Windows systems, execute commands, and interact with management services.

WinRM uses SOAP-based messages and commonly operates over the following ports:

| Port | Protocol | Description |
|---|---|---|
| 5985/TCP | HTTP | Default WinRM HTTP listener |
| 5986/TCP | HTTPS | Default WinRM HTTPS listener |

WinRM is integrated into Windows, but its availability depends on the operating system, configuration, listener setup, and firewall rules.

Windows Remote Shell (WinRS) and PowerShell remoting can use WinRM to execute commands on remote systems.

### 2.1 Footprinting WinRM

Nmap can identify WinRM listeners and retrieve basic service information.

**Command:**

```bash
nmap -sV -sC -p5985,5986 --disable-arp-ping -n 10.129.201.248
```

Example output:

```text
PORT     STATE SERVICE VERSION
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
```

An open port 5985 suggests that an HTTP-based WinRM listener may be available. However, an open port alone does not prove that authentication will succeed or that the endpoint is fully configured for remote management.

### 2.2 Checking WinRM from Windows

PowerShell provides the `Test-WSMan` cmdlet to test whether a remote computer responds to WS-Management requests.

**Command:**

```powershell
Test-WSMan -ComputerName 10.129.201.248
```

A successful response indicates that the remote system is responding to a WS-Management request. It does not necessarily mean that the current user has permission to execute remote commands.

### 2.3 Connecting to WinRM from Linux

`evil-winrm` is a tool commonly used in authorized penetration tests to interact with Windows hosts through WinRM.

Install it on a compatible Linux environment:

```bash
sudo gem install evil-winrm
```

Connect using a valid account:

```bash
evil-winrm -i 10.129.201.248 -u 'Cry0l1t3' -p 'PASSWORD'
```

If authentication succeeds and the account has the necessary permissions, an interactive PowerShell session may be opened.

Example prompt:

```text
*Evil-WinRM* PS C:\Users\Cry0l1t3\Documents>
```

At this point, commands can be executed remotely within the permissions granted to the authenticated account.

**Security note:** Use HTTPS where appropriate, restrict access to trusted management networks, enforce strong authentication, and avoid exposing WinRM directly to untrusted networks.

---

## 3. Windows Management Instrumentation (WMI)

### Overview

Windows Management Instrumentation (WMI) is Microsoft's implementation of a management framework based on the Common Information Model (CIM).

WMI enables administrators and applications to retrieve system information and manage Windows configuration, services, processes, hardware, and other operating system components.

WMI can be accessed through several interfaces, including:

- PowerShell.
- VBScript.
- Windows Management Instrumentation Command-line (WMIC).
- APIs and third-party administration tools.

WMI is not a single executable. It consists of management infrastructure, providers, interfaces, and repositories.

### 3.1 WMI Network Communication

Remote WMI commonly relies on Microsoft RPC/DCOM.

| Port | Protocol | Description |
|---|---|---|
| 135/TCP | RPC Endpoint Mapper | Initial RPC connection |
| Dynamic TCP ports | RPC | Subsequent WMI/DCOM communication |

The client initially contacts TCP port 135. RPC endpoint negotiation then identifies the port used for subsequent communication.

Consequently, checking port 135 alone is not sufficient to establish whether remote WMI operations will succeed. Firewall configuration, RPC availability, authentication, and user permissions also matter.

### 3.2 Executing Remote Commands with Impacket

The Impacket toolkit includes `wmiexec.py`, which can execute commands on a remote Windows system through WMI.

Example using a lab account:

```bash
impacket-wmiexec 'DOMAIN/USERNAME:PASSWORD@10.129.201.248' 'hostname'
```

For a local account, use the appropriate local-account syntax supported by the installed Impacket version.

Some distributions provide the script under a path such as:

```bash
/usr/share/doc/python3-impacket/examples/wmiexec.py
```

Example output:

```text
Impacket v0.9.22

[*] SMBv3.0 dialect used
ILF-SQL-01
```

The output indicates that the remote command returned the hostname `ILF-SQL-01`.

The exact executable name and installation path may differ depending on the distribution and Impacket version.

### 3.3 Requirements for Remote WMI

Successful remote WMI execution generally depends on:

- Valid credentials.
- Sufficient permissions on the target system.
- Available RPC/DCOM connectivity.
- Appropriate Windows firewall rules.
- Compatible remote management configuration.

WMI can be a useful administration mechanism, but it is also security-sensitive because authorized operations can expose extensive system information or allow remote command execution.

---

## 4. Comparing Windows Remote Management Protocols

| Feature | RDP | WinRM | WMI |
|---|---|---|---|
| Primary purpose | Graphical remote access | Remote administration and command execution | System management and instrumentation |
| Typical ports | 3389/TCP, 3389/UDP | 5985/TCP, 5986/TCP | 135/TCP and dynamic RPC ports |
| Interface | Graphical desktop | Command line / PowerShell | Management APIs and command-line tools |
| Common Linux tools | FreeRDP, Remmina | Evil-WinRM | Impacket `wmiexec.py` |
| Main requirement | Valid account and RDP access | Valid account and WinRM permissions | Valid account, RPC access, and WMI permissions |
| Common enumeration goal | Identify NLA, host information, and security settings | Identify listeners and test WS-Management access | Identify RPC exposure and assess remote management access |

---

## 5. Practical Enumeration Workflow

When assessing a Windows host in an authorized lab, the following workflow can help identify available remote management services.

### Step 1: Scan the relevant ports

```bash
nmap -sV -sC -p135,3389,5985,5986 10.129.201.248
```

### Step 2: Enumerate RDP

```bash
nmap -sV -p3389 --script rdp-enum-encryption,rdp-ntlm-info 10.129.201.248
```

Review the reported security protocols, NLA support, hostname, and version information.

### Step 3: Check WinRM

```bash
nmap -sV -sC -p5985,5986 10.129.201.248
```

If the listener is available, verify WS-Management access and determine whether the supplied account is authorized for remote administration.

### Step 4: Assess RPC and WMI exposure

```bash
nmap -sV -sC -p135 10.129.201.248
```

Remember that WMI/DCOM may require additional dynamic RPC ports. A scan of port 135 alone cannot confirm that remote WMI execution is possible.

### Step 5: Test authorized access

Use the appropriate client for each protocol, with credentials provided securely. Record which services are reachable, which authentication methods are supported, and what permissions the test account has.

---

## 6. Security Considerations

Windows remote management services are essential for administration, but improper configuration can increase attack surface.

Recommended defensive practices include:

- Restrict RDP, WinRM, and RPC access to trusted hosts or management networks.
- Require strong authentication and least-privilege permissions.
- Enable Network Level Authentication for RDP where appropriate.
- Configure trusted TLS certificates and validate server identities.
- Prefer HTTPS for WinRM when suitable for the environment.
- Limit dynamic RPC exposure through appropriate network and host firewall rules.
- Monitor authentication attempts, remote command execution, and unusual connection patterns.
- Disable unnecessary remote management services.
- Keep Windows systems and management tools updated.

## Conclusion

RDP, WinRM, and WMI provide different mechanisms for remotely administering Windows systems.

- **RDP** is primarily used for interactive graphical access.
- **WinRM** provides a foundation for remote command execution and PowerShell administration.
- **WMI** exposes extensive system management functionality through Windows management interfaces.

During penetration testing, identifying these services helps map the target's remote administration surface. Open ports and successful service detection do not, by themselves, establish that authentication or command execution is possible. Always distinguish service discovery from verified access and assess the permissions available to the account being tested.
