# Troubleshooting & Operations

Running a mail server is mostly about diagnosing *why a message didn't arrive* and
keeping your *sending reputation* healthy. This guide covers reading logs, decoding
bounces, escaping the spam folder, getting off blacklists, and the routine maintenance
that keeps it all working.

---

## Table of Contents

1. [Reading the Logs and the Queue](#reading-the-logs-and-the-queue)
2. [Decoding Bounces and Status Codes](#decoding-bounces-and-status-codes)
3. ["My Mail Lands in Spam" Checklist](#my-mail-lands-in-spam-checklist)
4. [Blacklists (DNSBLs)](#blacklists-dnsbls)
5. [Deliverability Deep-Dive](#deliverability-deep-dive)
6. [Monitoring & Maintenance](#monitoring--maintenance)
7. [Quick Reference: Symptom → Cause → Fix](#quick-reference-symptom--cause--fix)

---

## Reading the Logs and the Queue

The mail log is the **first place to look for everything**. On Debian/Ubuntu:

```bash
sudo tail -f /var/log/mail.log               # follow live
sudo journalctl -u postfix -f                # same, via journald
grep "status=bounced" /var/log/mail.log      # find rejected deliveries
```

### Log Status Values

| Log fragment | Meaning |
|---|---|
| status=sent | ✅ Delivered (or relayed) successfully |
| status=deferred | ⚠️ Temporary failure; Postfix will retry |
| status=bounced | ❌ Permanent failure; an NDR is generated |

### Queue Management

```bash
postqueue -p           # list queued messages (or: mailq)
postqueue -f           # flush: try delivering everything now
postcat -q <QUEUE_ID>  # view a specific queued message
postsuper -d <QUEUE_ID># delete one message
postsuper -d ALL       # delete ALL queued messages (caution!)
```

```mermaid
graph TD
    A["📋 Mail Log\n/var/log/mail.log"] --> B{status=?}
    B -->|"sent ✅"| OK["Message delivered\nNothing to do"]
    B -->|"deferred ⚠️"| DEF["Postfix retries automatically\nCheck deferral reason in log\npostqueue -p to inspect queue"]
    B -->|"bounced ❌"| BNC["NDR sent to sender\nPermanent error — must fix cause"]

    DEF --> GROW["Queue growing?"]
    GROW -->|"yes"| DIAG["Run diagnostics\nSee sections below"]
    GROW -->|"no"| WAIT["Wait for retries"]

    style OK fill:#4CAF50,color:#fff
    style DEF fill:#FF9800,color:#fff
    style BNC fill:#F44336,color:#fff
```

---

## Decoding Bounces and Status Codes

A **bounce** (NDR — Non-Delivery Report) is an automated message telling the *envelope*
sender that delivery failed.

```
550 5.1.1 <nobody@example.org>: Recipient address rejected: User unknown
└┬┘ └─┬─┘ └───────────────────── human-readable reason ──────────────────┘
 │    └ enhanced status code (class.subject.detail)
 └ classic 3-digit reply code
```

```mermaid
graph LR
    subgraph CODES["SMTP Reply Code Structure"]
        C4["4xx — TEMPORARY\nRetry automatically\n421, 451, 452"]
        C5["5xx — PERMANENT\nBounce immediately\n550, 554, 552"]
    end

    C4 --> R4["⏳ Message stays queued\nPostfix retries on schedule"]
    C5 --> R5["❌ Message bounced\nFix the cause — retrying won't help"]

    style C4 fill:#FF9800,color:#fff
    style C5 fill:#F44336,color:#fff
```

### Common Permanent Codes

| Code | Meaning | Action |
|------|---------|--------|
| `550 User unknown` | Recipient address doesn't exist | Fix the address; remove from list |
| `550 5.7.1 blocked` / `554` | IP or domain is blocked (reputation/blacklist) | Check DNSBLs (see §4) |
| `550 5.7.26 SPF/DKIM/DMARC` | Authentication failed | Fix your DNS records (see §3) |

---

## "My Mail Lands in Spam" Checklist

The #1 self-hosting complaint. Work through these **in order**:

```mermaid
flowchart TD
    START["🔍 Diagnosing: Mail in Spam"] --> SPF_CHK

    SPF_CHK{"☐ SPF passes\n& aligns?"}
    SPF_CHK -->|"❌ No"| SPF_FIX["Fix SPF record\ndig +short TXT yourdomain.com\nShould show v=spf1 ..."]
    SPF_CHK -->|"✅ Yes"| DKIM_CHK

    DKIM_CHK{"☐ DKIM passes?"}
    DKIM_CHK -->|"❌ No"| DKIM_FIX["Publish DKIM key in DNS\nCheck Rspamd is signing\ndig +short TXT dkim._domainkey.yourdomain.com"]
    DKIM_CHK -->|"✅ Yes"| DMARC_CHK

    DMARC_CHK{"☐ DMARC passes\n& aligns?"}
    DMARC_CHK -->|"❌ No"| DMARC_FIX["Ensure SPF/DKIM domain\nmatches From: domain"]
    DMARC_CHK -->|"✅ Yes"| PTR_CHK

    PTR_CHK{"☐ PTR matches\nmail hostname?"}
    PTR_CHK -->|"❌ No"| PTR_FIX["Set rDNS at hosting provider\ndig +short -x YOUR_IP"]
    PTR_CHK -->|"✅ Yes"| BL_CHK

    BL_CHK{"☐ IP not on\nblacklist?"}
    BL_CHK -->|"❌ Listed"| BL_FIX["See §4 — blacklist removal"]
    BL_CHK -->|"✅ Clean"| CONT_CHK

    CONT_CHK{"☐ Content clean?"}
    CONT_CHK -->|"❌ Spammy"| CONT_FIX["Avoid all-caps subjects\nNo link shorteners\nNo image-only bodies"]
    CONT_CHK -->|"✅ Yes"| WARM_CHK

    WARM_CHK{"☐ IP warmed up?"}
    WARM_CHK -->|"❌ New IP"| WARM_FIX["Ramp volume gradually\nStart with engaged recipients"]
    WARM_CHK -->|"✅ Yes"| DONE["🎉 Should be delivered\nto inbox"]

    style DONE fill:#4CAF50,color:#fff
    style START fill:#2196F3,color:#fff
```

**Fastest diagnostic:**
1. Send a message to **mail-tester.com** (gives a 0–10 score with specifics)
2. Open a Gmail copy → **"Show original"** → read the actual `Authentication-Results`

---

## Blacklists (DNSBLs)

A **DNSBL** (DNS-based blacklist) is a published list of IPs known for spam.
If your server's IP is listed, many receivers reject your mail outright.

```mermaid
graph TD
    LISTED["⚠️ IP listed on a DNSBL"] --> CAUSE["1. Find the root cause FIRST"]
    CAUSE --> C1["Compromised account\ndumping spam?"]
    CAUSE --> C2["Open relay\nbeing abused?"]
    CAUSE --> C3["Forwarding loop?"]
    CAUSE --> C4["Recycled IP from\nprevious owner?"]

    C1 --> FIX["2. Fix the root cause\n(before requesting removal)"]
    C2 --> FIX
    C3 --> FIX
    C4 --> FIX

    FIX --> DELIST["3. Request delisting\nvia each DNSBL's form"]
    DELIST --> WARMUP["4. Warm up again\n(reputation took a hit)"]

    style LISTED fill:#F44336,color:#fff
    style FIX fill:#FF9800,color:#fff
    style WARMUP fill:#4CAF50,color:#fff
```

**Check your IP:**

```bash
# Is 203.0.113.25 on Spamhaus zen? An A-record answer means "listed".
dig +short 25.113.0.203.zen.spamhaus.org
```

Online tools:
- **MXToolbox Blacklist Check** — https://mxtoolbox.com/blacklists.aspx
- **Spamhaus lookup** — https://check.spamhaus.org/

> **Prevention beats cure.** Don't be an open relay, rate-limit submissions,
> secure accounts with strong auth, and monitor your queue for sudden spikes.

---

## Deliverability Deep-Dive

Even with perfect config, *reputation* decides whether you reach the inbox:

```mermaid
mindmap
  root(("📊 Deliverability\nFactors"))
    🏆 IP & Domain Reputation
      Sending history
      Volume consistency
      Spam complaint rate
      Bounce rate
      Authentication results
    🌡️ IP Warm-up
      New IP = no reputation
      Start with low volume
      Ramp over days/weeks
      Engaged recipients first
    📢 Feedback Loops
      Register for FBLs
      Remove complainers immediately
      Complaints crater reputation
    🧹 List Hygiene
      Remove hard bounces promptly
      No purchased/stale lists
      High bounces = spammer signal
    📋 DMARC Reports
      XML aggregate reports
      Sent to rua= address
      Use dmarcian or Postmark
      Shows SPF/DKIM pass rates
```

### DMARC Tightening Timeline

```
Week 1:   p=none       ←  Monitor only, no effect on delivery
              ↓
              Read aggregate reports at rua= address
              ↓
Week 2+:  p=quarantine ←  Suspicious mail → spam folder
              ↓
              Continue monitoring, fix any misses
              ↓
Month 2+: p=reject     ←  Failed mail rejected outright
```

---

## Monitoring & Maintenance

Mail is a service that **breaks quietly**. Keep an eye on:

```mermaid
graph LR
    subgraph MONITOR["🔍 Monitoring Areas"]
        CERT["📜 TLS Certificates\nAlert before 90-day expiry\ncertbot renew --dry-run"]
        PORTS["🔌 Port Availability\n25, 587, 993 reachable\nExternal port check"]
        QUEUE["📦 Queue Size\nNot growing unbounded\npostqueue -p | tail"]
        DISK["💾 Disk Space\nMail store + logs\ndf -h; per-user quotas"]
        BL["🚫 Blacklist Status\nIP stays clean\nPeriodic MXToolbox check"]
        SEC["🔐 Security Updates\nOS + mail packages patched\nunattended-upgrades"]
        BAK["💼 Backups\nRestorable config + mail\n/etc/postfix, /etc/dovecot,\nDKIM keys, Maildirs"]
    end

    style MONITOR fill:#E3F2FD
```

| Area | What to Watch | How |
|------|--------------|-----|
| **Certificates** | TLS cert not expired | `certbot renew --dry-run`; alert before expiry |
| **Ports** | 25/587/993 reachable | External port check / uptime monitor |
| **Queue** | Not growing unbounded | `postqueue -p \| tail`; alert on size |
| **Disk** | Mail store + logs not full | `df -h`; per-user quotas in Dovecot |
| **Blacklists** | IP stays clean | Periodic MXToolbox/Spamhaus check |
| **Security** | OS + mail packages patched | `unattended-upgrades`; subscribe to advisories |
| **Backups** | Restorable config + mail | Back up `/etc/postfix`, `/etc/dovecot`, DKIM keys, Maildirs |

---

## Quick Reference: Symptom → Cause → Fix

```mermaid
graph TD
    subgraph SYMPTOMS["🚨 Common Symptoms"]
        S1["Mail to one domain deferred"]
        S2["ALL outbound bounce 5xx blocked"]
        S3["Delivered but lands in spam"]
        S4["550 User unknown"]
        S5["Can't send from mail app"]
        S6["Can't receive at all"]
        S7["Queue growing fast"]
        S8["TLS errors in client"]
    end
```

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Mail to one domain `deferred` | Greylisting / their rate limit | Wait; Postfix retries. Persistent → check their bounce text |
| **All** outbound bounce `5xx … blocked` | IP blacklisted | Check DNSBLs (§4); find root cause; request delisting |
| Mail delivered but lands in **spam** | SPF/DKIM/DMARC fail or unaligned; cold IP | Run the §3 checklist + mail-tester |
| `550 User unknown` on outbound | Recipient address wrong/dead | Fix the address; remove from list |
| Can't send from mail app | Submission/auth misconfigured | Use port 587/465 with auth; check `smtpd` logs |
| Can't receive at all | MX/PTR wrong, or port 25 blocked | Verify `dig MX`, PTR; confirm port 25 open inbound |
| Queue growing fast | Account hijacked / loop / downstream down | Inspect `postqueue -p`; lock the account; investigate |
| TLS errors in client | Expired/mismatched cert | `certbot renew`; reload Postfix + Dovecot |

---

### See Also

- [← Self-Hosting Guide](SELF_HOSTING.md) · [Protocols by Hand](PROTOCOLS.md) · [Email Anatomy](EMAIL_ANATOMY.md)
- **Next:** [Choosing Mail Server Software →](CHOOSING_SOFTWARE.md)

[← Back to index](../../README.md)
