# Route 53, ACM & HTTPS — Learning Notes
> Domain registration, DNS management, SSL certificates, and HTTPS on NLB — with real world context

---

## The Big Picture — Why All Three Work Together

Before any concept, understand what we're solving:

**The problem:**
Your site was accessible via an ugly AWS DNS name like `optimus-nlb-xxxx.ap-southeast-2.elb.amazonaws.com` over plain HTTP. Anyone intercepting traffic could read it. No professional product ships this way.

**The solution:**
```
https://najmushsaaquib.in
        │
        ▼ Route 53 — translates domain to NLB IP
NLB (54.79.192.189)
        │
        ▼ ACM certificate — proves identity, encrypts traffic
        │ TLSv1.3 encrypted
        │
        ▼ TLS terminated at NLB — decrypted here
        │ plain HTTP internally (VPC is trusted)
        │
        ▼ nginx (port 80) → Spring Boot (port 8080)
        │
        ▼ your API response
```

Three services, one seamless result:
- **Route 53** — translates your domain to the NLB
- **ACM** — proves you own the domain, issues and renews the certificate
- **NLB TLS listener** — terminates encryption, forwards plain HTTP to EC2

---

## Part 1 — HTTPS & TLS — How It Actually Works

### The Core Problem Encryption Solves

HTTP sends everything as plain text. Every packet travels through dozens of routers, ISPs, and networks. Anyone along that path can read or modify the data. This is a **man-in-the-middle attack**.

HTTPS solves this with **TLS (Transport Layer Security)** — the modern name for SSL.

### Asymmetric Encryption — The Breakthrough

You need two sides to communicate securely. But how do you share a secret key without someone intercepting it first? The answer is **asymmetric encryption** — two mathematically linked keys:

- **Public key** — share it with everyone freely
- **Private key** — never leaves your server, ever

The magic property:
> Anything encrypted with the public key can **only** be decrypted with the private key

```
Server generates: public key + private key
Browser receives: public key (given freely)

Browser encrypts message with public key
    │ (gibberish travelling over internet)
    ▼
Server decrypts with private key
    │
    ▼ readable message
```

Even if intercepted — the gibberish is useless without the private key.

### The Trust Problem — Certificates & Certificate Authorities

When a server hands your browser a public key, how does the browser know it belongs to the real `najmushsaaquib.in` and not an attacker pretending to be it?

This is solved by **Certificate Authorities (CAs)** — trusted third parties that vouch for domain ownership.

A certificate is a document saying:
> "I, Amazon (a trusted CA), verify this public key genuinely belongs to `najmushsaaquib.in`"

Your browser comes pre-installed with ~150 trusted CAs. If the certificate is signed by one of them → padlock appears.

**ACM (AWS Certificate Manager) is one of those trusted CAs** — which is why ACM certificates are trusted by all browsers automatically and for free.

### The TLS Handshake — Full Sequence

```
Browser                              Server (NLB)
   │                                      │
   │──── "Hello, I support TLS 1.3" ────▶│
   │                                      │
   │◀─── Certificate (public key) ────────│
   │                                      │
   │  Browser verifies:                   │
   │  ✅ Signed by trusted CA             │
   │  ✅ Domain matches                   │
   │  ✅ Not expired                      │
   │                                      │
   │──── Encrypted session key ──────────▶│
   │     (encrypted with public key)      │
   │                                      │
   │     Server decrypts with private key │
   │     Both sides now share session key │
   │                                      │
   │◀══════ Encrypted traffic ═══════════▶│
             (fast symmetric encryption)
```

**Why switch to symmetric encryption after the handshake?**
Asymmetric encryption is mathematically expensive. It's only used during the handshake to safely exchange a shared session key. After that, both sides use that session key with fast symmetric encryption for the actual data.

### TLS Termination — Why It Happens at the NLB

TLS termination means the NLB decrypts incoming HTTPS traffic and forwards plain HTTP to your EC2. The EC2 never deals with certificates or encryption.

