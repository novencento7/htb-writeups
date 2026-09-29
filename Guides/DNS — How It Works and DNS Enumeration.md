# DNS — How It Works and DNS Enumeration

## 1. Introduction

DNS stands for **Domain Name System**.

Its main purpose is to translate human-readable domain names into IP addresses.

For example:

```text
www.example.com
        ↓
    93.184.216.34
```

Humans prefer names, while computers communicate using IP addresses.

DNS provides the system that connects the two.

However, DNS is much more than a simple "name → IP" database.

It is a **distributed and hierarchical system** made of different levels of DNS servers.

Understanding this hierarchy is essential before learning DNS enumeration.

---

# 2. The DNS Hierarchy

DNS is organized like a tree.

At the top there is the **Root**.

Below the Root are the **Top-Level Domains (TLDs)**.

Below the TLDs are the domains and their authoritative DNS servers.

A simplified structure looks like this:

```text
                         .
                         │
              ┌──────────┴──────────┐
              │                     │
             com                   org
              │                     │
          example.org          example.com
              │
        ┌─────┴─────┐
        │           │
       www         mail
```

The dot:

```text
.
```

is the **DNS Root**.

A domain such as:

```text
www.example.com
```

can therefore be viewed from right to left:

```text
www . example . com .
 │       │       │    │
host   domain   TLD  root
```

The final dot is normally omitted when we write the domain.

---

# 3. Root Servers

The first level of the DNS hierarchy is the **Root DNS system**.

Root servers do not normally know the IP address of:

```text
www.example.com
```

Instead, they know where to find the DNS servers responsible for the `.com` TLD.

For example:

```text
Client
  │
  ▼
Root DNS
  │
  └── "I don't know example.com,
       but the .com TLD servers
       can help you."
```

The Root therefore directs the query toward the appropriate TLD servers.

There are 13 logical root server identities, distributed globally through many physical instances.

---

# 4. TLD Servers

TLD means:

**Top-Level Domain**

Examples include:

```text
.com
.org
.net
.it
.htb
```

A TLD server is responsible for knowing which authoritative DNS servers are responsible for domains underneath that TLD.

For example, for:

```text
example.com
```

the `.com` DNS infrastructure can tell us which DNS servers are authoritative for:

```text
example.com
```

It does not necessarily know the IP address of:

```text
www.example.com
```

Instead, it points us to the authoritative DNS servers for `example.com`.

---

# 5. Authoritative DNS Servers

An **authoritative DNS server** is the server that contains the actual DNS records for a particular DNS zone.

For example, suppose:

```text
example.com
```

is managed by:

```text
ns1.example.com
```

The authoritative server may contain records such as:

```text
www.example.com      A       93.184.216.34
mail.example.com     A       93.184.216.35
```

This server is authoritative because it has the information for that DNS zone.

The important distinction is:

```text
Root
  ↓
Knows where the TLD is

TLD
  ↓
Knows where the domain's authoritative DNS servers are

Authoritative DNS
  ↓
Knows the actual records for the zone
```

---

# 6. What Happens When a Client Wants an IP?

Suppose a user enters:

```text
www.example.com
```

into a browser.

The computer needs to discover the IP address.

The client normally does **not** directly perform the entire Root → TLD → Authoritative process itself.

Instead, it usually asks a **recursive DNS resolver**.

For example:

```text
Client
   │
   │ "What is the IP of www.example.com?"
   ▼
Recursive DNS Resolver
```

The resolver then performs the DNS lookup process.

---

# 7. Recursive DNS Resolver

A recursive DNS resolver acts on behalf of the client.

For example:

```text
Client
   │
   │ Query
   ▼
Recursive Resolver
   │
   ├── Root
   │
   ├── .com TLD
   │
   └── example.com authoritative DNS
```

The resolver collects the answer and sends it back to the client.

The client therefore normally only sees:

```text
Client
   │
   ▼
Recursive Resolver
   │
   ▼
IP address
```

