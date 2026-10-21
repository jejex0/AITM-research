# Investigating an Unknown AiTM Phishing Kit: What Was Inside a Nebula2FA Deployment

**From a trusted SharePoint document to MFA interception, session-cookie theft, automated infrastructure deployment, and a surprisingly capable authentication relay.**

> **Publication note.** This GitHub version mirrors the final Medium article and retains additional source-level technical detail and defensive guidance. All domain names are reserved `.example` placeholders and do not resolve. The origin IP is the only network indicator included. Hashes, live domains, panel credentials, API tokens, and victim-identifying data are withheld.

> **Naming note.** *Nebula2FA* is a research designation for the deployment described here. It is not an established industry family name.

---

## Introduction

Phishing attacks are becoming increasingly difficult to investigate because the phishing page is often only the visible tip of the infrastructure. The real story can be hidden behind multiple layers: trusted cloud services → traffic filtering → obfuscated payloads → authentication relay → session theft → automated exfiltration.

In early September 2026, I encountered a French-language phishing campaign that initially looked like a normal business-document email. The lure was a PDF named *"Offre de prix Devis — Disponible pour consultation"*. The document was hosted on a legitimate Microsoft SharePoint tenant. What initially looked like a simple phishing attempt eventually led to a live **adversary-in-the-middle (AiTM)** phishing infrastructure.

But there was something unusual.

I could identify several characteristics commonly associated with modern phishing-as-a-service platforms, including techniques seen in Tycoon2FA. However, nowhere in the infrastructure I investigated was there an explicit reference to Tycoon2FA. There was no Tycoon2FA branding. No obvious family name. No clear source-code marker tying the deployment directly to the public Tycoon2FA implementation.

Instead, I found **"SCGX02 PANEL"** and **"Nebula Panel v3.0"**.

Because I could not establish a definitive public attribution, I refer to this particular deployment as **Nebula2FA** throughout this article. Nebula2FA is a research designation, not an established industry family name.

The goal of this article is not to claim that I discovered a completely new phishing ecosystem. It is to document what the infrastructure actually contained and explain why its relationship to known AiTM kits deserves further investigation.

## The Initial Infection Chain

```
Phishing Email
      ↓
Legitimate Microsoft SharePoint
      ↓
Malicious PDF
      ↓
Attacker-Controlled Domain
      ↓
Anti-Bot / Human Verification
      ↓
Obfuscated Middleware
      ↓
Cloud-Hosted JavaScript Payload
      ↓
AiTM Authentication Relay
      ↓
MFA Interception
      ↓
Session Cookie Theft
      ↓
Attacker Exfiltration
```

What made the investigation interesting was that every stage had a different purpose.

## What This Threat Is

AiTM phishing is a more dangerous evolution of the classic credential-phishing page.

A traditional phishing kit presents a fake login page, collects whatever the victim types, and sends it to the attacker. It breaks the moment the victim has MFA enabled.

An AiTM kit is different. It runs a **reverse proxy** that sits between the victim and the real identity provider (Microsoft 365, Gmail, or a federated login). The victim talks to what they believe is the real sign-in page; in reality, the kit is relaying every request to the legitimate service and relaying the responses back. When the victim completes MFA, the kit captures the resulting **session cookie** — which grants access even if the password is changed later.

This is sold as a **Phishing-as-a-Service (PhaaS)** platform. The operators run the infrastructure; customers rent access and provide their own lures and targets. The deployment analyzed here shares tradecraft with documented AiTM platforms; its exact lineage is discussed in *Attribution* below.

## The Attack Chain

### Stage 1 — The Trusted SharePoint Lure

The phishing email directed the victim to a genuine Microsoft SharePoint URL:

```
victimtenant-my.sharepoint.com/:b:/g/personal/victimuser_victimcompany_fr/...
```

The document was a PDF containing a plausible business quotation and French-language content. At first glance, there was nothing obviously malicious about the hosting domain. The document was hosted on legitimate Microsoft infrastructure. This is an important part of the attack.

Instead of immediately sending the victim to an obviously malicious domain, the attacker first used a trusted cloud service. The document's embedded URI annotations eventually pointed toward the attacker's infrastructure. One observed URL followed a pattern similar to:

```
outlook-office-com-mail-sentitems.phish-op[.]example/lure/index.php?key=...
```

The hostname itself attempted to imitate Microsoft's URL naming conventions. The real domain, however, was attacker-controlled.

The page also embedded a `driveAccessToken` that exposed the document's download URL and drive metadata.

