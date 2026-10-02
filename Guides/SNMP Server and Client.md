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

In this c
