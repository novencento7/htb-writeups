# SNMP Server and Client — Local Lab

## Introduction

**SNMP (Simple Network Management Protocol)** is a protocol used to monitor and manage network devices and systems.

SNMP can be used to retrieve information such as:

* System information
* Network interfaces
* CPU and memory information
* Running processes
* Network statistics
* Device configuration information

SNMP normally uses:

* **UDP 161** → SNMP requests and responses
* **UDP 162** → SNMP traps

In this guide, we will create a simple **SNMP server and client on the same Linux machine**.

```text
SNMP Client
     |
     | UDP 161
     v
SNMP Server (snmpd)
```

This is useful for learning and testing SNMP before working with a real network.

---

# 1. Install SNMP

On Debian-based distributions:

```bash
sudo apt update
sudo apt install snmp snmpd
```

Two important packages are installed:

* `snmp` → SNMP client tools
* `snmpd` → SNMP server/agent

The `snmpd` service acts as the SNMP agent and waits for requests from SNMP clients.

---

# 2. Check the SNMP service

Check the status of `snmpd`:

```bash
sudo systemctl status snmpd
```

If the service is not running:

```bash
sudo systemctl start snmpd
```

To start it automatically at boot:

```bash
sudo systemctl enable snmpd
```

Or enable and start it in one command:

```bash
sudo systemctl enable --now snmpd
```

Check again:

```bash
sudo systemctl status snmpd
```

---

# 3. Check the SNMP port

SNMP normally listens on **UDP port 161**.

Use:

```bash
sudo ss -lunp | grep 161
```

You should see something similar to:

```text
UNCONN 0 0 127.0.0.1:161 0.0.0.0:* users:(("snmpd",...))
```

This confirms that `snmpd` is listening for SNMP requests.

---

# 4. SNMP configuration

The main server configuration file is:

```text
/etc/snmp/snmpd.conf
```

Open it with:

```bash
sudo nano /etc/snmp/snmpd.conf
```

A default configuration may contain:

```text
sysLocation    Sitting on the Dock of the Bay
sysContact     Me <me@example.org>
sysServices    72

master  agentx

agentaddress  127.0.0.1,[::1]

view   systemonly  included   .1.3.6.1.2.1.1
view   systemonly  included   .1.3.6.1.2.1.25.1

rocommunity  public default -V systemonly
rocommunity6 public default -V systemonly

rouser authPrivUser authpriv -V systemonly

includeDir /etc/snmp/snmpd.conf.d
```

---

# 5. Understanding `agentaddress`

This line:

```text
agentaddress 127.0.0.1,[::1]
```

controls where the SNMP agent listens.

In this configuration:

```text
127.0.0.1
```

means IPv4 localhost.

```text
[::1]
```

means IPv6 localhost.

Therefore, the SNMP server can only be accessed locally.

This is perfect for our test because both the client and server are running on the same machine.

---

# 6. SNMP community string

SNMPv2c uses a **community string** for access control.

In the configuration:

```text
rocommunity public default -V systemonly
```

the community string is:

```text
public
```

`rocommunity` means **read-only community**.

Therefore, a client can retrieve information but cannot modify SNMP values.

The community string is similar to a password, but SNMPv2c does not encrypt it.

For this reason, using:

```text
public
```

on an exposed production system is insecure.

---

# 7. SNMP views

The configuration contains:

```text
view   systemonly  included   .1.3.6.1.2.1.1
view   systemonly  included   .1.3.6.1.2.1.25.1
```

A **view** determines which parts of the SNMP information tree the client is allowed to access.

The OID:

```text
.1.3.6.1.2.1.1
```

contains general system information.

For example:

* System description
* System uptime
* System name
* System location
* System contact

The second OID:

```text
.1.3.6.1.2.1.25.1
```

contains information related to the host system.

---

# 8. Restart SNMP after configuration changes

After modifying `/etc/snmp/snmpd.conf`, restart the service:

```bash
sudo systemctl restart snmpd
```

Then check:

```bash
sudo systemctl status snmpd
```

---

# 9. Test the SNMP client

Because the server and client are on the same machine, we can use:

```text
localhost
```

as the target.

First, test a specific OID:

```bash
snmpget -v2c -c public localhost 1.3.6.1.2.1.1.1.0
```

Explanation:

```text
-v2c
```

Use SNMP version 2c.

```text
-c public
```

Use the community string `public`.

```text
localhost
```

Connect to the local SNMP server.

