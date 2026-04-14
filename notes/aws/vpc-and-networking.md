# AWS Infrastructure — Learning Notes
> Everything learned hands-on: EC2 → NLB → Private Subnet → NAT → Session Manager → nginx

---

## The Golden Mental Model

Before anything else, burn these three things into memory:

**1. AWS is deny by default.**
Nothing talks to anything until you explicitly allow it. Every timeout you hit is AWS working correctly — silently dropping traffic you haven't authorized yet.

**2. Security Groups are per-resource, not per-network.**
Every resource (EC2, NLB, RDS) gets its own Security Group. Creating a new resource means it starts with no rules — you must add them. This is the #1 cause of timeouts for beginners.

**3. Every hop in the traffic path needs its own permission.**
If traffic goes Internet → NLB → EC2, both the NLB and the EC2 need inbound rules allowing that traffic. Missing one hop = timeout.

---

## Core Concepts Cheat Sheet

### Security Groups
- Attached to a **specific resource**, not a subnet or VPC
- **Stateful** — if inbound is allowed, the response goes back automatically. No separate outbound rule needed for responses.
- Source can be an **IP range** (`0.0.0.0/0` = anywhere) or another **Security Group ID**
- Using a Security Group as source = "allow traffic from any resource that has this SG attached" — more flexible than IPs, survives infrastructure changes

### Subnets — Public vs Private

| | Public Subnet | Private Subnet |
|---|---|---|
| Route to Internet Gateway | Yes | No |
| Resources can get public IPs | Yes | No |
| Reachable from internet directly | Yes | No |
| Outbound internet access | Yes | Via NAT Gateway only |

### Internet Gateway vs NAT Gateway

