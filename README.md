# Inside a Live Tycoon 2FA Phishing Server: A Source-Level Analysis

**A technical teardown of an adversary-in-the-middle phishing-as-a-service deployment — from the SharePoint lure to the recovered origin, with three findings that extend the public record.**

> **Publication note (sanitized version).** All domain names in this article are reserved `.example` placeholders and do not resolve. The only network indicator included is the origin IP. Hashes, live domains, panel credentials, API tokens, victim-identifying data, and victim credentials are withheld. See *Scope and Handling* at the end.

---

## Introduction

In early September 2026, I came across a French-language phishing lure that looked, on the surface, like an ordinary business document. It was a PDF named *"Offre de prix Devis – Disponible pour consultation"* — a price quote — hosted on a legitimate Microsoft SharePoint tenant.

What followed was a full investigation into a live **adversary-in-the-middle (AiTM) phishing kit**: a commercial platform that doesn't just steal passwords, but intercepts the *session itself*, defeating multi-factor authentication by capturing the authenticated cookie after MFA has already been completed.

This article documents what I found — the kill chain, the command-and-control infrastructure, the server-side relay engine, and the operational weaknesses that let me profile the operators. It is written for defenders and threat researchers who want to understand how these platforms actually work under the hood.

> **Attribution note:** the panel identifies itself as **"Nebula Panel v3.0 by @SCGX02"**. The Tycoon 2FA label used here is a working attribution based on shared tradecraft; it is revisited in *Attribution* below.

---

## What This Threat Is

The kit belongs to a category called **AiTM phishing** — a more dangerous evolution of the classic credential-phishing page.

A traditional phishing kit presents a fake login page, collects whatever the victim types, and sends it to the attacker. It breaks the moment the victim has MFA enabled.

An AiTM kit is different. It runs a **reverse proxy** that sits between the victim and the real identity provider (Microsoft 365, Gmail, or a federated login). The victim talks to what they believe is the real sign-in page; in reality, the kit is relaying every request to the legitimate service and relaying the responses back. When the victim completes MFA, the kit captures the resulting **session cookie** — which grants access even if the password is changed later.

This is sold as a **Phishing-as-a-Service (PhaaS)** platform. The operators run the infrastructure; customers rent access and provide their own lures and targets. Based on multiple independent analyses and the panel's own branding, this deployment is a build of the platform family widely tracked as **Tycoon 2FA**.

---

## The Attack Chain

### Stage 1 — The lure: a trusted tenant, weaponized

The phishing email directed recipients to a genuine SharePoint link:

```
victimtenant-my.sharepoint.com/:b:/g/personal/victimuser_victimcompany_fr/...
```

This was not attacker infrastructure. It was a real Microsoft tenant hosting a PDF price quote. The document carried a plausible French business pretext and a fake sender signature.

The sophistication here is the *choice of hosting*. A link on `sharepoint.com` passes mail filters and human judgment far more easily than a freshly registered domain. The attacker abused the tenant's own API — the page embedded a `driveAccessToken` that exposed the document's download URL and drive metadata.

Inside the PDF, the `/URI` annotations pointed to the real destination, hidden between two legitimate decoy domains:

```
outlook-office-com-mail-sentitems.phish-op[.]example/lure/index.php?key=...
```

The subdomain `outlook-office-com-mail-sentitems` mimics Microsoft service URL anatomy while silently resolving to an attacker-registered domain.

### Stage 2 — The gate: polymorphism and a fake human check

The malicious link served a ~170 KB PHP page behind Cloudflare. This is the **bot gate** — and it is where the kit's sophistication first shows.

Each visit receives a *different* obfuscated payload: a hex blob paired with a random seed and password, decoded with a **xorshift PRNG**. The result is a fake "I'm human" checkbox that quietly collects behavioral signals — `human_score`, `interaction_time`, `mouse_moves`, `device_type`, and a honeypot field.

The gate is a discriminator, not just a CAPTCHA. Low "humanness" scores or a replayed token get redirected to Google. A submission with human-like values earns a `302` to a *rotating key*. Every visitor is treated as a bot until proven otherwise, and the key burns on use.

### Stage 3 — The middleware: a loader disguised as a page

The second key returned a small page titled `middleware`, obfuscated the same way. Decoded, it pulled three scripts: jQuery and Bootstrap from legitimate CDNs, and the real payload from a **Tencent Cloud Object Storage (COS)** bucket:

```
staging-bucket-XXXXXXXX.cos.na-siliconvalley.myqcloud.com/bootstrap.min.js
```

The filename is indistinguishable from the legitimate Bootstrap library. This is classic trusted-infrastructure abuse — hosting the final stage on a Chinese cloud object store, keeping the phishing server itself thin and the delivery chain clean.

### Stage 4 — The harvester: a pixel-faithful sign-in with a relay behind it

Inside that payload, a single base64 string revealed the heart of the operation:

```
aHR0cHM6Ly9jMi5waGlzaC1vcC5leGFtcGxlL2NhcHQucGhw
→ https://c2.phish-op[.]example/capt.php
```

