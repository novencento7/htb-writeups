# Linux Remote Management Protocols

## 1. Introduction

Remote management protocols allow administrators and authorized users to access, configure, and troubleshoot systems over a network without being physically present at the target machine.

During penetration testing and Hack The Box (HTB) labs, identifying these services can reveal potential entry points, exposed management interfaces, weak authentication mechanisms, and misconfigurations.

Common remote management protocols include:

| Protocol | Default Port | Transport | Purpose |
|---|---:|---|---|
| SSH | 22 | TCP | Secure remote shell and administration |
| Telnet | 23 | TCP | Remote terminal access without encryption |
| RDP | 3389 | TCP/UDP | Remote graphical desktop access |
| VNC | 5900+ | TCP | Remote graphical desktop sharing |
| WinRM | 5985 / 5986 | TCP | Windows remote management over HTTP/HTTPS |
| IPMI | 623 | UDP | Out-of-band hardware and server management |

**Important:** A port being open does not automatically indicate a vulnerability. Enumerate the service, identify its configuration, and investigate potential weaknesses before drawing conclusions.

## 2. Initial Enumeration

### 2.1 Scan Common Management Ports

Use Nmap to identify exposed remote management services.

```bash
sudo nmap -sV -sC -p 22,23,3389,5900,5985,5986 <TARGET_IP>
```

To scan all TCP ports:

```bash
sudo nmap -sV -sC -p- --min-rate 1000 <TARGET_IP>
```

To check UDP port 623 for IPMI:

```bash
sudo nmap -sU -p 623 <TARGET_IP>
```

To identify an IPMI service:

```bash
sudo nmap -sU -sV -p 623 --script ipmi-version <TARGET_IP>
```

### 2.2 Identify the Service Version

Service version detection helps identify the software running on an exposed port.

```bash
sudo nmap -sV -p <PORT> <TARGET_IP>
```

Record the following information:

- Open ports and associated services.
- Software versions and possible vulnerabilities.
- Authentication requirements.
- Whether the connection is encrypted.
- Whether the service exposes administrative functionality.

## 3. SSH — Secure Shell

SSH provides encrypted remote terminal access and supports file transfers, port forwarding, and remote command execution.

**Default port:** TCP 22.

### Enumeration

```bash
nmap -sV -p 22 <TARGET_IP>
```

Connect to the remote SSH service:

```bash
ssh <USERNAME>@<TARGET_IP>
```

Specify a private key:

```bash
ssh -i <PRIVATE_KEY> <USERNAME>@<TARGET_IP>
```

Use verbose output to troubleshoot authentication and connection issues:

```bash
ssh -v <USERNAME>@<TARGET_IP>
```

Inspect supported authentication methods:

```bash
nmap -p 22 --script ssh-auth-methods --script-args="ssh.user=<USERNAME>" <TARGET_IP>
```

### Security Considerations

- Weak or reused passwords may expose accounts to password guessing.
- Exposed private keys can allow unauthorized access.
- Outdated SSH implementations may contain known vulnerabilities.
- Excessive permissions on SSH keys and configuration files can weaken security.

Use key-based authentication, restrict access, disable obsolete algorithms, and keep the SSH server updated.

## 4. Telnet

Telnet provides remote terminal access but does not encrypt the session by default. Credentials and transmitted data may therefore be exposed to network interception.

**Default port:** TCP 23.

### Enumeration

```bash
nmap -sV -p 23 <TARGET_IP>
```

Connect to a Telnet service:

```bash
telnet <TARGET_IP> 23
```

Alternatively, use Netcat to test whether the port accepts connections:

```bash
nc -nv <TARGET_IP> 23
```

### Security Considerations

Telnet should generally be replaced with SSH. If a legacy application requires Telnet, isolate it on a restricted network and avoid transmitting sensitive credentials.

## 5. RDP — Remote Desktop Protocol

RDP provides graphical remote access to Windows systems. An exposed RDP service may allow an authenticated user to interact with the target desktop.

**Default port:** TCP/UDP 3389.

### Enumeration

```bash
nmap -sV -p 3389 <TARGET_IP>
```

Check for supported RDP security configurations:

```bash
nmap -p 3389 --script rdp-enum-encryption <TARGET_IP>
```

Check whether Network Level Authentication (NLA) is required:

```bash
nmap -p 3389 --script rdp-ntlm-info <TARGET_IP>
```

The `rdp-ntlm-info` script may disclose Windows domain and system information when the service responds accordingly. It does not, by itself, establish whether NLA is required.

### Connecting to RDP

On Linux, FreeRDP can be used to connect to a remote Windows desktop:

```bash
xfreerdp /v:<TARGET_IP> /u:<USERNAME>
```

If credentials are available, supply the password interactively or through the client's supported credential mechanism. Avoid placing passwords directly in shell commands because they may be exposed in shell history or process information.

### Security Considerations

- Exposed RDP services can be targeted by password guessing.
- Weak credentials may lead to unauthorized desktop access.
- Outdated Windows systems may be vulnerable to known RDP-related flaws.
- Excessive access permissions can increase the impact of a compromised account.

