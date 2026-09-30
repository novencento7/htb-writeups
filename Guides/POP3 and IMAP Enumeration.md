# POP3 and IMAP Enumeration

POP3 and IMAP are protocols used to access email stored on a mail server.

During penetration testing, these services can be useful because email accounts may contain:

* Credentials
* Passwords
* Internal information
* Configuration details
* Sensitive documents
* Information about other users or services

The two main protocols are:

* **POP3** - Post Office Protocol version 3
* **IMAP** - Internet Message Access Protocol

---

## 1. POP3

POP3 is commonly exposed on:

| Port | Protocol | Description            |
| ---- | -------- | ---------------------- |
| 110  | POP3     | Unencrypted/plain POP3 |
| 995  | POP3S    | POP3 over SSL/TLS      |

POP3 is primarily designed to download messages from the mail server.

---

## 2. IMAP

IMAP is commonly exposed on:

| Port | Protocol | Description            |
| ---- | -------- | ---------------------- |
| 143  | IMAP     | Unencrypted/plain IMAP |
| 993  | IMAPS    | IMAP over SSL/TLS      |

Unlike POP3, IMAP allows users to manage messages directly on the server and supports folders/mailboxes.

---

# Enumeration with Nmap

The first step is to determine whether POP3 or IMAP is available.

```bash
sudo nmap -p110,143,993,995 -sC -sV <IP>
```

For example:

```bash
sudo nmap -p110,143,993,995 -sC -sV 10.129.14.128
```

Nmap can identify:

* Open mail ports
* Service versions
* SSL/TLS configuration
* Server banners
* Supported capabilities

If SSL/TLS is available, ports `993` and `995` are particularly interesting.

---

# POP3 Enumeration

## Connecting to POP3

For an unencrypted POP3 service on port 110:

```bash
telnet <IP> 110
```

For example:

```bash
telnet 10.129.14.128 110
```

A successful connection may return something similar to:

```text
+OK POP3 server ready
```

---

## POP3 Commands

After connecting, POP3 provides several commands.

### USER

Specify the username:

```text
USER username
```

### PASS

Provide the password:

```text
PASS password
```

A successful login may return:

```text
+OK Logged in.
```

---

## LIST

After authentication, use `LIST` to see the available messages:

```text
LIST
```

Example:

```text
+OK 3 messages:
1 1250
2 2480
3 1024
.
```

The first number is the message ID and the second is its size.

---

## STAT

`STAT` shows the number of messages and their total size:

```text
STAT
```

Example:

```text
+OK 3 4754
```

This means:

* `3` = number of messages
* `4754` = total size in bytes

---

## RETR

Use `RETR` to retrieve a message.

For example, to read message number 1:

```text
RETR 1
```

The server will return the email, including its headers and body.

Example:

```text
From: user@example.com
To: admin@example.com
Subject: Password Reset

The password has been changed to:
...
```

---

## TOP

Some POP3 servers support the `TOP` command.

It can be used to retrieve the headers and the first lines of a message without downloading the entire email.

```text
TOP 1 10
```

This requests the headers and the first 10 lines of message 1.

---

## QUIT

Terminate the POP3 session:

```text
QUIT
```

---

# POP3 over SSL/TLS

If POP3S is running on port `995`, use OpenSSL:

```bash
openssl s_client -connect <IP>:995
```

For example:

```bash
openssl s_client -connect 10.129.14.128:995
```

Once the TLS connection has been established, POP3 commands can be entered directly:

```text
USER username
PASS password
LIST
RETR 1
```

---

# IMAP Enumeration

IMAP allows access to email while keeping messages and folders on the server.

The most common ports are:

* `143` - IMAP
* `993` - IMAPS

---

# Connecting to IMAP with OpenSSL

For IMAPS on port 993:

```bash
openssl s_client -connect <IP>:993
```

For example:

```bash
openssl s_client -connect 10.129.14.128:993
```

After the TLS connection is established, IMAP commands can be entered.