The payload renders a Microsoft "Sign In → Enter Password" page that is pixel-faithful to the real thing — including the victim's organization branding, fetched dynamically from Microsoft's own `GetCredentialType` endpoint. It walks the victim through the full MFA flow and posts every interaction back to `capt.php`.

---

## Command-and-Control and Infrastructure

### The panel

The operator's control panel lived on the same host that served the exfil endpoint:

```
c2.phish-op[.]example/panel/
```

Its login screen was branded **"SCGX02 PANEL"**, and the dashboard header identified the build as **"Nebula Panel v3.0 by @SCGX02"**. This naming is significant — it identifies the kit's author/publisher handle and its panel version.

### The origin behind Cloudflare

All domains resolved only to shared Cloudflare edge IPs, with a wildcard certificate and a fresh registration date. Passive DNS revealed nothing — no SPF, TXT, or MX records leaked an origin.

The origin was recovered not from DNS, but from the panel's own configuration data. It contained the operator's deployment metadata in plaintext, including:

```
serverIp: 43.157.34.231
```

That address is a **Tencent Cloud / ACEVILLE PTE LTD** host in Singapore — the true origin sitting behind Cloudflare. The same configuration held the Cloudflare zone ID, an API token, and the panel password hash.

### The staging infrastructure

The final-stage JavaScript was hosted on **Tencent COS**, a separate hop from the phishing origin. This separation of concerns — lure on SharePoint, gate and relay on the attacker domain, payload on Tencent COS — is deliberate: each layer can be rotated or taken down independently.

### The subdomain factory

The most operationally interesting component is a script I'll call the **subdomain factory**. It accepted an `action` from the panel and, using a stored Cloudflare API token, performed the full deployment cycle automatically:

1. Create a new proxied A record via the Cloudflare DNS API.
2. Create a web directory and drop in attacker-supplied PHP (AES-encrypted in transit).
3. Write an Apache `VirtualHost` config for both `:80` and `:443`.
4. Reload `httpd` via a pre-seeded sudoers rule.

Distinct actions existed for `create`, `create_redirect`, `create_attach_domain`, and `create_target_svg`. This is **infrastructure automation as a service** — the mechanism behind the platform's fast-rotating domain ecosystem, and the reason random-prefix subdomains proliferated across the operator's domains.

---

## Inside the Relay Engine (Technical)

The core of the kit is `capt.php`, a single PHP file that performs live authentication handshakes against real identity providers. This is the AiTM relay in its purest form, and it is worth examining closely.

### Identity provider detection

The relay first determines *what kind* of account the victim is entering, via Microsoft's `getuserrealm.srf` endpoint:

| Type | Handling |
|------|----------|
| `live.com` | Routed to `login.live.com` (Outlook/Hotmail) |
| `adfs` | Routed to an ADFS federation flow |
| `okta` | Routed to an Okta identity engine flow |
| `secureserver` / `godaddy` | Routed to GoDaddy/SecureServer SSO |
| default | Routed to standard `login.microsoftonline.com` |

This breadth is notable. Most published AiTM kit analyses focus on Microsoft 365 and Gmail; this build handles **federated enterprise IdPs** as well, expanding the victim surface to organizations using third-party identity providers.

### The Microsoft 365 relay

For standard Microsoft accounts, the relay drives the full authentication flow server-side:

- `GetCredentialType` fetches the victim's tenant branding.
- The credential POST is relayed to `login.microsoftonline.com/common/login`.
- If MFA is required, `BeginAuth` / `EndAuth` / `ProcessAuth` relay the challenge in real time.
- `processKMSIOffice` handles the "Keep me signed in" post-auth handshake.

### Session-cookie weaponization

The ultimate goal is not the password — it's the session cookie. The relay extracts `ESTSAUTH`, `ESTSAUTHLIGHT`, and `MSAAUTH` cookies and converts them, via a `cookieToJavascript` function, into a ready-to-replay browser script that restores the authenticated session with correct `Secure`, `SameSite`, and domain attributes.

### Anti-detection: residential proxies and geolocation

The relay routes all identity-provider traffic through a **residential proxy pool**, with the proxy username derived from the victim's country via a geolocation lookup. This makes the authentication traffic appear to originate from the victim's own region — defeating the geolocation-based anomaly detection that Microsoft applies to suspicious sign-ins.

### Exfiltration

Stolen credentials and session cookies are fanned out through three channels simultaneously:

- **Panel:** appended to a flat-file result log on the C2.
- **Telegram:** delivered as a formatted alert with the session cookie attached as a document.
- **Email:** sent via PHPMailer to a configured inbox.

---

## A Security Observation: The Panel's Own Weaknesses

While examining the panel, I observed two operational-security failures worth documenting, because they are instructive for how these platforms can be identified and disrupted.

First, the panel authenticated with a **single password field** and **no username**, and it used a **predictable, low-entropy default credential**. There was no brute-force lockout, no rate limiting, and no CAPTCHA. This is a recurring theme in PhaaS operations: the infrastructure sold to customers is often secured more poorly than the infrastructure it attacks.