#### Why SharePoint?

Because users are trained to trust familiar cloud services. A link containing `sharepoint.com` can appear much less suspicious than `random-new-domain.example`. The attacker was effectively borrowing Microsoft's reputation for the first stage of the attack.

### Stage 2 — The Gate

The malicious URL did not immediately display a Microsoft login page. Instead, it served a heavily obfuscated PHP-based page behind Cloudflare, approximately 170 KB. Its purpose was to determine whether the visitor looked like a real human. The decoded logic collected:

```
human_score
interaction_time
mouse_moves
device_type
hp_field
```

The page used a randomized payload-generation mechanism involving a seed, password, and xorshift-based pseudorandom number generation. Each visit received a different obfuscated payload, making static analysis harder.

The gate worked as a traffic filter. A visitor that failed the expected checks could be redirected elsewhere. A visitor that behaved like a real user received a temporary rotating key, which was then used to reach the next stage. The key burned on use.

#### Why does a phishing kit need this?

Because the attacker is not only trying to fool victims. They are also trying to avoid:

- Security scanners
- Sandboxes
- Automated crawlers
- Reputation engines
- Researchers

A phishing URL that appears harmless inside a sandbox does not necessarily mean the underlying infrastructure is harmless. The server may simply be deciding: "Are you a victim, or are you investigating me?"

### Stage 3 — The Middleware

The next stage returned a small page titled `middleware`, obfuscated in a similar way. After decoding, the page loaded several JavaScript dependencies. Some were legitimate: jQuery and Bootstrap. The interesting component was hosted separately on Tencent Cloud Object Storage:

```
staging-bucket-XXXXXXXX.cos.na-siliconvalley.myqcloud.com/bootstrap.min.js
```

The filename is indistinguishable from the legitimate Bootstrap library. The architecture became:

```
SharePoint
    ↓
Attacker Domain
    ↓
Middleware
    ↓
Tencent COS
    ↓
Final Payload
```

This separation makes infrastructure easier to rotate. The attacker can change the phishing server, the payload host, or individual components without rebuilding the entire operation.

### Stage 4 — The Login Page

Inside the final payload was a base64-encoded reference to the server-side capture endpoint. Decoded, it pointed to:

```
aHR0cHM6Ly9jMi5waGlzaC1vcC5leGFtcGxlL2NhcHQucGhw
→ https://c2.phish-op[.]example/capt.php
```

The payload rendered a Microsoft-style sign-in experience designed to look like the real Microsoft authentication flow rather than a simple static phishing form. One particularly interesting feature was organizational branding: the deployment used Microsoft's own authentication functionality to retrieve information about the victim's organization and make the login experience appear more authentic. This is where the attack moved from ordinary credential phishing into AiTM territory.

## Command-and-Control and Infrastructure

### The panel

The operator's control panel lived on the same host that served the exfil endpoint:

```
c2.phish-op[.]example/panel/
```

Its login screen was branded **"SCGX02 PANEL"**, and the dashboard header identified the build as **"Nebula Panel v3.0 by @SCGX02"**. This branding became an important fingerprint. It did not directly identify the platform as Tycoon2FA — that distinction matters.

### The origin behind Cloudflare

All domains resolved only to shared Cloudflare edge IPs, with a wildcard certificate and a fresh registration date. Passive DNS revealed nothing — no SPF, TXT, or MX records leaked an origin.

The origin was recovered not from DNS but from the panel's own configuration data, which contained the operator's deployment metadata in plaintext, including:

```
serverIp: 43.157.34.231
```

That address is a Tencent Cloud / ACEVILLE PTE LTD host in Singapore — the true origin sitting behind Cloudflare. The same configuration held the Cloudflare zone ID, an API token, and the panel password hash.

### The subdomain factory

One of the most interesting components was an infrastructure automation script. The panel exposed actions including:

```
create
create_redirect
create_attach_domain
create_target_svg
```

The deployment workflow could perform several operations automatically:

```
Cloudflare API
      ↓
Create DNS Record
      ↓
Create Web Directory
      ↓
Deploy PHP
      ↓
Generate Apache VirtualHost
      ↓
Reload httpd
```

The PHP content was transferred in encrypted form. The Apache configuration was generated automatically. A pre-existing sudo configuration allowed the web server to be reloaded. This is more than a phishing page. It is infrastructure automation.

#### Why infrastructure automation matters

Imagine an operator needs 100 phishing domains. Doing this manually would be tedious. An automated deployment pipeline changes the economics:

```
One Panel
   ↓
Create Domain
   ↓
Configure DNS
   ↓
Deploy Phishing Page
   ↓
Configure Web Server
   ↓
Go Live
```

The infrastructure becomes scalable. This helps explain how modern PhaaS operations can rotate domains and campaigns quickly. The phishing operation starts to look less like a collection of hacked websites and more like a software platform.

## Inside the Relay Engine

The core of the kit is `capt.php`, a single PHP file that performs live authentication handshakes against real identity providers. This is the AiTM relay in its purest form.

### Identity provider detection

The relay first determines what authentication system is associated with the account, using Microsoft's `getuserrealm.srf` endpoint. The observed logic supported multiple identity-provider paths:

| Type | Handling |
|---|---|
| `live.com` | Microsoft consumer authentication |
| `adfs` | ADFS federation |
| `okta` | Okta identity engine |
| `secureserver` / `godaddy` | GoDaddy/SecureServer SSO |
| default | Microsoft Online authentication |

This was one of the more interesting findings. The infrastructure wasn't narrowly focused on a single Microsoft login page. It contained logic for multiple enterprise authentication paths.

### Relaying Microsoft authentication

For Microsoft authentication, the relay handled several stages of the authentication process. The observed logic included:

- `GetCredentialType`
- Credential submission
- MFA-related authentication stages
- `BeginAuth`
- `EndAuth`
- `ProcessAuth`
- `processKMSIOffice`

The victim could therefore proceed through what appeared to be a legitimate authentication process. The phishing server remained in the middle.

### The password wasn't the ultimate target

The real objective was the authenticated session. The relay extracted cookies including:

```
ESTSAUTH
ESTSAUTHLIGHT
MSAAUTH
```

The deployment also contained functionality to convert captured session information into a browser-replay mechanism, via a `cookieToJavascript` function that restores the authenticated session with the correct `Secure`, `SameSite`, and domain attributes.

This is the key difference between ordinary phishing and AiTM phishing. The attacker does not necessarily need to defeat MFA directly. Instead, the attacker attempts to obtain the authenticated session after the victim has already completed MFA:

```
Correct password
        +
Successful MFA
        =
Compromised session
```

This is why phishing-resistant authentication mechanisms are so important.

### Residential proxies

The relay contained functionality for routing identity-provider traffic through residential proxy infrastructure. Proxy parameters were derived using the victim's geographic location to make the authentication traffic appear more consistent with the victim's region. This matters because many identity-security systems rely on signals such as IP reputation, ASN, geography, impossible travel, and sign-in location. Residential proxy infrastructure can make those signals more difficult to interpret.

### Exfiltration

Captured information was sent through multiple channels: panel, Telegram, and email.

- **Panel:** appended to a flat-file result log on the C2.
- **Telegram:** delivered as a formatted alert with the session cookie attached as a document.
- **Email:** sent via PHPMailer to a configured inbox.

## The Panel's Own Weaknesses

The panel's login form was tellingly simple: a **single password field, no username**, no rate limiting, and no CAPTCHA. Its default credential was predictable and low-entropy. This is a recurring theme in PhaaS operations: the infrastructure sold to customers is often secured more poorly than the infrastructure it attacks. The weakness is operationally significant because it is the same class of failure that exposes PhaaS infrastructure to identification and takedown.

The subdomain factory's `create_attach_domain` action accepted **arbitrary PHP code** (base64-encoded) and wrote it directly into the webroot without validation. This feature — intended to let the operator drop phishing pages onto new subdomains — is effectively a built-in arbitrary code execution primitive. It demonstrates that even the operational tooling of these platforms carries the same careless trust boundaries as their phishing pages.

These observations are presented for defensive awareness. They illustrate how quickly a PhaaS deployment's own weaknesses can expose its infrastructure.

## The Kasada Connection

One of the most unusual pieces of code was:

```
solve_kasada()
```

The function interacted with a third-party solving service to handle **Kasada** anti-bot challenges. The resulting challenge material was incorporated into authentication requests associated with GoDaddy/SecureServer flows before the authentication process continued. The relay replayed the `x-kpsdk-ct`, `x-kpsdk-cd`, `x-kpsdk-h`, and `x-kpsdk-v` headers.

This is significant because the kit was not merely bypassing traditional CAPTCHA mechanisms. It was attempting to defeat an additional commercial anti-bot layer within an authentication workflow. I did not find this exact integration described in the public Tycoon2FA research I reviewed. That does not prove that Nebula2FA is unrelated to Tycoon2FA. It does, however, make the relationship more interesting.