Restrict RDP to trusted networks or VPN connections, enforce strong authentication, enable NLA where appropriate, and apply security updates.

## 6. VNC — Virtual Network Computing

VNC enables remote graphical desktop sharing and control. The standard port is TCP 5900, although additional display numbers may use subsequent ports.

**Default port:** TCP 5900 and related display ports.

### Enumeration

```bash
nmap -sV -p 5900-5910 <TARGET_IP>
```

Check for VNC authentication information:

```bash
nmap -p 5900 --script vnc-info <TARGET_IP>
```

Connect using a VNC client:

```bash
vncviewer <TARGET_IP>:5900
```

The exact connection syntax may vary between clients.

### Security Considerations

VNC deployments may expose graphical sessions with insufficient authentication or encryption. Restrict access, configure strong authentication, and use a secure tunnel or VPN when appropriate.

## 7. WinRM — Windows Remote Management

WinRM is Microsoft's remote management service for administering Windows systems. It supports remote PowerShell operations and management tasks.

**Default ports:**

- TCP 5985: HTTP.
- TCP 5986: HTTPS.

Although port 5985 uses HTTP, WinRM can protect management traffic through its configured authentication and message-encryption mechanisms. HTTPS provides TLS protection at the transport layer.

### Enumeration

```bash
nmap -sV -p 5985,5986 <TARGET_IP>
```

### Connecting with Evil-WinRM

Evil-WinRM is commonly used in authorized Windows penetration-testing labs.

```bash
evil-winrm -i <TARGET_IP> -u <USERNAME> -p '<PASSWORD>'
```

For authentication with a private key and certificate:

```bash
evil-winrm -i <TARGET_IP> -c <CERTIFICATE_FILE> -k <PRIVATE_KEY_FILE> -S
```

The certificate and key must be valid for the target's configured authentication requirements. The `-S` option enables SSL.

### Security Considerations

- WinRM access may provide remote PowerShell execution.
- Overly broad account permissions can increase the impact of compromise.
- Exposed services can reveal authentication and configuration weaknesses.
- Unrestricted network access to WinRM increases the attack surface.

Restrict access to trusted administrative hosts, use strong authentication, apply least-privilege permissions, and configure secure transport.

## 8. IPMI — Intelligent Platform Management Interface

IPMI enables out-of-band management of physical servers through a Baseboard Management Controller (BMC). Administrators can use it to monitor hardware, manage power states, and access remote console functionality.

Unlike SSH or Telnet, IPMI management is handled by a dedicated hardware controller. Depending on the implementation, the BMC may remain accessible even when the host operating system is unavailable or powered off.

**Default port:** UDP 623.

### Enumeration

Check whether the target exposes IPMI:

```bash
sudo nmap -sU -sV -p 623 --script ipmi-version <TARGET_IP>
```

Metasploit also provides an IPMI version scanner:

```text
use auxiliary/scanner/ipmi/ipmi_version
set RHOSTS <TARGET_IP>
run
```

### Security Considerations

Some older IPMI 2.0 implementations are affected by authentication weaknesses associated with the Remote Authenticated Key-Exchange Protocol (RAKP), including CVE-2013-4786.

Under vulnerable configurations, an attacker may be able to obtain challenge-response data that permits offline password verification. This is not equivalent to automatically obtaining a plaintext password or gaining access to every BMC.

Protect BMC interfaces through network isolation, strong unique credentials, firmware updates, and disabling unnecessary management services.

## 9. General Enumeration Workflow

A practical workflow for HTB labs is:

1. **Discover ports:** Identify exposed TCP and UDP services.
2. **Fingerprint services:** Determine protocols, versions, and relevant configurations.
3. **Investigate authentication:** Identify supported methods and required credentials.
4. **Review known weaknesses:** Check whether the detected implementation has relevant vulnerabilities.
5. **Validate access:** Use only authorized credentials and permitted testing techniques.
6. **Document findings:** Record commands, responses, evidence, and remediation recommendations.

Example initial scan:

```bash
sudo nmap -sC -sV -p 22,23,3389,5900,5985,5986 <TARGET_IP>
```

Example UDP scan for IPMI:

```bash
sudo nmap -sU -sV -p 623 --script ipmi-version <TARGET_IP>
```

## 10. Defensive Checklist

- [ ] Disable unused remote management services.
- [ ] Restrict administrative ports using firewall rules.
- [ ] Require strong, unique credentials and appropriate multi-factor authentication.
- [ ] Prefer encrypted protocols and secure configurations.
- [ ] Update operating systems, management software, and BMC firmware.
- [ ] Apply least-privilege access controls.
- [ ] Monitor authentication failures and suspicious management activity.
- [ ] Avoid exposing administrative interfaces directly to the public internet.
- [ ] Keep sensitive credentials and private keys out of command history and repositories.

## Conclusion

Remote management services are important targets during network enumeration because they can provide direct access to operating systems, graphical desktops, administrative interfaces, or hardware controllers.

The objective of enumeration is to determine which services are available, understand their configuration, and identify weaknesses that can be validated safely in an authorized environment.

Always distinguish between an exposed service, a potential vulnerability, and a confirmed security issue.