**Why this is the industry standard:**
- Application code never deals with certificate management
- Rotating/renewing certificates happens at NLB — zero EC2 changes needed
- Decryption overhead stays at the edge, not on app servers
- Internal VPC traffic is already on a trusted private network — encrypting it adds CPU overhead for no real security benefit

```
Internet → HTTPS (encrypted) → NLB → HTTP (plain) → EC2
```

The EC2 only ever sees plain HTTP on port 80. It has no idea HTTPS was involved.

### HTTPS as a Business Decision — Not Just Technical

| Reason | Business Impact |
|---|---|
| Google ranking | HTTPS sites rank higher since 2014 — directly affects traffic |
| Browser warnings | Chrome shows "Not Secure" on HTTP — users leave, conversions drop |
| Legal compliance | GDPR, PCI-DSS, HIPAA all require encryption in transit |
| It's free | ACM + Let's Encrypt make certificates free — no cost argument against it |

**Real world:** Any HTTP-only production site today is negligence, not a cost decision.

---

## Part 2 — DNS & Route 53

### How DNS Actually Works

When you type `najmushsaaquib.in`, your browser needs the actual IP address. DNS is the system that translates domain names to IPs.

```
You type: najmushsaaquib.in
        │
        ▼ Your computer asks: "who knows about this domain?"
Global DNS root
        │
        ▼ "Ask the .in nameservers"
.in nameservers
        │
        ▼ "Ask Route 53 — they manage this domain"
Route 53 (your Hosted Zone)
        │
        ▼ "Here's the IP: 54.79.192.189"
Your browser connects to 54.79.192.189
```

**Why Route 53 is called Route 53:** DNS operates on port 53. Classic AWS naming.

### DNS Record Types — Know These

| Record | Purpose | Example |
|---|---|---|
| `A` | Domain → IP address | `najmushsaaquib.in → 54.79.192.189` |
| `CNAME` | Domain → another domain | `www.site.com → site.com` |
| `ALIAS` | Domain → AWS resource (Route 53 specific) | `najmushsaaquib.in → NLB DNS` |
| `NS` | Who answers DNS for this domain | `najmushsaaquib.in → ns-1163.awsdns-17.org` |
| `TXT` | Arbitrary text — used for verification | ACM validation, Google verification |
| `MX` | Mail servers | `najmushsaaquib.in → mail.google.com` |
| `SOA` | Start of Authority — zone admin info | Auto-managed, never touch it |

### Hosted Zone — The Core Route 53 Concept

A **Hosted Zone** is Route 53's way of saying "I will answer all DNS queries for this domain." It's where all your DNS records live.

When you point a domain's nameservers to Route 53, Route 53's Hosted Zone becomes the single source of truth for that domain's DNS.

**One Hosted Zone = one domain = all records in one place.**

Real world benefit: When your domain registrar, DNS, certificates, and infrastructure are all in AWS, everything integrates automatically. ACM validates via Route 53. NLB integrates with Route 53 ALIAS. No cross-provider confusion.

### NS Records — The Critical Handoff

NS (Nameserver) records tell the global DNS system who answers queries for your domain.

**Before our change:**
```
najmushsaaquib.in NS → Hostinger's nameservers
(Hostinger answered all DNS queries)
```

**After our change:**
```
najmushsaaquib.in NS → Route 53's nameservers
(Route 53 answers all DNS queries)
```

This change is made at your **domain registrar** (Hostinger) — not in Route 53. You're telling Hostinger "stop handling DNS, Route 53 does it now."

After this change, Hostinger's DNS management panel becomes irrelevant. All DNS changes happen in Route 53.

### ALIAS Record — Why Not CNAME for Root Domain

**The DNS standard (RFC 1034) rule:**
> A CNAME record cannot coexist with any other records on the same name

Your root domain `najmushsaaquib.in` must have NS and SOA records — mandatory for any domain. These already exist. Adding a CNAME would conflict with them — illegal by DNS standard.

```
najmushsaaquib.in  NS    → Route 53 nameservers (mandatory)
najmushsaaquib.in  SOA   → admin record (mandatory)
najmushsaaquib.in  CNAME → nlb-dns... ← ILLEGAL — conflicts with NS/SOA
```

