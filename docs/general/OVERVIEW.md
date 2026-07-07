# Mail Server — What It Is and What It Does

A **mail server** is a computer (or program running on one) whose job is to
**send, receive, store, and forward email** on behalf of its users. Every time
you press "Send" in Gmail, Outlook, or any other email app, a chain of mail
servers quietly carries your message across the internet and drops it into the
recipient's inbox.

---

## Table of Contents

1. [What is a Mail Server?](#what-is-a-mail-server)
2. [End-to-End Email Flow](#end-to-end-email-flow)
3. [Core Components](#core-components)
4. [Key Protocols & Ports](#key-protocols--ports)
5. [DNS Records That Make Email Work](#dns-records-that-make-email-work)
6. [Security & Spam](#security--spam)
7. [Self-Hosting Basics](#self-hosting-basics)
8. [Glossary](#glossary)

---

## What is a Mail Server?

Think of a mail server as a **digital post office**. When you mail a physical
letter, you hand it to the post office — it figures out the route, passes it
between sorting facilities, and delivers it to the right mailbox.

A mail server performs **two cooperating roles**:

```mermaid
graph TD
    subgraph "MAIL SERVER"
        direction LR
        O["📤 OUTGOING SERVER<br>---<br>Accepts messages you send<br>and relays them to their<br>destination via SMTP<br><br>Protocol: SMTP"]
        I["📥 INCOMING SERVER<br>---<br>Receives messages for users<br>stores them, and lets people<br>read via IMAP or POP3<br><br>Protocol: IMAP / POP3"]
    end
    
    style O fill:#2196F3,color:#fff,stroke:#1976D2,stroke-width:2px
    style I fill:#FF9800,color:#fff,stroke:#F57C00,stroke-width:2px
```

---

## End-to-End Email Flow

Suppose **alice@example.com** sends an email to **bob@sample.org**:

```mermaid
sequenceDiagram
    participant A as 👩 Alice (MUA)
    participant MSA as 📤 Outgoing Server<br/>(example.com SMTP)
    participant DNS as 🌐 DNS
    participant MTA as 📬 Incoming Server<br/>(sample.org MTA)
    participant MDA as 📁 Delivery Agent<br/>(MDA)
    participant B as 👨 Bob (MUA)

    A->>MSA: Submit message (Port 587, STARTTLS)
    MSA->>DNS: MX record for sample.org?
    DNS-->>MSA: mail.sample.org
    MSA->>MTA: SMTP relay (Port 25)
    MTA->>MTA: SPF / DKIM / DMARC checks
    MTA->>MDA: Accepted → deliver to mailbox
    MDA->>MDA: Written to Bob's mailbox
    B->>MDA: Fetch mail (IMAP 993 / POP3 995)
    MDA-->>B: Message displayed ✅
```

### Step-by-Step Breakdown

| Step | What Happens |
|------|-------------|
| **1. Compose & Submit** | Alice writes the message; the app submits it to her provider's outgoing server over **SMTP** (port 587, authenticated) |
| **2. Look up destination** | Alice's server queries DNS for the MX record of `sample.org` |
| **3. Server-to-server transfer** | Alice's server opens an SMTP connection to `sample.org`'s server (port 25) |
| **4. Acceptance & filtering** | Bob's server checks SPF/DKIM/DMARC, runs spam filtering, accepts or rejects |
| **5. Delivery to mailbox** | A **delivery agent (MDA)** files the message into Bob's mailbox |
| **6. Bob reads it** | Bob's app retrieves the message via **IMAP** (port 993) or **POP3** (port 995) |

> **Key insight:** Servers talk to each other via **SMTP**; users read their mail via **IMAP / POP3**.

---

## Core Components

```mermaid
graph LR
    A["👩 User<br/>MUA"] -->|"Port 587<br/>STARTTLS"| B["MSA<br/>Submission"]
    B -->|"SMTP"| C["MTA<br/>Routing"]
    C -->|"DNS MX<br/>Port 25"| D["MTA<br/>Receiving"]
    D --> E["MDA<br/>Delivery"]
    E --> F["💾 Mailbox<br/>Storage"]
    F -->|"Port 993<br/>IMAP"| G["👨 User<br/>MUA"]

    style A fill:#4CAF50,color:#fff
    style G fill:#4CAF50,color:#fff
    style B fill:#2196F3,color:#fff
    style C fill:#FF9800,color:#fff
    style D fill:#FF9800,color:#fff
    style E fill:#9C27B0,color:#fff
    style F fill:#607D8B,color:#fff
```

| Component | Full Name | Role | Example Software |
|-----------|-----------|------|-----------------|
| **MUA** | Mail User Agent | The app a person uses to read/write email | Thunderbird, Apple Mail, Gmail, Outlook |
| **MSA** | Mail Submission Agent | Accepts outgoing mail from a MUA (port 587) | Postfix `submission` |
| **MTA** | Mail Transfer Agent | Routes and relays mail between servers over SMTP | Postfix, Exim, Sendmail |
| **MDA** | Mail Delivery Agent | Files accepted messages into the correct local mailbox | Dovecot LDA, Procmail |
| **Mailbox store** | — | Where messages physically live (files or database) | Maildir, mbox |
| **Access server** | — | Serves stored mail to MUAs via IMAP/POP3 | Dovecot, Courier |

> **Classic self-hosted pair:** **Postfix** (handles SMTP) + **Dovecot** (handles IMAP/POP3 + delivery)

---

## Key Protocols & Ports

| PROTOCOL | PLAIN | ENCRYPTED | PURPOSE |
|---|---|---|---|
| **SMTP** | 25<br>(server) | 465 (TLS)<br>587 (STARTTLS) | Send mail:<br>25 = server relay<br>587 = client submission |
| **IMAP** | 143 | 993 (TLS) | Read mail kept on the server<br>Syncs across all devices |
| **POP3** | 110 | 995 (TLS) | Download mail to one device<br>Usually deletes from server |

### IMAP vs POP3 — Which to Use?

```mermaid
graph TB
    subgraph IMAP["📱 IMAP — Multi-device (recommended)"]
        direction TB
        I1["📧 Mail stays on server"]
        I2["📱 Phone reads it → marked ✅ Read"]
        I3["💻 Laptop also sees ✅ Read"]
        I4["🔄 Fully synchronized!"]
        I1 --> I2 --> I3 --> I4
    end

    subgraph POP3["💾 POP3 — Single device"]
        direction TB
        P1["📧 Mail on server"]
        P2["💻 Laptop downloads it"]
        P3["🗑️ Deleted from server"]
        P4["📱 Phone cannot see it"]
        P1 --> P2 --> P3 --> P4
    end

    style IMAP fill:#E3F2FD
    style POP3 fill:#FFF3E0
```

> **Use IMAP** for multiple devices — it's what nearly everyone uses today.
> **Port 25** is for server-to-server relay and is often blocked by ISPs to fight spam.
> **Port 587** with authentication is what your mail *app* uses to submit new mail.

---

## DNS Records That Make Email Work

Email delivery depends heavily on **DNS** (the internet's address book).

```mermaid
graph LR
    subgraph DNS_RECORDS["🌐 DNS Records for Email"]
        MX["📮 MX\n(Mail Exchange)\nNames the server that\nreceives mail for a domain"]
        A_REC["🔢 A / AAAA\nMaps mail server\nhostname → IP address"]
        PTR["🔄 PTR\n(Reverse DNS)\nMaps IP → hostname\nRequired by many receivers"]
        SPF["🛡️ SPF\nLists servers allowed\nto send for your domain"]
        DKIM["🔑 DKIM\nCryptographic signature\non outgoing mail"]
        DMARC["📋 DMARC\nPolicy when SPF/DKIM fail\n+ reporting"]
    end

    MX -->|"required"| RECV["📥 Receiving works"]
    SPF -->|"authentication"| AUTH["🔐 Sender verified"]
    DKIM -->|"authentication"| AUTH
    DMARC -->|"policy"| AUTH
    AUTH --> INBOX["📬 Reaches Inbox"]

    style DNS_RECORDS fill:#E8F5E9
    style INBOX fill:#4CAF50,color:#fff
```

| Record | What It Does | Why It Matters |
|--------|-------------|----------------|
| **MX** | Names the server(s) that receive mail for a domain, with priority | Without it, no one can deliver mail to your domain |
| **A / AAAA** | Maps the mail server's hostname to its IPv4 / IPv6 address | The MX record points to a hostname; this resolves it to an IP |
| **PTR** | Maps the server's IP back to its hostname | Many receivers reject mail from IPs without matching reverse DNS |
| **SPF** | Lists which servers are allowed to send mail for your domain | Stops others from forging your domain as the sender |
| **DKIM** | Adds a cryptographic signature to your outgoing mail | Lets receivers verify the message wasn't altered |
| **DMARC** | Tells receivers what to do when SPF/DKIM checks fail | Ties SPF + DKIM together into an enforceable policy |

### Example DNS Records

```dns
; Where to deliver mail for example.com
example.com.        IN  MX   10 mail.example.com.
mail.example.com.   IN  A    203.0.113.25

; SPF: only this server may send mail as example.com
example.com.        IN  TXT  "v=spf1 mx -all"

; DKIM: public key for signature verification (selector "s1")
s1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCSq..."

; DMARC: reject failures, email reports to the postmaster
_dmarc.example.com. IN  TXT  "v=DMARC1; p=reject; rua=mailto:postmaster@example.com"
```

---

## Security & Spam

```mermaid
mindmap
  root(("📧 Email\nSecurity"))
    🔒 Encryption
      TLS / STARTTLS
      Ports 465 / 993 / 995
      Encrypts passwords & content
    🔑 Authentication
      SASL login
      Username + Password
      OAuth tokens
    🚫 Spam Filtering
      SpamAssassin
      Rspamd
      Content analysis
      Sender reputation
    📋 Blacklists
      DNSBLs
      Spamhaus
      IP reputation lists
    📊 Reputation
      Built slowly
      Lost quickly
      IP warm-up required
```

| Security Layer | Description |
|----------------|-------------|
| **TLS / STARTTLS encryption** | Mail connections are encrypted so messages and passwords aren't sent in plain text |
| **SASL authentication** | Before relaying mail, the server verifies your identity — prevents **open relay** abuse |
| **Spam filtering** | Incoming mail is scored using content analysis and authentication results |
| **DNSBLs (blacklists)** | Published lists of known-bad IPs — if your server's IP is listed, mail gets blocked everywhere |
| **Deliverability** | Even a perfect config can land in spam if the IP is new or volume spikes suddenly |

---

## Self-Hosting Basics

```mermaid
graph TD
    A["🎯 Want to run your own mail server?"] --> B{What's your goal?}
    B -->|"Learning / full control"| C["🔧 Build it yourself\nPostfix + Dovecot + Rspamd"]
    B -->|"Self-host easily"| D["📦 All-in-one stack\nMailcow / Mail-in-a-Box"]
    B -->|"Just want email\nthat works"| E["☁️ Managed provider\nGmail / Fastmail / Proton"]

    C --> F["📋 Prerequisites"]
    D --> F
    F --> G["🌐 Static IP address"]
    F --> H["📧 Ability to set PTR / rDNS"]
    F --> I["🔓 Port 25 unblocked"]
    F --> J["✅ Correct MX, SPF, DKIM, DMARC"]
    F --> K["🔄 Ongoing maintenance"]

    style E fill:#FF9800,color:#fff
    style C fill:#4CAF50,color:#fff
    style D fill:#2196F3,color:#fff
```

### Self-Host vs Managed Provider

| Feature | Self-Hosted | Managed Provider<br>(Gmail, Fastmail, etc.) |
|---|---|---|
| **Control/Privacy** | Full control of data | Provider holds your data |
| **Cost** | Server cost + time | Monthly per-user fee |
| **Setup difficulty** | High (DNS, TLS, spam) | Minimal |
| **Deliverability** | Build IP reputation | Established reputation out of box |
| **Maintenance** | Ongoing — on you | Handled for you |

> For most people and businesses, a **managed provider** is the pragmatic choice.
> Self-hosting is worth it for learning, full data ownership, or specialized needs —
> just budget for the operational effort.

---

## Glossary

```mermaid
graph LR
    subgraph Agents["🤖 Mail Agents"]
        MUA2["MUA — Mail User Agent\nEmail app you use"]
        MSA2["MSA — Mail Submission Agent\nAccepts outgoing mail from MUA"]
        MTA2["MTA — Mail Transfer Agent\nRelays mail between servers"]
        MDA2["MDA — Mail Delivery Agent\nFiles mail into mailbox"]
    end

    subgraph Protocols["📡 Protocols"]
        SMTP2["SMTP — Send mail"]
        IMAP2["IMAP — Read mail (server-side)"]
        POP3_2["POP3 — Download mail"]
    end

    subgraph Security["🔐 Security / DNS"]
        MX2["MX — Mail Exchange DNS record"]
        SPF3["SPF — Sender Policy Framework"]
        DKIM3["DKIM — Cryptographic mail signature"]
        DMARC3["DMARC — Auth policy + reporting"]
        TLS2["TLS / STARTTLS — Encryption"]
        DNSBL2["DNSBL — IP blacklist"]
    end

    style Agents fill:#E3F2FD
    style Protocols fill:#E8F5E9
    style Security fill:#FFF3E0
```

| Term | Definition |
|------|-----------|
| **MUA** | Mail User Agent — the email app you use |
| **MSA** | Mail Submission Agent — accepts outgoing mail from a MUA |
| **MTA** | Mail Transfer Agent — relays mail between servers |
| **MDA** | Mail Delivery Agent — files mail into the correct mailbox |
| **SMTP** | Simple Mail Transfer Protocol — sends mail |
| **IMAP** | Internet Message Access Protocol — reads mail kept on the server |
| **POP3** | Post Office Protocol v3 — downloads mail to one device |
| **MX record** | DNS record naming a domain's mail servers |
| **SPF** | Sender Policy Framework — authorizes sending servers |
| **DKIM** | DomainKeys Identified Mail — cryptographically signs mail |
| **DMARC** | Domain-based Message Authentication, Reporting & Conformance |
| **TLS / STARTTLS** | Encryption for mail connections |
| **DNSBL** | DNS Blacklist of bad sender IPs |
| **Open relay** | Misconfigured server that forwards mail for anyone (abused by spammers) |

---

### See Also

- **Next:** [Cloud Providers & Infrastructure →](CLOUD_PROVIDERS.md)
- [Self-Hosting a Mail Server](SELF_HOSTING.md)
- [Anatomy of an Email](EMAIL_ANATOMY.md)
- [Speaking the Protocols by Hand](PROTOCOLS.md)
- [Choosing Mail Server Software](CHOOSING_SOFTWARE.md)

[← Back to index](../../README.md)
