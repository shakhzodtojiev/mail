# Choosing Mail Server Software — Which One Is Best?

**Short answer: there is no single "best" mail server.** The right choice depends
entirely on your goal. This guide gives a clear recommendation for each common scenario,
then explains the options behind each one.

---

## Table of Contents

1. [First: Should You Self-Host at All?](#first-should-you-self-host-at-all)
2. [All-in-One Self-Hosted Stacks](#all-in-one-self-hosted-stacks)
3. [Build-Your-Own: Individual Components](#build-your-own-individual-components)
4. [Managed Providers (Don't Self-Host)](#managed-providers-dont-self-host)
5. [Comparison at a Glance](#comparison-at-a-glance)
6. [Recommendations: "If You Want X, Use Y"](#recommendations-if-you-want-x-use-y)

---

## First: Should You Self-Host at All?

Before picking software, answer this honestly:

```mermaid
graph TD
    A["🤔 Do you want to self-host?"] --> B{Why?}

    B -->|"To learn how email works"| C["✅ Self-host\nBuild it yourself:\nPostfix + Dovecot + Rspamd"]
    B -->|"Full data ownership / privacy"| D["✅ Self-host\nAll-in-one stack:\nMailcow or Mail-in-a-Box"]
    B -->|"Special requirements\nprovider can't meet"| E["✅ Self-host\n(evaluate carefully)"]
    B -->|"Just want email\nthat reliably works"| F["☁️ Use a Managed Provider\nGmail / Fastmail / Proton"]

    C --> G["⚠️ Budget for operational effort:\nDNS, TLS, spam, maintenance"]
    D --> G
    E --> G

    F --> H["✅ No maintenance\nEstablished reputation\nPer-user fee"]

    style F fill:#4CAF50,color:#fff
    style H fill:#4CAF50,color:#fff
    style G fill:#FF9800,color:#fff
```

Once you've decided to self-host, the next fork is:

```
All-in-one stack  →  §2 (easier, faster)
       vs.
Build your own    →  §3 (maximum control + learning)
```

---

## All-in-One Self-Hosted Stacks

These bundle Postfix + Dovecot + spam filtering + DKIM + a web admin (and often webmail)
into one installable package.

```mermaid
graph LR
    subgraph ALL_IN_ONE["📦 All-in-One Stacks"]
        MAILCOW["🐄 Mailcow\nDockerized\n★★★★★"]
        MIAB["📮 Mail-in-a-Box\nOne-script install\n★★★★"]
        IREDMAIL["📧 iRedMail\nBare OS install\n★★★"]
        MAILU["🦆 Mailu\nDocker lightweight\n★★★★"]
        DMS["🐳 docker-mailserver\nOne container, no UI\n★★★"]
        MODOBOA["🐍 Modoboa\nPython/Django admin\n★★★"]
    end

    style MAILCOW fill:#4CAF50,color:#fff
    style MIAB fill:#2196F3,color:#fff
```

| Stack | What It Is | Best For | Watch Out For |
|-------|-----------|----------|---------------|
| **Mailcow** | Docker-based, modern web UI, very popular | Self-hosters wanting polished, full-featured server | Needs Docker; wants decent RAM |
| **Mail-in-a-Box** | Opinionated, one-script install on a clean box | Beginners wanting "just works" with minimal choices | Less flexible by design; takes over the box |
| **iRedMail** | Mature, runs on bare OS, free + paid admin panel | Traditional VM installs, multiple distros | Admin UI is basic in the free tier |
| **Mailu** | Docker-based, lightweight, config-file driven | Container fans wanting something lean | Fewer features than Mailcow |
| **docker-mailserver** | Just the mail stack in one container, no UI | Users comfortable with config files | No admin web UI |
| **Modoboa** | Python/Django admin + mail stack | Those wanting a clean management UI | Smaller community |

> **The usual picks:** **Mailcow** (most popular, feature-rich) or **Mail-in-a-Box** (simplest).

---

## Build-Your-Own: Individual Components

Maximum control and the best way to *understand* the system.
You pick one from each layer:

```mermaid
graph TD
    subgraph STACK["🔧 Build-Your-Own Stack"]
        MTA_L["MTA — Sends & Relays Mail (SMTP)"]
        IMAP_L["IMAP/POP3 — Serves Mail to Apps"]
        SPAM_L["Spam Filtering & DKIM"]
        TLS_L["TLS Certificates"]
    end

    MTA_L --> POSTFIX["⭐ Postfix\nDefault recommendation\nSecure, well-documented"]
    MTA_L --> EXIM["Exim\nExtremely flexible\nSteeper config"]
    MTA_L --> SENDMAIL["Sendmail\nOriginal, archaic config\n(avoid for new setups)"]

    IMAP_L --> DOVECOT["⭐ Dovecot\nDefault recommendation\nFast, secure, pairs with Postfix"]
    IMAP_L --> COURIER["Courier\nOlder alternative\nLess common today"]

    SPAM_L --> RSPAMD["⭐ Rspamd\nModern, fast\nDoes spam + DKIM signing"]
    SPAM_L --> SPAMASS["SpamAssassin\nClassic, mature\nSlower than Rspamd"]
    SPAM_L --> OPENDKIM["OpenDKIM\nDKIM-only milter\nIf not using Rspamd"]

    TLS_L --> LE["⭐ Let's Encrypt\nFree, auto-renewing"]

    style POSTFIX fill:#4CAF50,color:#fff
    style DOVECOT fill:#4CAF50,color:#fff
    style RSPAMD fill:#4CAF50,color:#fff
    style LE fill:#4CAF50,color:#fff
```

### The Canonical Stack

> **RECOMMENDED SELF-HOSTED STACK**
> 
> **Postfix + Dovecot + Rspamd + Let's Encrypt**
> 
> It's what all-in-one bundles assemble for you, and what the Self-Hosting Guide builds by hand.

---

## Managed Providers (Don't Self-Host)

If your goal is *email that reliably works* rather than *running a mail server*,
a managed provider is almost always the right answer.

```mermaid
graph LR
    subgraph PROVIDERS["☁️ Managed Providers"]
        GW["🔵 Google Workspace\nBest deliverability\nGmail + Docs/Drive"]
        M365["🟦 Microsoft 365\nOutlook ecosystem\nExchange integration"]
        FM["⚡ Fastmail\nPrivacy-respecting\nCustom domains, fast"]
        PM["🔒 Proton Mail\nEnd-to-end encryption\nMaximum privacy"]
        ZOHO["🟡 Zoho Mail\nBudget-friendly\nGenerous free tier"]
    end

    PROVIDERS --> BENEFITS["✅ Benefits"]
    BENEFITS --> B1["Established IP reputation"]
    BENEFITS --> B2["Zero maintenance"]
    BENEFITS --> B3["Built-in security"]
    BENEFITS --> B4["Reliable deliverability"]

    style GW fill:#4285F4,color:#fff
    style M365 fill:#0078D4,color:#fff
    style FM fill:#FF5722,color:#fff
    style PM fill:#6D4AFF,color:#fff
    style ZOHO fill:#E8A400,color:#fff
```

| Provider | Best For |
|----------|----------|
| **Google Workspace** | Businesses wanting Gmail + Docs/Drive; excellent deliverability |
| **Microsoft 365** | Organizations in the Microsoft/Outlook ecosystem |
| **Fastmail** | Privacy-respecting, fast; great for individuals & small teams; custom domains |
| **Proton Mail** | End-to-end encryption / maximum privacy focus |
| **Zoho Mail** | Budget-friendly business mail, generous free tier |

---

## Comparison at a Glance

```mermaid
graph LR
    subgraph COMPARE["⚖️ Comparison Matrix"]
        AIO["📦 All-in-One Stack\n(Mailcow, MIAB)"]
        BYO["🔧 Build-Your-Own\n(Postfix + Dovecot)"]
        MANAGED["☁️ Managed Provider\n(Gmail, Fastmail)"]
    end
```

|   | All-in-One Stack | Build-Your-Own | Managed Provider |
|---|---|---|---|
| Control/Privacy | Full | Full | Provider holds data |
| Setup difficulty | Medium | High | Minimal |
| Maintenance | On you | On you | Handled |
| Deliverability | You build rep. | You build rep. | Established |
| Cost | Server + time | Server + more time | Per-user fee |
| Best for | Self-host without<br>hand-wiring | Learning / total<br>control | Just want it to work |
| Examples | Mailcow, MIAB | Postfix + Dovecot | Workspace, Fastmail |

---

## Recommendations: "If You Want X, Use Y"

```mermaid
graph TD
    WANT["🎯 What do you want?"] --> W1
    WANT --> W2
    WANT --> W3
    WANT --> W4
    WANT --> W5

    W1["Just email that works,\nno fuss"] --> R1["☁️ Managed provider\nFastmail (individuals)\nGoogle Workspace / M365 (business)"]
    W2["Self-host the easy way"] --> R2["📦 Mailcow (feature-rich)\nor Mail-in-a-Box (simplest)"]
    W3["Learn how mail servers\nactually work"] --> R3["🔧 Build it yourself:\nPostfix + Dovecot + Rspamd\nSee Self-Hosting Guide"]
    W4["Privacy / encryption\nis top priority"] --> R4["🔒 Proton Mail (managed)\nor self-host with full data ownership"]
    W5["Developer testing\nmail sending"] --> R5["🧪 Local dev server:\naiosmtpd or Mailpit/Mailtrap\n(see Protocols Guide §6)"]

    style R1 fill:#4CAF50,color:#fff
    style R2 fill:#2196F3,color:#fff
    style R3 fill:#FF9800,color:#fff
    style R4 fill:#9C27B0,color:#fff
    style R5 fill:#607D8B,color:#fff
```

> **Bottom line:** "best" = the option that matches your goal.
> Most people should use a managed provider.
> Self-hosters should reach for an all-in-one stack like Mailcow unless learning the
> internals is the point — in which case build from Postfix + Dovecot + Rspamd.

---

### See Also

- [← Troubleshooting & Ops](TROUBLESHOOTING.md) · [Self-Hosting Guide](SELF_HOSTING.md) · [Overview](OVERVIEW.md)

[← Back to index](../../README.md)