`www.najmushsaaquib.in` works fine as CNAME because it has no mandatory NS/SOA records.

**AWS invented ALIAS to solve this:**
- Behaves like CNAME (points to another domain name, like the NLB DNS)
- Technically an A record underneath — doesn't conflict with NS/SOA
- Only works inside Route 53 Hosted Zones
- Free — ALIAS queries don't cost money, CNAME queries do

```
najmushsaaquib.in     A (ALIAS) → optimus-nlb-xxx.elb.amazonaws.com ✅
www.najmushsaaquib.in A (ALIAS) → optimus-nlb-xxx.elb.amazonaws.com ✅
```

### DNS Propagation — Why Changes Take Time

When you change DNS records, that information is cached by DNS servers worldwide. Each cache has a **TTL (Time To Live)** — a timer after which it discards the cache and re-fetches.

Until caches expire, different users may get different results depending on which DNS server they hit.

**Typical propagation times:**
- Best case: 15-30 minutes
- Typical: 2-4 hours
- Worst case: 48 hours (high TTL set previously)

**Real world lesson:** Never make DNS changes 5 minutes before a launch. Change DNS 24-48 hours in advance. Lower TTL days before a planned migration so caches clear faster.

**Famous failure:** In 2021, Facebook's entire DNS went down for 6 hours when an engineer accidentally removed their own nameserver records during a routine update. Even after fixing it, propagation delay meant it took time to restore worldwide.

**How to check propagation:**
```bash
# Check via Google's DNS directly (bypasses your ISP cache)
dig najmushsaaquib.in @8.8.8.8

# Check worldwide propagation
# https://dnschecker.org/#A/najmushsaaquib.in
```

### SOA Record — What It Is, Why You Never Touch It

SOA (Start of Authority) is automatically created by Route 53. Contains:
```
ns-1163.awsdns-17.org.        ← primary nameserver
awsdns-hostmaster.amazon.com.  ← DNS admin contact
1                              ← serial number (zone version)
7200                           ← refresh interval
900                            ← retry interval
1209600                        ← expiry
86400                          ← negative TTL
```

Route 53 manages this automatically. You will never need to edit it.

---

## Part 3 — ACM (AWS Certificate Manager)

### What ACM Does

ACM is AWS's Certificate Authority service. It:
- Issues SSL/TLS certificates for your domain — **free**
- **Auto-renews** before expiry (certificates expire every 13 months)
- Attaches directly to AWS services (NLB, ALB, CloudFront, API Gateway)
- Private key **never leaves AWS** — more secure than managing certificates yourself

**The one catch:** ACM certificates only work with AWS services. You cannot download them for use on non-AWS servers (unless you choose the paid "Enable export" option).

### Certificate Request Settings — Explained

**Domain names:**
Add both root and www variant in one certificate:
- `najmushsaaquib.in`
- `www.najmushsaaquib.in`

One certificate covers both. Users reaching either variant get the padlock.

**Allow export:**
| Option | Use case | Cost |
|---|---|---|
| Disable export | AWS-only infrastructure | Free |
| Enable export | Non-AWS servers need the certificate | Paid per domain |

Always disable export for AWS-native setups. Private key never leaving AWS is a security feature, not a limitation.

**Validation method:**
| Method | How it works | Renewal |
|---|---|---|
| DNS validation | Add CNAME record to your DNS | Automatic forever |
| Email validation | Click link in email sent to domain owner | Manual every 13 months |

**Always use DNS validation.** Email validation breaks silently — if the contact email changes, renewals fail, certificate expires, users see a scary security warning.

Real world failure: In 2020 a large bank's online portal went down because their certificate expired after email validation failed silently for months. DNS validation with CNAME prevents this entirely.

**Key algorithm:**
| Algorithm | Compatibility | Use case |
|---|---|---|
| RSA 2048 | Maximum — works everywhere | Default choice |
| ECDSA P 256 | Modern browsers only, faster | Performance-sensitive modern stacks |
| ECDSA P 384 | Modern browsers only | High security requirements |