Unlike POP3, IMAP commands normally use a tag before the command.

For example:

```text
a1 LOGIN username password
```

The tag `a1` identifies the command.

---

# IMAP Login

Use:

```text
a1 LOGIN username password
```

Example:

```text
a1 LOGIN user p4ssw0rd
```

A successful login may return:

```text
a1 OK LOGIN completed
```

---

# Listing Mailboxes

After authentication, list the available folders:

```text
a2 LIST "" "*"
```

The server may return:

```text
* LIST (\HasNoChildren) "/" "INBOX"
* LIST (\HasNoChildren) "/" "Sent"
* LIST (\HasNoChildren) "/" "Drafts"
a2 OK LIST completed
```

This allows us to identify available mailboxes.

---

# Selecting INBOX

Before reading messages, select the mailbox:

```text
a3 SELECT INBOX
```

The server may return:

```text
* 5 EXISTS
```

This means that the INBOX contains 5 messages.

---

# Searching for Messages

Use:

```text
a4 SEARCH ALL
```

Example:

```text
* SEARCH 1 2 3 4 5
a4 OK SEARCH completed
```

This tells us that messages `1` through `5` exist.

---

# Reading Email Headers

Before downloading the complete email, it can be useful to retrieve only the headers.

```text
a5 FETCH 1:* (FLAGS BODY[HEADER.FIELDS (FROM TO SUBJECT DATE)])
```

This can show:

* Sender
* Recipient
* Subject
* Date

Example:

```text
From: user@example.com
To: admin@example.com
Subject: Password Reset
Date: Wed, 30 Sep 2026 10:15:00 +0000
```

This is useful when there are many messages and we want to identify interesting ones before downloading them.

---

# Reading an Email

To retrieve the complete first message:

```text
a6 FETCH 1 BODY[]
```

For the second message:

```text
a7 FETCH 2 BODY[]
```

For example:

```text
a6 FETCH 1 BODY[]
```

The server may return:

```text
* 1 FETCH (BODY[] {1234}
From: user@example.com
To: admin@example.com
Subject: Internal Information

Hello,

The password for the development server is:
...
)
a6 OK FETCH completed
```

The complete message can contain both headers and the email body.

---

# Reading Multiple Emails

If `SEARCH ALL` returned:

```text
* SEARCH 1 2 3 4 5
```

we can retrieve them individually:

```text
a6 FETCH 1 BODY[]
a7 FETCH 2 BODY[]
a8 FETCH 3 BODY[]
a9 FETCH 4 BODY[]
a10 FETCH 5 BODY[]
```

This is useful when looking for credentials, usernames, internal hostnames or other information.

---

# Useful IMAP Commands

| Command  | Purpose                      |
| -------- | ---------------------------- |
| `LOGIN`  | Authenticate                 |
| `LIST`   | List mailboxes/folders       |
| `SELECT` | Select a mailbox             |
| `SEARCH` | Search for messages          |
| `FETCH`  | Retrieve message data        |
| `STATUS` | Retrieve mailbox information |
| `CREATE` | Create a mailbox             |
| `DELETE` | Delete a mailbox             |
| `LOGOUT` | End the session              |

---

# Useful IMAP Workflow

A simple workflow for an authenticated IMAPS server is:

```text
a1 LOGIN username password
a2 LIST "" "*"
a3 SELECT INBOX
a4 SEARCH ALL
a5 FETCH 1:* (FLAGS BODY[HEADER.FIELDS (FROM TO SUBJECT DATE)])
a6 FETCH 1 BODY[]
```

The process is:

```text
LOGIN
  |
  v
LIST mailboxes
  |
  v
SELECT INBOX
  |
  v
SEARCH messages
  |
  v
Read headers
  |
  v
FETCH interesting messages
```

---

# Using cURL with IMAPS

If credentials are available, `curl` can also be used to interact with IMAP.

For example:

```bash
curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd
```

To access a specific mailbox:

```bash
curl -k 'imaps://10.129.14.128/INBOX' --user user:p4ssw0rd
```

The `-k` option tells `curl` to ignore certificate validation errors, which is often necessary in lab environments where the server uses a self-signed certificate.

---

# OpenSSL vs cURL

Both methods can be useful.

### OpenSSL

```bash
openssl s_client -connect <IP>:993
```

Advantages:

* Interactive
* Allows manual IMAP commands
* Useful for understanding the protocol
* Useful during manual enumeration

### cURL

```bash
curl -k 'imaps://<IP>' --user username:password
```

Advantages:

* Easier to automate
* Useful for retrieving mailboxes
* Can be integrated into scripts

---

# Important Differences: POP3 vs IMAP

| Feature                   | POP3                   | IMAP                                |
| ------------------------- | ---------------------- | ----------------------------------- |
| Default port              | 110                    | 143                                 |
| SSL/TLS port              | 995                    | 993                                 |
| Server-side folders       | Limited                | Yes                                 |
| Messages remain on server | Usually downloaded     | Yes                                 |
| Folder management         | Limited                | Yes                                 |
| Multiple devices          | Less suitable          | Designed for it                     |
| Useful commands           | `LIST`, `STAT`, `RETR` | `LIST`, `SELECT`, `SEARCH`, `FETCH` |

---

# Practical Enumeration Workflow

When encountering a mail server during a penetration test:

### 1. Scan the mail ports

```bash
sudo nmap -p110,143,993,995 -sC -sV <IP>
```

### 2. Identify the protocol

Determine whether the server provides:

* POP3
* POP3S
* IMAP
* IMAPS

### 3. Check for authentication

For POP3:

```text
USER username
PASS password
```

For IMAP:

```text
a1 LOGIN username password
```

### 4. Enumerate mailboxes

For IMAP:

```text
a2 LIST "" "*"
```

### 5. Select the mailbox

```text
a3 SELECT INBOX
```

### 6. Find available messages

```text
a4 SEARCH ALL
```

### 7. Inspect headers

```text
a5 FETCH 1:* (FLAGS BODY[HEADER.FIELDS (FROM TO SUBJECT DATE)])
```

### 8. Read interesting messages

```text
a6 FETCH 1 BODY[]
```

---

# What to Look For

During authorized penetration testing or an HTB lab, email contents can reveal information such as:

* Usernames
* Passwords
* Internal hostnames
* IP addresses
* Network information
* VPN credentials
* Service credentials
* Password reset links
* Internal documentation
* Information about other employees
* References to other systems

Email enumeration is therefore often an important step after obtaining valid credentials.

---

# Quick Reference

## POP3

```bash
telnet <IP> 110
```

```text
USER username
PASS password
LIST
STAT
RETR 1
QUIT
```

## POP3S

```bash
openssl s_client -connect <IP>:995
```

```text
USER username
PASS password
LIST
RETR 1
QUIT
```

## IMAP

```bash
openssl s_client -connect <IP>:993
```

```text
a1 LOGIN username password
a2 LIST "" "*"
a3 SELECT INBOX
a4 SEARCH ALL
a5 FETCH 1:* (FLAGS BODY[HEADER.FIELDS (FROM TO SUBJECT DATE)])
a6 FETCH 1 BODY[]
a7 LOGOUT
```

## IMAPS with cURL

```bash
curl -k 'imaps://<IP>' --user username:password
```

---

# Conclusion

POP3 and IMAP can provide valuable information during service enumeration.

POP3 is mainly focused on retrieving messages, while IMAP provides much more extensive control over mailboxes and messages.

When IMAPS is available, `openssl s_client` provides a simple way to manually interact with the server and understand how IMAP works:

```bash
openssl s_client -connect <IP>:993
```

From there, the basic IMAP workflow is:

```text
LOGIN → LIST → SELECT → SEARCH → FETCH
```

This workflow is enough to authenticate, enumerate mailboxes, identify messages and retrieve their contents.
