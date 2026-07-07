# Mail Servers — A Practical Documentation Set

A from-the-ground-up guide to **how email and mail servers actually work** — what the
pieces are, how to run one, how to read raw mail, how to debug it, and which software to
choose. Written as a learning reference, not a deployment script.

## Documentation

| # | Guide | What it covers |
|---|-------|----------------|
| 1 | [Overview](docs/general/OVERVIEW.md) | What a mail server is, how email travels, core components, protocols & ports, DNS records, security, self-hosting basics, glossary. **Start here.** |
| 2 | [Cloud Providers & Infrastructure](docs/general/CLOUD_PROVIDERS.md) | Why residential IPs fail, PTR records, Port 25, and a comparison of cloud providers (Hetzner, Vultr, etc.). |
| 3 | [Self-Hosting a Mail Server](docs/general/SELF_HOSTING.md) | Hands-on, step-by-step setup: Postfix + Dovecot + DNS + Let's Encrypt TLS + Rspamd DKIM/spam, with smoke tests. |
| 4 | [Anatomy of an Email](docs/general/EMAIL_ANATOMY.md) | The message format: envelope vs. headers, RFC 5322 structure, the `Received:` trail, MIME, encoding, an annotated raw email. |
| 5 | [Speaking the Protocols by Hand](docs/general/PROTOCOLS.md) | SMTP/IMAP/POP3 typed at the server, then the same flows in Python (`smtplib`, `imaplib`, `aiosmtpd`). |
| 6 | [Troubleshooting & Operations](docs/general/TROUBLESHOOTING.md) | Logs and the queue, decoding bounces, escaping the spam folder, blacklists, deliverability, monitoring & maintenance. |
| 7 | [Choosing Mail Server Software](docs/general/CHOOSING_SOFTWARE.md) | "Which mail server is best?" — all-in-one stacks vs. build-your-own vs. managed providers, with per-scenario recommendations. |
| 8 | [**Mailcow: To'liq Qo'llanma** 🇺🇿](docs/mailcow/MAILCOW_GUIDE.md) | Mailcow dockerized — o'rnatish, DNS, DKIM, SPF, DMARC, backup, troubleshooting va boshqalar. **O'zbek tilida.** |
| 9 | [**Nega aynan Hetzner?** 🇺🇿](docs/mailcow/HETZNER_RECOMMENDATION.md) | Nima uchun Mailcow uchun aynan Hetzner VPS tavsiya qilinadi? RAM, 25-port muammosi va PTR sozlashlari. |
| 10 | [**DNS Tezkor Qo'llanma** 🇺🇿](docs/general/DNS_REFERENCE.md) | DNS yozuvlar: MX, A, SPF, DKIM, DMARC, PTR — to'liq namuna va tekshirish buyruqlari. |
| 11 | [**Spam Muammosi** 🇺🇿](docs/general/SPAM_TROUBLESHOOTING.md) | Xatlar spam papkasiga tushishi — diagnostika, yechi mlar, Gmail/Outlook deliverability. |

## Suggested reading order

- **New to email?** Read [1 — Overview](docs/general/OVERVIEW.md) first; it defines every
  term the other guides assume.
- **Deciding what to run?** Jump to
  [6 — Choosing Software](docs/general/CHOOSING_SOFTWARE.md).
- **Setting up your own server?** Follow [2 — Self-Hosting](docs/general/SELF_HOSTING.md),
  keep [5 — Troubleshooting](docs/general/TROUBLESHOOTING.md) handy.
- **Building something that sends/reads mail?** See
  [3 — Anatomy](docs/general/EMAIL_ANATOMY.md) and
  [4 — Protocols by Hand](docs/general/PROTOCOLS.md).
- **Mailcow o'rnatish?** To'g'ridan-to'g'ri
  [7 — Mailcow Complete Guide](docs/mailcow/MAILCOW_GUIDE.md) ga o'ting.

> Everything here uses **illustrative** examples — adapt hostnames, addresses, paths,
> and versions to your own setup, and check the official
> [Postfix](https://www.postfix.org/documentation.html) /
> [Dovecot](https://doc.dovecot.org/) /
> [Mailcow](https://docs.mailcow.email/) docs for production use.
