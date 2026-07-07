# Spam Troubleshooting — Why Emails Land in Spam

> **Problem:** Mail sent through your server is being placed in the spam folder by
> Gmail, Outlook, and other providers.
>
> This guide walks you through diagnosing the problem step-by-step and fixing it.

---

## Table of Contents

1. [Quick Diagnosis (5 Minutes)](#quick-diagnosis-5-minutes)
2. [Full Verification Checklist](#full-verification-checklist)
3. [DKIM Failures](#dkim-failures)
4. [SPF Failures](#spf-failures)
5. [PTR (Reverse DNS) Missing](#ptr-reverse-dns-missing)
6. [IP on a Blacklist](#ip-on-a-blacklist)
7. [Hostname Mismatch](#hostname-mismatch)
8. [TLS Certificate Problems](#tls-certificate-problems)
9. [Escaping Gmail's Spam Folder](#escaping-gmails-spam-folder)
10. [Escaping Outlook's Spam Folder](#escaping-outlooks-spam-folder)
11. [Deliverability Score Card](#deliverability-score-card)
12. [Monitoring Script](#monitoring-script)

---

## Quick Diagnosis (5 Minutes)

```mermaid
flowchart TD
    START["🔍 Mail landing in spam?"] --> STEP1

    STEP1["Step 1: Run mail-tester.com\n1. Go to mail-tester.com\n2. Send a test email to the address shown\n3. Click 'Then check your score'\n4. Target: 10/10 (minimum 8/10)"] --> SCORE{Score?}

    SCORE -->|"≥ 8/10 ✅"| STEP2
    SCORE -->|"< 8/10 ❌"| FIX_LIST["Work through the\nchecklist below"]

    STEP2["Step 2: Check Gmail headers\n1. Open email in Gmail\n2. Click ⋮ → 'Show original'\n3. Find Authentication-Results section"] --> CHECK_HEADERS

    CHECK_HEADERS["Look for:
    dkim=pass    ← ✅ or ❌ fail
    spf=pass     ← ✅ or ❌ fail
    dmarc=pass   ← ✅ or ❌ fail"]

    CHECK_HEADERS --> ALL_PASS{All pass?}
    ALL_PASS -->|"✅ Yes"| REPUTATION["Problem is IP reputation\nSee §9 — Gmail reputation tools"]
    ALL_PASS -->|"❌ No"| DNS_FIX["Fix the failing check\nSee sections below"]

    style START fill:#2196F3,color:#fff
    style REPUTATION fill:#FF9800,color:#fff
    style DNS_FIX fill:#F44336,color:#fff
```

---

## Full Verification Checklist

```bash
DOMAIN="yourdomain.com"
IP="203.0.113.25"

echo "--- MX ---"
dig MX $DOMAIN +short

echo "--- A (mail subdomain) ---"
dig A mail.$DOMAIN +short

echo "--- SPF ---"
dig TXT $DOMAIN +short | grep spf

echo "--- DKIM ---"
dig TXT dkim._domainkey.$DOMAIN +short

echo "--- DMARC ---"
dig TXT _dmarc.$DOMAIN +short

echo "--- PTR ---"
dig -x $IP +short
```

**Expected output:**
```
--- MX ---
10 mail.yourdomain.com.

--- A (mail subdomain) ---
203.0.113.25

--- SPF ---
"v=spf1 mx ip4:203.0.113.25 ~all"

--- DKIM ---
"v=DKIM1; k=rsa; t=s; s=email; p=MIIBIjAN..."

--- DMARC ---
"v=DMARC1; p=none; rua=mailto:admin@yourdomain.com"

--- PTR ---
mail.yourdomain.com.
```

### Blacklist Check

```bash
# Check manually against major blacklists
for bl in zen.spamhaus.org bl.spamcop.net b.barracudacentral.org; do
  result=$(dig +short $IP.$bl 2>/dev/null)
  if [ -n "$result" ]; then
    echo "❌ BLACKLISTED: $bl ($result)"
  else
    echo "✅ Clean: $bl"
  fi
done
```

Or online: https://mxtoolbox.com/blacklists.aspx

---

## DKIM Failures

```mermaid
graph TD
    FAIL["❌ dkim=fail"] --> CAUSE{What's the cause?}

    CAUSE --> C1["Cause 1:\nDKIM TXT record missing from DNS"]
    CAUSE --> C2["Cause 2:\nDKIM key format is wrong\n(truncated or missing quotes)"]
    CAUSE --> C3["Cause 3:\nRspamd is not signing outgoing mail"]

    C1 --> F1["Fix: Add TXT record to DNS\ndig TXT dkim._domainkey.yourdomain.com +short\n(should return key, not empty)"]
    C2 --> F2["Fix: Copy the FULL key from admin panel\nv=DKIM1; k=rsa; t=s; s=email; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."]
    C3 --> F3["Fix: Restart Rspamd and Postfix\ndocker compose restart rspamd-mailcow postfix-mailcow\n\nCheck logs:\ndocker compose logs rspamd-mailcow | grep -i dkim"]

    style FAIL fill:#F44336,color:#fff
    style F1 fill:#4CAF50,color:#fff
    style F2 fill:#4CAF50,color:#fff
    style F3 fill:#4CAF50,color:#fff
```

**Diagnosing Rspamd (for Mailcow):**

```bash
# Check Rspamd logs for DKIM activity
docker compose logs rspamd-mailcow | grep -i dkim

# Check Postfix logs for signing
docker compose logs postfix-mailcow | grep -i dkim

# Verify key status in admin panel
# Admin → System → Configuration → Options → ARC/DKIM keys
# Look for green "Key valid" badge next to your domain
```

---

## SPF Failures

```mermaid
graph TD
    FAIL["❌ spf=fail or spf=softfail"] --> CAUSE{What's the cause?}

    CAUSE --> C1["Wrong IP in SPF record"]
    CAUSE --> C2["No SPF record at all"]
    CAUSE --> C3["Multiple SPF records\n(only 1 allowed per domain)"]

    C1 --> F1["Fix: Check your real server IP\ncurl ifconfig.me\nUpdate SPF to match:\n@ TXT v=spf1 mx ip4:YOUR_REAL_IP ~all"]
    C2 --> F2["Fix: Add SPF to DNS\n@ TXT v=spf1 mx ip4:203.0.113.25 ~all"]
    C3 --> F3["Fix: Merge into ONE SPF record\n\n❌ Bad (two records):\nv=spf1 mx ~all\nv=spf1 ip4:203.0.113.25 ~all\n\n✅ Good (one record):\nv=spf1 mx ip4:203.0.113.25 ~all"]

    style FAIL fill:#F44336,color:#fff
    style F1 fill:#4CAF50,color:#fff
    style F2 fill:#4CAF50,color:#fff
    style F3 fill:#4CAF50,color:#fff
```

---

## PTR (Reverse DNS) Missing

```mermaid
graph LR
    A["📧 Your Server\n203.0.113.25"] -->|"PTR record"| B["mail.yourdomain.com"]
    B -->|"A record"| A

    NOTE["⚠️ PTR is NOT set in your DNS panel\nIt's set at your HOSTING PROVIDER's control panel"]

    style NOTE fill:#FF9800,color:#fff
```

**Check if PTR is set:**

```bash
dig -x 203.0.113.25 +short
# Should return: mail.yourdomain.com.
# If empty or wrong → PTR is not set
```

**How to set PTR (varies by provider):**

| Provider | Where to Set PTR |
|----------|-----------------|
| **Hetzner** | Cloud panel → Server → Networking → Reverse DNS |
| **DigitalOcean** | Droplet → Networking → rDNS |
| **Vultr** | Instances → IPv4 → Reverse DNS |
| **AWS** | Submit support request (requires Elastic IP) |
| **Generic** | Open a support ticket requesting: `203.0.113.25 → mail.yourdomain.com` |

---

## IP on a Blacklist

```mermaid
flowchart TD
    LISTED["⚠️ IP is blacklisted"] --> WHY{Find root cause first!}

    WHY --> W1["Compromised account\nsending spam?"]
    WHY --> W2["Open relay being abused?"]
    WHY --> W3["Previous IP owner\nleft bad history?"]

    W1 --> FIX["Fix the root cause\nbefore requesting delisting"]
    W2 --> FIX
    W3 --> FIX

    FIX --> DELIST["Request delisting\nfrom each blacklist"]
    DELIST --> BL1["Spamhaus: https://check.spamhaus.org/"]
    DELIST --> BL2["Barracuda: https://www.barracudacentral.org/lookups"]
    DELIST --> BL3["SpamCop: https://www.spamcop.net/"]

    DELIST --> WARMUP["After delisting:\nWarm up IP gradually\nreputation took a hit"]

    style LISTED fill:#F44336,color:#fff
    style WARMUP fill:#FF9800,color:#fff
```

> **Prevention:** Never be an open relay, rate-limit submissions, use strong authentication,
> and monitor your queue for sudden volume spikes (a hijacked account is the usual cause).

---

## Hostname Mismatch

**Problem:** Server hostname doesn't match the SMTP EHLO name.

```bash
# Check server hostname
hostname -f
# Should be: mail.yourdomain.com

# Check Postfix hostname (for Mailcow)
docker exec postfix-mailcow postconf myhostname
# Should be: mail.yourdomain.com
```

**Fix:**

```bash
# Set server hostname
hostnamectl set-hostname mail.yourdomain.com

# For Mailcow: check mailcow.conf
grep MAILCOW_HOSTNAME /opt/mailcow-dockerized/mailcow.conf
# Should be: MAILCOW_HOSTNAME=mail.yourdomain.com
```

---

## TLS Certificate Problems

```mermaid
graph LR
    TLS_ERR["🔐 TLS errors"] --> CHECK["Check certificate"]
    CHECK --> CMD["echo | openssl s_client\n-connect mail.yourdomain.com:443 2>/dev/null |\nopenssl x509 -noout -dates"]
    CMD --> EXPIRED{Certificate expired?}
    EXPIRED -->|"Yes"| RENEW["Renew certificate\n\nFor Mailcow:\ndocker compose exec acme-mailcow\n/srv/obtain-certificate.sh\n\ndocker compose restart nginx-mailcow"]
    EXPIRED -->|"No"| MISMATCH["Check certificate\nmatches mail hostname\nssl-checker.online-domain-tools.com"]

    style TLS_ERR fill:#F44336,color:#fff
    style RENEW fill:#4CAF50,color:#fff
```

```bash
# Check certificate expiry date
echo | openssl s_client -connect mail.yourdomain.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Check ACME logs (Mailcow)
docker compose logs acme-mailcow | tail -30

# Force certificate renewal (Mailcow)
cd /opt/mailcow-dockerized
docker compose exec acme-mailcow /srv/obtain-certificate.sh
docker compose restart nginx-mailcow
```

---

## Escaping Gmail's Spam Folder

```mermaid
graph TD
    A["📧 Gmail marking as spam"] --> B["1. Sign up for\nGoogle Postmaster Tools"]
    B --> C["postmaster.google.com\nAdd and verify your domain"]
    C --> D["Monitor these metrics:"]
    D --> D1["IP Reputation → aim for 'High'"]
    D --> D2["Domain Reputation → aim for 'High'"]
    D --> D3["Spam rate → keep below 0.10%"]
    D --> D4["Authentication → SPF/DKIM/DMARC pass"]

    E["⚡ Quick fix"] --> F["In Gmail spam folder:\nFind your email\nClick 'Not spam'"]
    F --> G["Sends positive signal\nto Google's algorithm"]

    H["📊 Monitor DMARC reports"] --> I["Reports sent to rua= address\nUse dmarcian.com or\ndmarc.postmarkapp.com to read them"]

    style A fill:#F44336,color:#fff
    style G fill:#4CAF50,color:#fff
```

### Google Postmaster Tools Setup

1. Go to https://postmaster.google.com
2. Sign in with a Google account
3. Click **"+ Domain"** → enter `yourdomain.com`
4. Verify domain ownership (via DNS TXT record)
5. Monitor **IP Reputation** and **Domain Reputation** dashboards

**Target:** `Good` or `High` reputation

---

## Escaping Outlook's Spam Folder

```mermaid
graph LR
    A["📧 Outlook/Hotmail\nmarking as spam"] --> B["Check Microsoft SNDS"]
    B --> C["sendersupport.olc.protection.outlook.com/snds/\nEnter your IP address"]
    C --> D{Status?}
    D -->|"Green ✅"| E["Good — problem is\ncontent or reputation"]
    D -->|"Yellow ⚠️"| F["Monitor and improve\nauthentication"]
    D -->|"Red ❌"| G["Request delisting:\nmicrosoft.com/en-us/\noffice/junk-email-reporting-program"]

    style E fill:#4CAF50,color:#fff
    style F fill:#FF9800,color:#fff
    style G fill:#F44336,color:#fff
```

---

## Deliverability Score Card

Use this table to track your configuration status:

| Check | Result | Status |
|---|---|---|
| **DKIM** | pass / fail | ☐ |
| **SPF** | pass / fail | ☐ |
| **DMARC** | pass / fail | ☐ |
| **PTR (reverse DNS)** | set / missing | ☐ |
| **Blacklists** | clean / listed | ☐ |
| **TLS grade** | A+ / B / C | ☐ |
| **mail-tester score** | x / 10 | ☐ |
| **PTR ↔ HELO match** | match / no | ☐ |

*All checks ✅ = mail should reliably reach the inbox*

---

## Monitoring Script

Save this script on your server and run it daily via cron:

```bash
#!/bin/bash
# /opt/mailcheck.sh
# Run daily: crontab -e
# 0 9 * * * /opt/mailcheck.sh | mail -s "Mail Server Status" admin@yourdomain.com

DOMAIN="yourdomain.com"
IP="203.0.113.25"

echo "=============================="
echo "Mail Server Status: $(date)"
echo "=============================="

# MX check
MX=$(dig MX $DOMAIN +short)
echo "MX: $MX"

# SPF check
SPF=$(dig TXT $DOMAIN +short | grep spf)
echo "SPF: $SPF"

# DKIM check
DKIM=$(dig TXT dkim._domainkey.$DOMAIN +short | head -c 50)
echo "DKIM: $DKIM..."

# PTR check
PTR=$(dig -x $IP +short)
echo "PTR: $PTR"

# Disk usage
DISK=$(df -h /opt/mailcow-dockerized 2>/dev/null || df -h /var/mail | tail -1 | awk '{print $5}')
echo "Disk: $DISK used"

# Check if any containers are stopped (Mailcow)
if command -v docker &>/dev/null; then
  STOPPED=$(docker compose -f /opt/mailcow-dockerized/docker-compose.yml ps --filter "status=exited" -q 2>/dev/null | wc -l)
  echo "Stopped containers: $STOPPED"
fi

# Blacklist check
echo "--- Blacklist Status ---"
for bl in zen.spamhaus.org bl.spamcop.net b.barracudacentral.org; do
  result=$(dig +short $(echo $IP | awk -F. '{print $4"."$3"."$2"."$1}').$bl 2>/dev/null)
  if [ -n "$result" ]; then
    echo "❌ BLACKLISTED: $bl"
  else
    echo "✅ Clean: $bl"
  fi
done

echo "=============================="
```

```bash
chmod +x /opt/mailcheck.sh
```

---

### See Also

- [← Mailcow Complete Guide](../mailcow/MAILCOW_GUIDE.md)
- [← DNS Quick Reference](DNS_REFERENCE.md)
- [Troubleshooting & Operations](TROUBLESHOOTING.md)

[← Back to index](../../README.md)