Second, the subdomain factory's `create_attach_domain` action accepted **arbitrary PHP code** (base64-encoded) and wrote it directly into the webroot without validation. This feature — intended to let the operator drop phishing pages onto new subdomains — is effectively a built-in arbitrary code execution primitive. It demonstrates that even the operational tooling of these platforms carries the same careless trust boundaries as their phishing pages.

I should note these observations are presented for defensive awareness. They illustrate how quickly a PhaaS deployment's own weaknesses can expose its full infrastructure to identification and takedown.

---

## What's New: Findings Beyond the Public Record

The Tycoon 2FA platform is well documented — Proofpoint (2024), Cybereason, Group-IB, ANY.RUN, and Microsoft's March 2026 disruption report all describe its CAPTCHA gates, obfuscation, and session-cookie theft. This investigation confirms those at source level. Three findings go further:

**1. A Kasada anti-bot bypass fused into federated logins.**

The relay contains a `solve_kasada()` function that calls a third-party solving service to defeat **Kasada's** proof-of-work challenge, then replays the resulting `x-kpsdk-ct`, `x-kpsdk-cd`, `x-kpsdk-h`, and `x-kpsdk-v` headers into the GoDaddy and SecureServer SSO login flows before forwarding the federated result onward to Microsoft.

This is the most novel finding: the attacker is not merely proxying MFA, but actively defeating a commercial anti-bot vendor to reach the federated Microsoft login path. To my knowledge, this specific integration has not been described in the published Tycoon 2FA literature.

**2. Source-level confirmation of the automated subdomain factory.**

Microsoft's takedown report described automated subdomain rotation but did not publish the mechanism. The factory script observed here is the concrete implementation: Cloudflare DNS API → directory creation → encrypted PHP drop → Apache vHost generation → `httpd` reload, all triggered remotely from the panel.

**3. The panel config as an origin-deanonymization vector.**

The recovered origin IP, Cloudflare zone ID, API token, and password hash were all sitting in plaintext inside the panel's configuration. For defenders, the practical lesson is to route hosting metadata recovered from a panel to the provider's abuse team. For investigators working with recovered artifacts, the configuration is often a faster path to the true origin than DNS enumeration.

---

## Attribution

The kit is **Tycoon 2FA**, a phishing-as-a-service platform active since August 2023. Microsoft Threat Intelligence tracks its developer/operator as **Storm-1747**, and the March 2026 Europol/Microsoft takedown publicly named **Saad Fridi** (Faisalabad, Pakistan) as its primary developer, operating under the "Tycoon Group" brand.

It is important to distinguish the *platform developer* from the *operator of this specific deployment*. The panel branding (`@SCGX02`, "Nebula Panel v3.0") does not match any published Tycoon 2FA alias, which suggests this instance was run by a **re-seller or sub-operator** running a re-skinned build of the kit — not the original developer. A live, re-branded deployment five months after the public takedown is consistent with that model. This is a financially-motivated criminal ecosystem, not a nation-state APT.

The branding observed here should be treated as a working attribution pending code-level comparison; it does not by itself establish authorship.

---

## Defensive Recommendations

**Prevent:**
- Enforce phishing-resistant MFA (FIDO2/WebAuthn passkeys, Windows Hello for Business) — AiTM kits are designed to defeat SMS, TOTP, and push-based MFA.
- Block newly registered, high-entropy, and typo-squatted domains at the mail gateway.

**Detect:**
- Hunt for sign-ins from residential proxy infrastructure (datacenter-to-residential anomalies, impossible travel, mismatched ASN/geolocation).
- Monitor for `getuserrealm.srf`, `GetCredentialType`, and SAS `BeginAuth`/`EndAuth` call patterns originating from non-Microsoft IPs — these are relay fingerprints.

> **Detection caveat:** `getuserrealm.srf` and `GetCredentialType` are also called by legitimate Microsoft clients. Alert on these endpoints only in combination with sign-in telemetry (ASN shift between authentication and resource access, token replay, new MFA registration), not as standalone signals.

**Respond:**
- On suspected AiTM compromise, reset the password *and* revoke all active sessions and tokens — a password reset alone does not kill a stolen session cookie.
- Review and remove any new MFA devices, authenticator registrations, or mailbox rules.

---

## Indicator of Compromise

- **True origin IP:** `43.157.34[.]231` (Tencent Cloud / ACEVILLE PTE LTD, Singapore)

Domains, URLs, file hashes, panel identifiers, and detection strings are withheld from this public version. They can be shared with vetted defenders on request.

---

## Scope and Handling

Domain names in this article are reserved `.example` placeholders and do not resolve. The origin IP is included because it is the actionable network indicator. No victim credentials, session cookies, or victim-identifying data are reproduced here.

The author is an independent security researcher. This article is based on artifacts and responses gathered during the investigation. The origin IP and hosting details should be reported to the hosting provider's abuse contact and to the relevant platform abuse teams before wider publication.

*If you are a defender and need the withheld indicators, contact the author through GitHub.*