| | Internet Gateway | NAT Gateway |
|---|---|---|
| Used by | Public subnets | Private subnets |
| Allows unsolicited inbound from internet | Yes | No |
| Allows outbound to internet | Yes | Yes |
| Resources get public IP | Yes | No (NAT's IP is used) |

**How NAT works:** Your private EC2 sends a request outbound. NAT Gateway (sitting in public subnet) forwards it to the internet using its own public IP. The internet responds to NAT, NAT translates it back to your private EC2. The internet never knows your private EC2 exists.

**Key distinction — inbound vs response traffic:**
NAT blocks *unsolicited* inbound (someone from internet trying to start a connection with your EC2). It allows *response* traffic (your EC2 started the conversation, response is just the continuation). Think of it like a phone call — you calling someone is different from someone calling you.

### Network ACLs (NACLs)
- Attached to a **subnet** (not a resource)
- **Stateless** — unlike Security Groups, you must explicitly allow both inbound AND outbound traffic
- Rules evaluated in **order by rule number** — first match wins
- Rule `*` (asterisk) = default deny — always at the bottom, catches everything not matched above
- Default NACL allows all traffic both ways — usually fine for learning

### Route Tables
- Every subnet has a route table controlling where traffic goes
- Public subnet route table has: `0.0.0.0/0 → Internet Gateway`
- Private subnet route table has: `0.0.0.0/0 → NAT Gateway`
- `local` route is always present — traffic within the VPC stays internal
- You can find a subnet's route table under: VPC → Subnets → your subnet → Route Table tab

---

## What Was Built

```
Internet
    │
    ▼
NLB (public subnet)          ← has Security Group, port 80 open to 0.0.0.0/0
    │
    ▼
Private EC2 (private subnet) ← has Security Group, port 80 open to NLB's SG only
    │
    ▼
nginx serving /var/www/optimus-prime
    │
    ├── index.html  (home page)
    └── 404.html    (error page)

Private subnet → NAT Gateway (public subnet) → Internet   [for outbound only e.g. yum installs]
```

---

## Step-by-Step: Everything Done

### Phase 1 — EC2 Public + Exposed Directly

1. Launch EC2 in a public subnet, auto-assign public IP enabled
2. Create Security Group: inbound port 80 from `0.0.0.0/0`
3. Run Python server on port 80
4. Access via EC2's public IP directly

**What you learned:** EC2 in a public subnet with an Internet Gateway route can be reached directly from the internet.

---

### Phase 2 — Add NLB in Front of EC2

**Concepts:** NLB sits in public subnets, receives traffic, forwards to Target Group (your EC2).

1. Create Target Group
   - Target type: Instances
   - Protocol: TCP, Port: 80
   - Register your EC2 as a target
   - Health check: TCP on port 80

2. Create NLB
   - Scheme: Internet-facing
   - Subnets: select public subnets across AZs
   - **Create/attach a Security Group** — port 80 open from `0.0.0.0/0`
   - Listener: TCP port 80 → forward to your Target Group

3. Wait for Target Group health check to show **Healthy**

4. Test via NLB DNS name

**The gotcha learned:** NLB starts with no Security Group. Port 80 must be explicitly opened on it — the EC2's Security Group alone is not enough. Both hops need their own rules.

**Debugging checklist when NLB DNS times out:**
- Is the NLB Security Group allowing port 80 inbound? ← most common miss
- Is the EC2 Security Group allowing port 80 from the NLB?
- Is the Target Group health status Healthy?
- Is something actually running on that port on the EC2?

---

### Phase 3 — Make EC2 Private

**Goal:** EC2 should only be reachable via NLB, not directly from the internet.

#### Step 1 — Create a private subnet
- VPC → Subnets → Create Subnet
- Do NOT enable auto-assign public IPv4
- Ensure its route table has NO route to an Internet Gateway

#### Step 2 — Check existing NAT Gateway
Before creating anything, check VPC → NAT Gateways. One may already exist.
- If healthy and in a public subnet → use it, don't create a duplicate
- **Lesson learned:** Always check what already exists before creating new resources. Duplicates cost money silently.

#### Step 3 — Add NAT Gateway route to private subnet's route table
- VPC → Subnets → your private subnet → Route Table tab → click route table ID
- Edit Routes → Add route: `0.0.0.0/0 → your NAT Gateway`
- This allows private EC2 to reach the internet outbound (for yum installs etc.)

#### Step 4 — Launch new EC2 in private subnet
- Network settings: pick your VPC + private subnet
- Auto-assign public IP: **Disabled**
- Attach a Security Group (see Step 5)

> Note: You cannot move an existing EC2 to a different subnet. Launch a new one.

#### Step 5 — Configure EC2 Security Group
- **Remove** any rule allowing port 80 from `0.0.0.0/0`
- **Add** inbound rule: TCP port 80, source = **NLB's Security Group ID**
- This means only the NLB can send traffic to the EC2. The internet cannot reach it directly.

#### Step 6 — Update Target Group
- EC2 → Target Groups → your TG → Targets tab
- Deregister old public EC2
- Register new private EC2 on port 80
- Wait for health status → **Healthy**

**If health check stays Unhealthy, check in order:**
1. Is something running on port 80 on the EC2? (most common)
2. Does the EC2 Security Group allow traffic from the NLB's SG?
3. Does the private subnet NACL allow traffic both inbound and outbound?

---

### Phase 4 — Access Private EC2 (Session Manager)

You cannot SSH into a private EC2 from the internet — there's no route in. Use **AWS Systems Manager Session Manager** instead.

**Why Session Manager:** No SSH, no open ports, no Bastion Host needed. AWS handles the secure tunnel.

#### Setup steps:

1. **Create IAM Role**
   - IAM → Roles → Create Role
   - Trusted entity: AWS Service → EC2
   - Attach policy: `AmazonSSMManagedInstanceCore`
   - Name it e.g. `ec2-ssm-role`

2. **Attach role to private EC2**
   - EC2 → your instance → Actions → Security → Modify IAM Role
   - Select `ec2-ssm-role` → Update

3. **Reboot the instance** — SSM agent needs a restart to pick up the new role

4. **Connect**
   - EC2 → your instance → Connect → Session Manager tab → Connect

**If Session Manager tab is greyed out:** Role isn't attached or SSM agent hasn't registered yet. Wait 2-3 minutes post-reboot.

---

### Phase 5 — Install and Configure nginx

#### Install nginx
```bash
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx    # auto-start on reboot — critical
sudo systemctl status nginx    # verify: should say active (running)
```

#### Proper directory structure
Don't use nginx's default `/usr/share/nginx/html`. Proper convention:
```bash
sudo mkdir -p /var/www/your-site-name
sudo chown -R nginx:nginx /var/www/your-site-name
```

#### Configure nginx to serve your directory
```bash
sudo nano /etc/nginx/nginx.conf
```

Find and update:
```nginx
root  /var/www/your-site-name;
```

The default config already handles 404 pages:
```nginx
error_page 404 /404.html;
```
Just create `/var/www/your-site-name/404.html` and it works automatically.

#### Test config and reload (no downtime)
```bash
sudo nginx -t                    # always test before reloading
sudo systemctl reload nginx      # reload, not restart — zero downtime
```

> Always run `nginx -t` before reloading. It catches config errors before they take down your server.

---

## Elastic IP — What It Is and When to Clean Up

An Elastic IP is a static public IP address assigned to you. NAT Gateway requires one so the internet always knows where to send return traffic back.

**Cost warning:** Elastic IPs are free when attached to a running resource. If you delete a NAT Gateway but forget to release its Elastic IP, AWS charges you for it sitting idle.

**Cleanup order when tearing down:**
1. Delete NAT Gateway
2. Wait for status → Deleted
3. VPC → Elastic IPs → Release the address

---

## Troubleshooting Checklist

When something times out or health checks fail, go through this in order:

| # | Check | Where to look |
|---|---|---|
| 1 | Security Group on the **receiving** resource — is the port open? Is the source correct? | EC2 / NLB → Security Groups |
| 2 | Is something actually **running** on that port? | SSH/Session Manager into EC2, check `systemctl status nginx` |
| 3 | Route Table — does the subnet have a route to where traffic needs to go? | VPC → Route Tables |
| 4 | NACL — are both inbound AND outbound rules allowing traffic? | VPC → Network ACLs |
| 5 | IAM Role — does the EC2 have the right permissions (e.g. for SSM)? | EC2 → IAM Role |

**Most issues are #1 or #2.** Start there every time.

---

## Cost Awareness — Things That Charge You Money

| Resource | Cost behaviour |
|---|---|
| NAT Gateway | Per hour + per GB processed |
| Elastic IP | Free if attached, charged if floating unattached |
| NLB | Per hour + per LCU used |
| EC2 | Per hour while running |
| Data transfer | Charged for cross-AZ traffic |

**Best practice:** When learning, tear down resources you're not using. Delete NAT Gateways and release Elastic IPs especially — they charge even when idle.

---

## What's Next

- **HTTPS on NLB** — add SSL certificate via AWS Certificate Manager (ACM), update NLB listener to port 443
- **Route 53** — set up a proper domain name instead of the ugly NLB DNS
- **Auto Scaling Groups** — EC2s that spin up/down automatically based on traffic
- **CloudWatch** — logs and metrics so you can see what's happening
- **IAM deep dive** — users, roles, policies, least privilege
- **S3** — object storage, most used AWS service
- **RDS** — managed databases, same networking concepts as EC2 (VPC, subnets, Security Groups)