Possible explanations include:

- A customized Tycoon2FA build
- A private fork
- Shared code between different phishing kits
- A separate kit implementing similar techniques
- A newer or less-documented PhaaS platform

More code-level comparison would be required to distinguish these possibilities.

## Attribution: Why I Didn't Call It Tycoon2FA

During the investigation, I found multiple characteristics that resemble publicly documented Tycoon2FA behavior:

- Anti-bot gates
- Obfuscated payloads
- Microsoft authentication relaying
- MFA interception
- Session-cookie theft
- Telegram exfiltration
- Cloudflare infrastructure
- Automated subdomain provisioning

These similarities are significant, but they are not sufficient on their own to prove family attribution. I did not find an explicit Tycoon2FA identifier in the infrastructure I analyzed. Nor did I find a direct source-code marker conclusively linking the SCGX02 / Nebula Panel deployment to the publicly documented Tycoon2FA implementation.

There are therefore several possibilities:

```
Known Tycoon2FA
       ↓
Modified / Re-skinned Version
       ↓
Nebula Panel
```

or:

```
Separate AiTM Kit
       ↓
Copied / Reimplemented
Tycoon-like Techniques
       ↓
Nebula Panel
```

or:

```
Another PhaaS Family
       ↓
Shared Code / Shared Techniques
       ↓
Nebula Panel
```

At this stage, I consider "Tycoon2FA-like" a technical comparison, not a confirmed attribution. The branding observed here should be treated as a working attribution pending code-level comparison.

## What Makes Nebula2FA Interesting?

At a high level, the deployment combines several capabilities:

```
Trusted Cloud Lure
        +
Anti-Bot Filtering
        +
Polymorphic / Obfuscated Payload
        +
Multi-IdP Relay
        +
MFA Interception
        +
Session Cookie Theft
        +
Residential Proxy Support
        +
Multi-Channel Exfiltration
        +
Automated Infrastructure Provisioning
```

Individually, most of these techniques are not new. The interesting part is their combination. The infrastructure behaves like a complete phishing platform rather than a single phishing webpage.

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

## Indicator of Compromise

- **True origin IP:** `43.157.34[.]231` (Tencent Cloud / ACEVILLE PTE LTD, Singapore)

Domains, URLs, file hashes, panel identifiers, and detection strings are withheld from this public version. They can be shared with vetted defenders on request.

## Final Thoughts

What started as a simple business-document phishing email eventually turned into an investigation of a much more capable authentication platform. The complete chain looked like this:

```
French Business Lure
        ↓
Microsoft SharePoint
        ↓
Attacker Domain
        ↓
Human Verification Gate
        ↓
Obfuscated Middleware
        ↓
Tencent COS
        ↓
Microsoft-Style Login
        ↓
AiTM Relay
        ↓
MFA Completion
        ↓
Session Cookie Capture
        ↓
Multi-Channel Exfiltration
```

The investigation also revealed:

- Multi-identity-provider support
- Residential proxy integration
- Automated Cloudflare DNS provisioning
- Automated Apache configuration
- Session-cookie handling
- Telegram and email exfiltration
- Kasada anti-bot solving
- An exposed operational panel

Several of these characteristics overlap with known Tycoon2FA behavior. But attribution is where we need to be careful. I did not find enough evidence to conclusively call this Tycoon2FA. For now, Nebula2FA is a research designation for the observed deployment.

Whether it is a heavily modified Tycoon2FA build, a private fork, or a separate phishing-as-a-service platform remains an open question. That uncertainty is exactly what makes the investigation interesting. The larger lesson for defenders is straightforward:

**Don't stop at the phishing page.**

The phishing page tells you what the victim sees. The relay tells you what the attacker is trying to steal. The infrastructure tells you how the operation scales. And sometimes, the attacker's own infrastructure reveals more than the attacker intended.

## Scope and Handling

Domain names in this article are reserved `.example` placeholders and do not resolve. The origin IP is included because it is the actionable network indicator. No victim credentials, session cookies, or victim-identifying data are reproduced here.

The author is an independent security researcher. The origin IP and hosting details should be reported to the hosting provider's abuse contact and to the relevant platform abuse teams before wider publication.

*Also published on Medium: [Investigating an Unknown AiTM Phishing Kit — Here's What Was Inside](https://medium.com/@thebluesec/investigating-a-unknown-aitm-phishing-kit-heres-what-was-inside-dc849fbaf341).*