```text
1.3.6.1.2.1.1.1.0
```

Request the system description.

You should receive a response containing information about the operating system and SNMP agent.

---

# 10. Using `snmpwalk`

Instead of querying one OID at a time, we can use:

```bash
snmpwalk -v2c -c public localhost
```

`snmpwalk` automatically queries a sequence of OIDs and displays the available information.

This is particularly useful during SNMP enumeration.

For example:

```bash
snmpwalk -v2c -c public localhost 1.3.6.1.2.1.1
```

queries the system information branch.

---

# 11. Allowing access to the complete SNMP tree

The default configuration restricts access using:

```text
-V systemonly
```

For a **local learning laboratory**, we can remove this restriction.

Change:

```text
rocommunity public default -V systemonly
```

to:

```text
rocommunity public
```

You can also comment out the `systemonly` view definitions:

```text
# view   systemonly  included   .1.3.6.1.2.1.1
# view   systemonly  included   .1.3.6.1.2.1.25.1
```

The resulting configuration can contain:

```text
agentaddress 127.0.0.1,[::1]

rocommunity public
```

Restart the service:

```bash
sudo systemctl restart snmpd
```

Now try:

```bash
snmpwalk -v2c -c public localhost
```

The client can walk through a much larger part of the SNMP information tree.

> **Warning:** This configuration is intended for a local laboratory. Do not expose an SNMPv2c agent with a simple community string such as `public` to an untrusted network.

---

# 12. Understanding OIDs

SNMP information is organized using **OIDs (Object Identifiers)**.

An OID is a hierarchical numerical identifier.

For example:

```text
1.3.6.1.2.1.1.1.0
```

can be interpreted as a path through the SNMP hierarchy.

You can think of it as a filesystem:

```text
/
└── 1
    └── 3
        └── 6
            └── 1
                └── 2
                    └── 1
                        └── 1
                            └── 1
                                └── 0
```

Each branch contains different types of information.

---

# 13. MIBs

A **MIB (Management Information Base)** describes SNMP objects in a human-readable format.

MIBs provide information such as:

* Object name
* OID
* Data type
* Access permissions
* Description

Instead of working only with:

```text
1.3.6.1.2.1.1.1.0
```

a MIB can allow the object to be represented by a more understandable name.

---

# 14. `/etc/snmp/snmp.conf`

There is another important configuration file:

```text
/etc/snmp/snmp.conf
```

This file is related to the SNMP client/library configuration.

On some systems you may find:

```text
mibs :
```

This disables automatic loading of MIB files.

Therefore, you may see numeric OIDs instead of symbolic names when using SNMP commands.

For our basic laboratory, this does not prevent SNMP from working.

---

# 15. Server and client on the same machine

Our complete laboratory looks like this:

```text
+---------------------------------------+
|              Linux Host               |
|                                       |
|   +-------------+                     |
|   | SNMP Client  |                     |
|   |  snmpwalk    |                     |
|   +------+------+\                      |
|          |                            |
|          | UDP 161                    |
|          v                            |
|   +-------------+                     |
|   | SNMP Server  |                     |
|   |    snmpd     |                     |
|   +-------------+                     |
|                                       |
+---------------------------------------+
```

The client sends an SNMP request to:

```text
localhost:161
```

The `snmpd` service receives the request and returns the requested information.

---

# 16. Complete local test

Check that the service is running:

```bash
sudo systemctl status snmpd
```

Check that port 161 is listening:

```bash
sudo ss -lunp | grep 161
```

Test a specific OID:

```bash
snmpget -v2c -c public localhost 1.3.6.1.2.1.1.1.0
```

Then perform a walk:

```bash
snmpwalk -v2c -c public localhost
```

If the configuration allows access to the complete tree, `snmpwalk` should return many SNMP objects.

---

# 17. Useful SNMP enumeration tools

Several tools are commonly used when enumerating SNMP services.

## snmpwalk

Used to retrieve a large number of SNMP objects:

```bash
snmpwalk -v2c -c public 10.10.10.10
```

For our local laboratory:

```bash
snmpwalk -v2c -c public localhost
```

---

## onesixtyone

`onesixtyone` can be used to test SNMP community strings.

Install it:

```bash
sudo apt install onesixtyone
```

Example:

```bash
onesixtyone 10.10.10.10
```

You can also provide a community-string wordlist.

---

## braa

`braa` is another tool that can be used for SNMP enumeration.

Example:

```bash
braa public@10.10.10.10:.1.3.6.1.2.1.1.*
```