The recursive resolver handles the rest.

---

# 8. DNS Caching

The recursive resolver normally caches DNS responses.

This is important because the resolver does not need to repeat the entire process every time.

For example, if someone previously requested:

```text
www.example.com
```

the resolver may already have:

```text
www.example.com → 93.184.216.34
```

in its cache.

The next client can receive the answer immediately.

DNS records therefore have a **TTL (Time To Live)** that determines how long they can be cached.

For example:

```text
www.example.com.    300    IN    A    93.184.216.34
```

The value:

```text
300
```

means that the record can be cached for 300 seconds.

---

# 9. The Complete Resolution Process

Let's follow the resolution of:

```text
www.example.com
```

### Step 1 — Client

The client asks its configured recursive resolver:

```text
"What is the IP address of www.example.com?"
```

### Step 2 — Resolver → Root

The resolver asks the Root DNS system:

```text
"Who is responsible for .com?"
```

The Root responds with information about the `.com` TLD servers.

### Step 3 — Resolver → .com TLD

The resolver asks:

```text
"Who is authoritative for example.com?"
```

The `.com` servers respond with the authoritative DNS servers for:

```text
example.com
```

### Step 4 — Resolver → Authoritative DNS

The resolver asks the authoritative server:

```text
"What is the A record for www.example.com?"
```

The authoritative server responds:

```text
www.example.com → 93.184.216.34
```

### Step 5 — Resolver → Client

Finally, the resolver sends the answer back:

```text
www.example.com → 93.184.216.34
```

The browser can now connect to the IP address.

---

# 10. Why DNS Is Hierarchical

The hierarchy makes DNS scalable.

Imagine if one global server had to store:

```text
every domain
every subdomain
every hostname
every IP address
```

and answer every DNS query on the Internet.

This would not scale.

Instead, responsibility is distributed:

```text
Root
 │
 ├── .com
 │    ├── example.com
 │    ├── google.com
 │    └── ...
 │
 ├── .org
 │    ├── example.org
 │    └── ...
 │
 └── .it
      ├── example.it
      └── ...
```

Each level delegates responsibility to the next level.

---

# 11. DNS Zones

A **DNS zone** is a portion of the DNS namespace managed by a particular set of authoritative DNS servers.

For example:

```text
inlanefreight.htb
```

can be a DNS zone.

The zone can contain records such as:

```text
app.inlanefreight.htb
dev.inlanefreight.htb
internal.inlanefreight.htb
mail1.inlanefreight.htb
```

A zone does not necessarily contain every possible name beneath the domain.

A subdomain can itself be delegated as another zone.

For example:

```text
inlanefreight.htb
        │
        └── internal.inlanefreight.htb
```

`internal.inlanefreight.htb` can therefore be both:

```text
a subdomain
```

and:

```text
a DNS zone
```

This distinction becomes very important during DNS enumeration.

---

# 12. DNS Delegation

DNS delegation allows responsibility for part of the namespace to be transferred to another DNS server.

For example:

```text
inlanefreight.htb
        │
        ├── app.inlanefreight.htb
        │
        └── internal.inlanefreight.htb
                    │
                    ├── dc1
                    ├── dc2
                    ├── ws1
                    └── ws2
```

The parent zone can delegate:

```text
internal.inlanefreight.htb
```

to another authoritative DNS server.

This creates a new administrative boundary.

---

# 13. DNS Records

DNS zones contain different types of records.

The most important ones are:

| Record  | Meaning            |
| ------- | ------------------ |
| `A`     | IPv4 address       |
| `AAAA`  | IPv6 address       |
| `CNAME` | Alias              |
| `MX`    | Mail server        |
| `NS`    | Nameserver         |
| `TXT`   | Text information   |
| `SOA`   | Start of Authority |
| `PTR`   | Reverse DNS        |

---

# 14. A Record

An `A` record maps a hostname to an IPv4 address.

