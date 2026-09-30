# IMAP / POP3 Footprinting

## Introduction

`IMAP` (Internet Message Access Protocol) allows users to access and manage email stored on a remote mail server.

Unlike `POP3` (Post Office Protocol), IMAP keeps emails on the server and supports folders, synchronization, and access from multiple email clients.

IMAP is commonly used to:

- Synchronize a local email client with a mailbox on the server.
- Manage folders directly on the mail server.
- Access multiple mailboxes during a session.
- Browse and manage messages without downloading the entire mailbox.

POP3 provides a simpler set of operations, mainly retrieving and deleting messages.

### IMAP vs POP3

| Protocol | Plaintext | TLS/SSL | Port |
|---|---:|---:|---:|
| POP3 | Yes | No | `110` |
| POP3S | No | Yes | `995` |
| IMAP | Yes | No | `143` |
| IMAPS | No | Yes | `993` |

Without encryption, usernames, passwords, commands, and email contents can potentially be transmitted in plaintext. TLS/SSL should therefore be used when supported by the server.

---

## IMAP Commands

IMAP is a text-based protocol. Commands are normally preceded by a tag such as `A001`, which allows the client to associate server responses with individual commands.

Some useful commands are:

| Command | Description |
|---|---|
| `A001 LOGIN username password` | Authenticate to the server |
| `A002 LIST "" *` | List available mailboxes/directories |
| `A003 CREATE "INBOX"` | Create a mailbox |
| `A004 DELETE "INBOX"` | Delete a mailbox |
| `A005 RENAME "ToRead" "Important"` | Rename a mailbox |
| `A006 LSUB "" *` | List subscribed mailboxes |
| `A007 SELECT INBOX` | Select a mailbox |
| `A008 UNSELECT INBOX` | Leave the selected mailbox |
| `A009 FETCH <ID> ALL` | Retrieve data associated with a message |
| `A010 CLOSE` | Close the selected mailbox |
| `A011 LOGOUT` | Close the IMAP connection |

For example:

```text
A001 LOGIN robin robin
A002 LIST "" *
A003 SELECT INBOX
A004 FETCH 1 ALL
A005 LOGOUT
```

---

## POP3 Commands

POP3 provides a simpler command set:

| Command | Description |
|---|---|
| `USER username` | Specify the username |
| `PASS password` | Authenticate with the password |
| `STAT` | Display the number of messages |
| `LIST` | Display message numbers and sizes |
| `RETR <id>` | Retrieve a message |
| `DELE <id>` | Mark a message for deletion |
| `CAPA` | Display server capabilities |
| `RSET` | Reset the current session state |
| `QUIT` | Close the connection |

---

# Service Enumeration

The first step is to determine whether IMAP or POP3 services are exposed.

A useful Nmap scan is:

```bash
sudo nmap 10.129.14.128 -sV -p110,143,993,995 -sC
```

Example output:

```text
PORT    STATE SERVICE  VERSION
110/tcp open  pop3     Dovecot pop3d
143/tcp open  imap     Dovecot imapd
993/tcp open  ssl/imap Dovecot imapd
995/tcp open  ssl/pop3 Dovecot pop3d
```

This immediately tells us that the target exposes both POP3 and IMAP, including encrypted versions.

Nmap may also retrieve:

- Service versions.
- Supported capabilities.
- TLS certificates.
- Certificate subject names.
- Hostnames associated with the mail server.

For example, the certificate may reveal:

```text
CN=mail1.inlanefreight.htb
```

This can provide useful information about the hostname of the mail server.

---

# Connecting to IMAPS with cURL

Once valid credentials are available, `curl` can be used to interact with an IMAPS server.

```bash
curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd
```

The `-k` option tells `curl` to continue even if the TLS certificate cannot be verified.

A successful connection may return mailbox information such as:

```text
* LIST (\HasNoChildren) "." Important
* LIST (\HasNoChildren) "." INBOX
```

This indicates that the authenticated user has access to mailboxes such as `Important` and `INBOX`.

---

## Verbose IMAPS Connection

The `-v` option provides additional information about the connection:

```bash
curl -k 'imaps://10.129.14.128' --user cry0l1t3:1234 -v
```