This queries objects below the specified OID.

---

# 18. SNMP versions

There are three commonly encountered SNMP versions.

### SNMPv1

SNMPv1 does not provide encryption and has very limited security.

### SNMPv2c

SNMPv2c uses community strings.

Example:

```bash
snmpwalk -v2c -c public localhost
```

The community string is transmitted without encryption.

### SNMPv3

SNMPv3 provides stronger security features, including authentication and encryption.

It is more complex to configure but should be preferred when stronger SNMP security is required.

---

# 19. Security considerations

SNMP can expose a significant amount of information about a system.

A poorly configured SNMP service may reveal:

* Host information
* Network interfaces
* Operating system information
* Processes
* Network configuration
* System statistics

Avoid configurations such as:

```text
rwcommunity public
```

or:

```text
rwuser noauth
```

when they are exposed to an untrusted network.

Read-write SNMP access can allow configuration changes and therefore presents a much greater security risk than read-only access.

For a real environment:

* Restrict access to trusted hosts
* Avoid default community strings
* Prefer SNMPv3 when possible
* Use firewall rules
* Avoid exposing UDP 161 directly to the Internet

---

# 20. Local lab vs network configuration

Our laboratory uses:

```text
agentaddress 127.0.0.1,[::1]
```

This means that only the local machine can communicate with `snmpd`.

If the SNMP server were on another machine, the agent would need to listen on an appropriate network interface/address.

For example:

```text
SNMP Client
10.10.10.20
      |
      | UDP 161
      |
      v
SNMP Server
10.10.10.10
```

In that scenario, firewall rules and the `agentaddress` configuration would need to allow the network connection.

---

# 21. Troubleshooting

## Check the service

```bash
sudo systemctl status snmpd
```

## Restart the service

```bash
sudo systemctl restart snmpd
```

## Check port 161

```bash
sudo ss -lunp | grep 161
```

## Check the configuration

```bash
sudo cat /etc/snmp/snmpd.conf
```

## Test locally

```bash
snmpget -v2c -c public localhost 1.3.6.1.2.1.1.1.0
```

## Test with `snmpwalk`

```bash
snmpwalk -v2c -c public localhost
```

If you receive an error, first check:

1. Is `snmpd` running?
2. Is UDP 161 listening?
3. Is the community string correct?
4. Is the requested OID allowed by the configured view?
5. Is a firewall blocking the connection?

---

# 22. Key concepts

The most important concepts from this laboratory are:

| Concept          | Meaning                                                     |
| ---------------- | ----------------------------------------------------------- |
| SNMP             | Protocol used to monitor/manage systems and network devices |
| `snmpd`          | SNMP server/agent                                           |
| `snmp`           | SNMP client tools                                           |
| UDP 161          | SNMP requests and responses                                 |
| UDP 162          | SNMP traps                                                  |
| Community string | Access string used by SNMPv1/v2c                            |
| OID              | Numerical identifier for an SNMP object                     |
| MIB              | Description of SNMP objects                                 |
| `snmpget`        | Retrieves a specific SNMP object                            |
| `snmpwalk`       | Walks through a hierarchy of SNMP objects                   |
| SNMPv2c          | Community-based SNMP version                                |
| SNMPv3           | SNMP version with authentication and encryption             |

---

# 23. Quick reference

### Install

```bash
sudo apt update
sudo apt install snmp snmpd
```

### Start service

```bash
sudo systemctl enable --now snmpd
```

### Check service

```bash
sudo systemctl status snmpd
```

### Check UDP 161

```bash
sudo ss -lunp | grep 161
```

### Test SNMP

```bash
snmpget -v2c -c public localhost 1.3.6.1.2.1.1.1.0
```

### Walk SNMP tree

```bash
snmpwalk -v2c -c public localhost
```

### Configuration

```text
/etc/snmp/snmpd.conf
```

### Client configuration

```text
/etc/snmp/snmp.conf
```

---

# Conclusion

In this laboratory, we configured an SNMP server and client on the same Linux machine.

The `snmpd` service acts as the SNMP agent and listens on **UDP port 161**. The SNMP client tools communicate with it using the configured community string.

The basic workflow is:

```text
SNMP Client
     |
     | SNMP request
     | UDP 161
     v
   snmpd
     |
     | SNMP response
     v
SNMP Client
```

Once this local configuration is understood, the same concepts can be applied to SNMP enumeration on remote systems in a controlled lab environment.