RSA 2048 for most cases. ECDSA for when you need performance and control your client base.

### DNS Validation — How It Works

ACM gives you a CNAME record to add to your domain's DNS:

```
_f44bb4d59dd.najmushsaaquib.in  CNAME  _9ee25005564.validations.aws
```

ACM checks if this CNAME exists. If yes → you control the DNS → you own the domain → certificate issued.

**Why CNAME and not TXT:**
The CNAME points to `validations.aws` — an AWS-controlled domain. When ACM renews your certificate, it checks that CNAME again, finds it still pointing to AWS, auto-renews. No human involvement ever again.

**When Route 53 and ACM are in the same account:** ACM shows a "Create records in Route 53" button — it adds the CNAME automatically. One click, done.

### Certificate Lifecycle

```
Request certificate
        │
        ▼
ACM generates: public key + private key
Private key stays in AWS forever
        │
        ▼
ACM gives you: CNAME record to add to DNS
        │
        ▼
You add CNAME to Route 53
        │
        ▼
ACM verifies CNAME exists → issues certificate
Status: Issued ✅
        │
        ▼ (every ~11 months)
ACM checks CNAME still exists
Still there → auto-renews
Certificate never expires as long as CNAME stays in DNS
```

---

## Part 4 — NLB vs ALB — The Real Decision

### Layer 4 vs Layer 7

| | NLB (Network Load Balancer) | ALB (Application Load Balancer) |
|---|---|---|
| OSI Layer | 4 (TCP/UDP) | 7 (HTTP/HTTPS) |
| Understands HTTP | No — raw bytes only | Yes — reads URLs, headers, methods |
| HTTP → HTTPS redirect | No (must handle in nginx) | Yes — native, one click |
| URL-based routing | No | Yes (`/api/*` → service A, `/*` → service B) |
| Static IP | Yes (Elastic IP) | No (DNS name only) |
| Latency | Microseconds | Milliseconds (parses HTTP) |
| Protocol support | TCP, UDP, TLS | HTTP, HTTPS, gRPC |
| Certificate attachment | TLS listener | HTTPS listener |

### When to Choose NLB

- Raw TCP/UDP protocols (not HTTP) — crypto exchange order matching, custom game servers, MQTT
- Ultra-low latency requirements — high-frequency trading, real-time financial systems
- Static IP required — enterprise firewalls, compliance whitelisting
- Non-HTTP workloads — database proxies, custom binary protocols

**Real world:** Netflix uses NLB for internal service-to-service communication where latency is critical. ALB for public-facing HTTP APIs where routing flexibility matters.

### When to Choose ALB

- Standard web applications and REST APIs
- Microservices with path-based routing
- HTTP → HTTPS redirect needed at LB level
- WebSocket applications
- Host-based routing (multiple domains, one LB)

### Why We Stayed With NLB

> "This is what we have in prod" is a completely valid engineering reason.

Consistency with production outweighs theoretical elegance:
- Familiarity — your team already knows NLB deeply
- Debugging is easier when dev matches prod
- Switching LB types in prod carries risk
- Seniors can review and help more effectively

This is called **"boring technology"** — deliberately choosing proven, familiar tools. It's a feature, not a limitation.

### NLB TLS Listener — How It Works

NLB doesn't have an "HTTPS" protocol option — it's Layer 4. For encrypted traffic, use **TLS protocol**:

```
Listener: TLS:443
Certificate: ACM certificate (najmushsaaquib.in)
Target group: your EC2 on port 80
Security policy: ELBSecurityPolicy-TLS13-1-2 (auto-applied)
```

The security policy controls which TLS versions and cipher suites are accepted. TLS 1.3 is the latest and most secure — AWS defaults to a good policy automatically.

**Port 443 on EC2 Security Group: NOT needed.**
TLS terminates at the NLB. EC2 only receives plain HTTP on port 80. Opening 443 on EC2 would be wrong — nothing is listening there.

---

## Part 5 — Full Setup: Step by Step

### Step 1 — Buy a Domain (External Registrar)