The output can reveal:

- The destination port.
- The TLS version.
- The cipher suite.
- Certificate information.
- The certificate subject and issuer.
- The IMAP server banner.
- Supported IMAP capabilities.

For example:

```text
* Trying 10.129.14.128:993...
* Connected to 10.129.14.128 port 993
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* Server certificate:
* subject: ... CN=mail1.inlanefreight.htb ...
< * OK [CAPABILITY IMAP4rev1 ... AUTH=PLAIN] HTB-Academy IMAP4
```

The server banner and certificate can therefore provide useful fingerprinting information.

---

# Connecting with OpenSSL

`openssl s_client` can be used to establish a TLS connection directly to the mail service.

## POP3S

```bash
openssl s_client -connect 10.129.14.128:pop3s
```

A successful connection may show information such as:

```text
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
Verify return code: 18 (self signed certificate)
```

The POP3 banner can then appear:

```text
+OK HTB-Academy POP3 Server
```

## IMAPS

```bash
openssl s_client -connect 10.129.14.128:imaps
```

A successful connection may return:

```text
* OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE ... AUTH=PLAIN] HTB-Academy IMAP4
```

This confirms that the TLS connection to the IMAP service was established successfully.

---

# Security Considerations

Misconfigured mail servers can expose sensitive information.

Potentially interesting configuration issues include:

| Setting | Description |
|---|---|
| `auth_debug` | Enables authentication debugging |
| `auth_debug_passwords` | Controls the logging of authentication passwords |
| `auth_verbose` | Logs unsuccessful authentication attempts and their reasons |
| `auth_verbose_passwords` | Logs passwords used during authentication |
| `auth_anonymous_username` | Specifies the username used with the `ANONYMOUS` SASL mechanism |

Poorly configured logging or authentication settings can expose credentials or other sensitive information.

Mail services should therefore be configured carefully, especially when they are exposed to untrusted networks.

---

# Authentication and Mailbox Enumeration

If valid credentials have been obtained during an authorized assessment, they can be tested against IMAP or POP3.

For example:

```bash
curl -k 'imaps://10.129.14.128' --user robin:robin
```

After authentication, mailbox names can be enumerated:

```text
A001 LIST "" *
```

A mailbox can then be selected:

```text
A002 SELECT INBOX
```

Messages can be retrieved with:

```text
A003 FETCH 1 ALL
```

The exact commands supported depend on the server implementation and configuration.

---

# Practical Enumeration Workflow

A simple workflow for IMAP/POP3 footprinting is:

### 1. Scan the mail ports

```bash
sudo nmap 10.129.14.128 -sV -p110,143,993,995 -sC
```

### 2. Identify the services

Determine whether the target exposes:

- POP3 (`110`)
- IMAP (`143`)
- POP3S (`995`)
- IMAPS (`993`)

### 3. Inspect certificates

Look for:

- Hostnames.
- Organization names.
- Email addresses.
- Certificate validity.
- Self-signed certificates.

### 4. Identify capabilities

Nmap, `curl`, and `openssl` can reveal supported protocol capabilities.

### 5. Test discovered credentials

Only use credentials obtained during an authorized assessment.

```bash
curl -k 'imaps://TARGET' --user USER:PASSWORD
```

### 6. Enumerate mailboxes

After successful authentication:

```text
A001 LIST "" *
A002 SELECT INBOX
```

### 7. Retrieve relevant messages

```text
A003 FETCH 1 ALL
```

---

# Key Takeaways

- `IMAP` is designed for managing mailboxes directly on the server and synchronizing them between clients.
- `POP3` provides a simpler mechanism for retrieving messages.
- IMAP normally uses `143`, while IMAPS uses `993`.
- POP3 normally uses `110`, while POP3S uses `995`.
- Nmap can identify mail services, versions, capabilities, and TLS certificate information.
- `curl` can interact with IMAPS when valid credentials are available.
- `openssl s_client` is useful for inspecting TLS-protected IMAP and POP3 services.
- Mail server certificates can reveal useful hostnames and organizational information.
- Misconfigured authentication and debugging options can expose sensitive information.
- During an authorized penetration test, valid mailbox credentials may allow further enumeration of folders and messages.