Example:

```text
dc1.internal.inlanefreight.htb.    A    10.129.34.16
```

This means:

```text
dc1.internal.inlanefreight.htb
            ↓
       10.129.34.16
```

---

# 15. NS Record

An `NS` record identifies an authoritative nameserver.

Example:

```text
inlanefreight.htb.    NS    ns.inlanefreight.htb.
```

This means that:

```text
ns.inlanefreight.htb
```

is a nameserver for the zone.

---

# 16. SOA Record

The `SOA` record means:

**Start of Authority**

It contains administrative information about the zone.

For example:

```text
inlanefreight.htb. IN SOA inlanefreight.htb. root.inlanefreight.htb. ...
```

The SOA contains information such as:

* Primary nameserver
* Responsible party
* Serial number
* Refresh interval
* Retry interval
* Expiration interval
* Minimum TTL

---

# 17. TXT Records

TXT records contain arbitrary text associated with a DNS name.

They are commonly used for:

* SPF
* Domain verification
* Email configuration
* Security policies
* Other administrative information

For example:

```text
inlanefreight.htb. TXT "v=spf1 ..."
```

During penetration testing, TXT records can sometimes contain useful information or challenge flags.

---

# 18. Forward DNS

Forward DNS resolves:

```text
hostname → IP address
```

For example:

```text
dc1.internal.inlanefreight.htb
              ↓
        10.129.34.16
```

The main record used is:

```text
A
```

for IPv4.

---

# 19. Reverse DNS

Reverse DNS performs the opposite operation:

```text
IP address → hostname
```

For example:

```text
10.129.34.16
     ↓
dc1.internal.inlanefreight.htb
```

Reverse DNS uses:

```text
PTR
```

records.

IPv4 reverse DNS uses the special domain:

```text
in-addr.arpa
```

For example:

```text
10.129.34.16
```

is represented as:

```text
16.34.129.10.in-addr.arpa
```

The octets are reversed.

---

# 20. Why DNS Is Interesting for Penetration Testing

DNS is particularly useful during reconnaissance because DNS records can reveal infrastructure that is not obvious from a normal web application.

For example, a domain might expose:

```text
www.example.com
mail.example.com
vpn.example.com
dev.example.com
internal.example.com
dc1.internal.example.com
```

This can reveal:

* Development systems
* VPN servers
* Mail servers
* Domain controllers
* Internal workstations
* Internal DNS servers
* Other infrastructure

This is why DNS enumeration is often performed early during reconnaissance.

---

# 21. DNS Zone Transfer

Now that we understand zones, we can understand **Zone Transfer**.

DNS servers need a mechanism to replicate zone information between authoritative servers.

One of the mechanisms used for this is:

```text
AXFR
```

AXFR means:

**Full Zone Transfer**

A server supporting AXFR can provide the complete contents of a DNS zone to an authorized DNS server.

Normally, this should be restricted to trusted DNS servers.

If it is incorrectly configured to allow anyone to request it, an attacker may be able to retrieve the entire zone.

---

# 22. Why AXFR Can Be Dangerous

Suppose the zone contains:

```text
app.inlanefreight.htb
dev.inlanefreight.htb
internal.inlanefreight.htb
mail1.inlanefreight.htb
```

A successful AXFR could reveal all of these at once.

The attacker does not need to guess the names.

Even more importantly, a discovered subdomain may itself be another DNS zone.

For example:

```text
internal.inlanefreight.htb
```

could contain:

```text
dc1.internal.inlanefreight.htb
dc2.internal.inlanefreight.htb
ws1.internal.inlanefreight.htb
ws2.internal.inlanefreight.htb
```

This is why the enumeration process can be recursive.

---

# 23. The HTB Example

In the Hack The Box lab, we had:

```text
DNS server:
10.129.230.172

Zone:
inlanefreight.htb
```

We first queried the DNS server:

```bash
dig @10.129.230.172 inlanefreight.htb
```