If Route 53 domain registration is blocked (new accounts):
- Buy from Namecheap, GoDaddy, Hostinger etc.
- `.in`, `.link`, `.click`, `.xyz` are cheapest options
- You'll point it to Route 53 in the next step

### Step 2 — Create Route 53 Hosted Zone

1. Route 53 → Hosted Zones → Create Hosted Zone
2. Domain name: `yourdomain.in`
3. Type: Public Hosted Zone
4. Create

Route 53 automatically creates:
- NS record with 4 nameservers
- SOA record

Copy the 4 NS values — you'll need them next.

### Step 3 — Point Domain to Route 53 (External Registrar)

In your registrar (Hostinger, Namecheap etc.):
1. Find DNS / Nameservers settings
2. Switch to Custom nameservers
3. Enter all 4 Route 53 nameservers (without trailing dot):
```
ns-1163.awsdns-17.org
ns-1981.awsdns-55.co.uk
ns-311.awsdns-38.com
ns-552.awsdns-05.net
```

After this, your registrar's DNS panel becomes irrelevant. All DNS management moves to Route 53.

Verify propagation:
```bash
dig yourdomain.in @8.8.8.8
# Should show Route 53 nameservers in authority section
```

### Step 4 — Request ACM Certificate

1. ACM → Request certificate → Request public certificate
2. Add domain names:
   - `yourdomain.in`
   - `www.yourdomain.in`
3. Validation: DNS validation
4. Export: Disabled
5. Key algorithm: RSA 2048
6. Request

On the certificate page → click **"Create records in Route 53"**

ACM adds CNAME records to your Hosted Zone automatically. Certificate status changes from Pending → Issued within minutes.

### Step 5 — Add TLS Listener to NLB

1. EC2 → Load Balancers → your NLB → Listeners → Add listener
2. Protocol: TLS
3. Port: 443
4. Forward to: your target group
5. Certificate: select your ACM certificate
6. Save

### Step 6 — Create ALIAS Records in Route 53

Create record 1 (root domain):
- Record name: (blank)
- Type: A
- Alias: ON
- Route traffic to: Alias to Network Load Balancer
- Region: your NLB's region
- Load balancer: select your NLB

Create record 2 (www):
- Record name: `www`
- Type: A
- Alias: ON
- Same NLB target

### Step 7 — Verify Everything

```bash
# DNS resolves correctly
dig yourdomain.in @8.8.8.8

# HTTPS works with valid certificate
curl -v https://yourdomain.in

# Check certificate details in output:
# SSL certificate verify ok ✅
# subject: CN=yourdomain.in ✅
# issuer: Amazon ✅
```

---

## Troubleshooting Checklist

| Symptom | Check |
|---|---|
| `DNS_PROBE_POSSIBLE` in browser | DNS not propagated yet — wait or check dnschecker.org |
| `SSL certificate verify failed` | Certificate not issued yet or wrong domain in cert |
| Connection timeout on port 443 | NLB Security Group missing port 443 inbound rule |
| Certificate shows NLB domain not your domain | Wrong certificate attached to listener |
| Site works on NLB DNS but not custom domain | ALIAS records not created or wrong region selected |
| Certificate stuck at Pending validation | CNAME records not added to Route 53 — click "Create records in Route 53" |

**Quick debug commands:**
```bash
# Check DNS from Google's DNS (bypasses ISP cache)
dig yourdomain.in @8.8.8.8

# Check full TLS handshake details
curl -v https://yourdomain.in

# Check worldwide propagation
# https://dnschecker.org/#A/yourdomain.in
```

---

## What's Next

- **HTTP → HTTPS redirect** — configure nginx to redirect port 80 traffic to HTTPS
- **HSTS** — tell browsers to always use HTTPS, never HTTP
- **Multiple EC2s behind NLB** — real load balancing, Auto Scaling Groups
- **CloudFront** — CDN in front of your setup for global performance
- **ALB** — learn path-based routing, host-based routing for microservices
- **CI/CD pipeline** — automate build → S3 → EC2 deploy with GitHub Actions