Then we tested whether the server allowed a zone transfer:

```bash
dig @10.129.230.172 inlanefreight.htb AXFR
```

The server returned several records.

Among them were:

```text
app.inlanefreight.htb
dev.inlanefreight.htb
internal.inlanefreight.htb
mail1.inlanefreight.htb
```

This showed us that:

```text
inlanefreight.htb
```

contained several subdomains.

---

# 24. Investigating a Discovered Subdomain

The important idea is not to stop here.

We discovered:

```text
internal.inlanefreight.htb
```

The hint told us:

```text
Zones often have the name of a subdomain.
```

Therefore, we tested:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR
```

The transfer succeeded.

We discovered:

```text
dc1.internal.inlanefreight.htb
dc2.internal.inlanefreight.htb
mail1.internal.inlanefreight.htb
ns.internal.inlanefreight.htb
vpn.internal.inlanefreight.htb
ws1.internal.inlanefreight.htb
ws2.internal.inlanefreight.htb
wsus.internal.inlanefreight.htb
```

This demonstrates the importance of understanding the DNS hierarchy.

A discovered subdomain can lead to another zone and therefore to another set of DNS records.

---

# 25. Finding TXT Records

The `internal.inlanefreight.htb` zone also contained TXT records.

We can filter the AXFR output:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR | grep TXT
```

This is useful when an exercise specifically asks for a TXT record.

In our example, the zone contained:

```text
HTB{DN5_z0N3_7r4N5F3r_iskdufhcnlu34}
```

The important point is that the flag was not discovered by guessing.

The process was:

```text
DNS server
    ↓
Zone
    ↓
AXFR
    ↓
Subdomain
    ↓
Subdomain is also a zone
    ↓
Second AXFR
    ↓
TXT record
    ↓
Flag
```

---

# 26. DNS Enumeration Tools

Now that we understand how DNS works, we can look at the tools used to interact with it.

The main tools are:

```text
dig
nslookup
dnsenum
```

---

# 27. dig

`dig` is one of the most useful DNS analysis tools.

Basic syntax:

```bash
dig @DNS_SERVER DOMAIN RECORD_TYPE
```

For example:

```bash
dig @10.129.230.172 inlanefreight.htb A
```

This asks:

```text
DNS server:
10.129.230.172

Question:
What is the A record for inlanefreight.htb?
```

---

# 28. Querying Different Record Types

### A

```bash
dig @10.129.230.172 inlanefreight.htb A
```

### NS

```bash
dig @10.129.230.172 inlanefreight.htb NS
```

### SOA

```bash
dig @10.129.230.172 inlanefreight.htb SOA
```

### TXT

```bash
dig @10.129.230.172 inlanefreight.htb TXT
```

### Reverse lookup

```bash
dig @10.129.230.172 -x 10.129.230.172
```

---

# 29. AXFR with dig

To test a zone transfer:

```bash
dig @10.129.230.172 inlanefreight.htb AXFR
```

The structure is:

```text
dig
 │
 ├── @10.129.230.172
 │       ↓
 │   DNS server
 │
 ├── inlanefreight.htb
 │       ↓
 │     zone
 │
 └── AXFR
         ↓
    full zone transfer
```

If successful, the server returns the zone contents.

---

# 30. nslookup

`nslookup` is another DNS querying tool.

For example:

```bash
nslookup inlanefreight.htb 10.129.230.172
```

The general structure is:

```text
nslookup DOMAIN DNS_SERVER
```

It can be useful for simple DNS queries and troubleshooting.

---

# 31. dnsenum

`dnsenum` automates several DNS enumeration techniques.

Example:

```bash
dnsenum --dnsserver 10.129.230.172 inlanefreight.htb
```

It can help discover:

* DNS records
* Nameservers
* Mail servers
* Subdomains
* Hosts
* Other DNS information

It is useful when you want to automate parts of the enumeration process instead of manually querying every record.

---

# 32. Searching Tool Output

When a tool produces a large amount of information, filtering becomes useful.

For example:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR | grep TXT
```

Find hosts whose IP ends in `.203`:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR | grep '\.203'
```

Save output:

```bash
dnsenum --dnsserver 10.129.230.172 inlanefreight.htb | tee dnsenum.txt
```

Then search:

```bash
grep '\.203' dnsenum.txt
```

---

# 33. Finding an FQDN from an IP

Suppose an exercise asks:

```text
What is the FQDN of the host where the last octet ends with ".203"?
```

We are looking for a record like:

```text
server.internal.inlanefreight.htb.    A    10.129.x.203
```

The answer is:

```text
server.internal.inlanefreight.htb
```

because that is the **Fully Qualified Domain Name (FQDN)**.

---

# 34. A Practical DNS Enumeration Methodology

When given a target DNS server, follow this sequence.

## Step 1 — Identify the DNS server

Example:

```text
10.129.230.172
```

---

## Step 2 — Query the domain

```bash
dig @10.129.230.172 inlanefreight.htb
```

Look at:

```text
SOA
NS
A
```

records.

---

## Step 3 — Try a zone transfer

```bash
dig @10.129.230.172 inlanefreight.htb AXFR
```

If successful, enumerate the returned records.

---

## Step 4 — Identify subdomains

For example:

```text
app.inlanefreight.htb
dev.inlanefreight.htb
internal.inlanefreight.htb
mail1.inlanefreight.htb
```

---

## Step 5 — Test interesting subdomains as zones

For example:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR
```

---

## Step 6 — Enumerate the second zone

Look for:

```text
A
AAAA
NS
MX
TXT
CNAME
```

records.

---

## Step 7 — Filter the results

For example:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR | grep TXT
```

or:

```bash
dig @10.129.230.172 internal.inlanefreight.htb AXFR | grep '\.203'
```

---

# 35. The Most Important Concept

The most important thing to understand is the relationship between:

```text
Domain
Subdomain
Zone
Nameserver
Record
```

They are not the same thing.

For example:

```text
inlanefreight.htb
        │
        ├── app.inlanefreight.htb
        │
        ├── dev.inlanefreight.htb
        │
        └── internal.inlanefreight.htb
                    │
                    ├── dc1.internal.inlanefreight.htb
                    ├── dc2.internal.inlanefreight.htb
                    ├── ws1.internal.inlanefreight.htb
                    └── ws2.internal.inlanefreight.htb
```

`internal.inlanefreight.htb` started as a **subdomain discovered in the parent zone**.

We then discovered that it was also a **DNS zone**.

That zone contained additional records.

This is the key idea behind recursive DNS enumeration.

---

# 36. Quick Summary

DNS is hierarchical:

```text
Root
 ↓
TLD
 ↓
Domain
 ↓
Subdomain / delegated zone
 ↓
Hostname
```

A normal DNS lookup generally involves:

```text
Client
 ↓
Recursive DNS Resolver
 ↓
Root
 ↓
TLD
 ↓
Authoritative DNS
 ↓
Answer
```

For penetration testing, we can directly interact with a DNS server.

Useful tools:

```text
dig
nslookup
dnsenum
```

Useful commands:

```bash
dig @DNS_SERVER DOMAIN
```

```bash
dig @DNS_SERVER DOMAIN AXFR
```

```bash
dig @DNS_SERVER DOMAIN TXT
```

```bash
dig @DNS_SERVER DOMAIN NS
```

```bash
dig @DNS_SERVER DOMAIN SOA
```

The most important enumeration technique discussed in this guide is:

```text
Find a zone
     ↓
Attempt AXFR
     ↓
Discover subdomains
     ↓
Check whether a subdomain is another zone
     ↓
Attempt AXFR again
     ↓
Enumerate additional records
```

Understanding **how DNS works first** makes the tools much easier to understand.
