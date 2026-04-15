# Deploying Spring Boot on Private EC2 — Learning Notes
> Java 21 + Spring Boot 3.x + nginx reverse proxy + systemd + S3 deployment

---

## The Mental Model

Before anything else, understand the full request flow:

```
Internet
    │
    ▼
NLB (port 80)
    │
    ▼
EC2 — nginx (port 80)          ← receives request, proxies internally
    │
    ▼
Spring Boot / Tomcat (port 8080) ← handles business logic
    │
    ▼
JSON response back up the chain
```

**Key insight:** nginx and Spring Boot serve different purposes. nginx is the traffic cop — handles connections, SSL, headers. Spring Boot is the application — handles business logic. Never expose Tomcat (8080) directly to the internet. Always put nginx in front.

---

## Core Concepts

### What is a JAR file?

When you build a Spring Boot app with Maven, it produces a **fat JAR** — a single file containing:
- Your compiled application code
- Every dependency (Spring Framework, Jackson, etc.)
- Embedded Tomcat server

This is why the JAR is ~22MB even for a tiny app. Everything needed to run is bundled inside. No separate server installation needed.

```
mvn clean package -DskipTests
    │
    ▼
target/your-app-0.0.1-SNAPSHOT.jar   ← deploy this file
```

### What is Tomcat?

Tomcat is a **web server and servlet container** embedded inside your Spring Boot JAR. Its job:
- Listens on port 8080
- Accepts HTTP connections
- Hands requests to your Spring Boot code
- Returns responses

In the old days Tomcat was a separate installation. Spring Boot embeds it so `java -jar app.jar` is all you need.

### What is a Reverse Proxy?

nginx sits in front of Spring Boot and forwards requests to it. This gives you:
- SSL termination at nginx (EC2 handles plain HTTP internally)
- nginx's performance for connection handling
- Spring Boot focused purely on application logic
- Port 8080 never exposed to the outside world

### What is systemd?

Linux's service manager. It manages background processes (services) that:
- Run without a terminal session
- Auto-restart on crash
- Auto-start on server reboot

The same system behind `systemctl start nginx`. You register your Spring Boot app the same way.

### What is the Instance Metadata Service (IMDS)?

A special internal AWS endpoint at `http://169.254.169.254` only accessible from within an EC2. When the AWS CLI runs on an EC2 with an IAM role attached, it automatically calls IMDS to get temporary credentials — no access keys or passwords needed.

```
aws s3 cp ...
    │
    ▼ "what credentials do I use?"
Instance Metadata Service (169.254.169.254)
    │ "here are temporary credentials from your attached IAM role"
    ▼
S3 — checks IAM role permissions → allowed
```

This is why IAM roles on EC2 are always preferred over hardcoded access keys. Keys can leak. Roles are automatic, temporary, and rotate themselves.

---

## Step-by-Step: Full Deployment

### Phase 1 — Create Spring Boot Project

1. Go to **[start.spring.io](https://start.spring.io)**
2. Settings:

| Field | Value |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | 3.4.x (latest stable) |
| Packaging | Jar |
| Java | 21 (LTS) |

3. Add dependencies:
   - `Spring Web` — REST endpoints + embedded Tomcat
   - `Spring Boot Actuator` — free `/actuator/health` endpoint (use for NLB health checks)

4. Generate and unzip

**Why Java 21?** It's the latest LTS (Long Term Support) version. Always pick LTS in production — supported with security patches for years. Non-LTS versions are dropped after 6 months.

---

### Phase 2 — Build the JAR

Open terminal in project root:

```bash
mvn clean package -DskipTests
```

- `clean` — deletes previous build output
- `package` — compiles and bundles everything into JAR
- `-DskipTests` — skips tests for faster builds

Output: `target/your-app-0.0.1-SNAPSHOT.jar`

**Always test locally first:**
```bash
# runs on port 8080 by default
java -jar target/your-app-0.0.1-SNAPSHOT.jar
```

Hit `http://localhost:8080` to confirm before touching EC2.

---

### Phase 3 — Transfer JAR to Private EC2 via S3

Private EC2 has no public IP — you can't SCP directly to it. Solution: upload to S3, EC2 pulls it down.

**Why this works:** EC2 has outbound internet via NAT Gateway + IAM role with S3 read access = can download from S3 without any credentials.

#### Step 1 — Create S3 bucket (AWS Console)
- S3 → Create Bucket
- Name: something unique e.g. `your-app-deployments-yourname`
- Region: same as your EC2
- **Block all public access: ON** — EC2 accesses via IAM, not public URL
- Everything else default

#### Step 2 — Upload JAR
- Open bucket → Upload → select your JAR → Upload

#### Step 3 — Add S3 permissions to EC2's IAM role
- IAM → Roles → your EC2 role (e.g. `ec2-ssm-role`)
- Add permissions → Attach policies → `AmazonS3ReadOnlyAccess`

#### Step 4 — Download on EC2 (via Session Manager)
```bash
aws s3 cp s3://your-bucket-name/your-app.jar /home/ssm-user/projects/app.jar
ls -lh /home/ssm-user/projects/app.jar    # verify ~22MB
```

---

### Phase 4 — Install Java on EC2

```bash
# Amazon Corretto = AWS's free production-ready Java distribution
sudo yum install -y java-21-amazon-corretto

# Verify
java -version
# Should show: openjdk version "21.x.x" ... Corretto
```

---

### Phase 5 — Register as systemd Service

Never run Spring Boot in the foreground (`java -jar` in terminal). Register it as a service so it survives terminal disconnects and reboots.

#### Create service file:
```bash
sudo nano /etc/systemd/system/your-app.service
```

```ini
[Unit]
Description=Your App Name
After=network.target

[Service]
Type=simple
User=ssm-user
ExecStart=java -jar /home/ssm-user/projects/app.jar
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**What each section does:**

`[Unit]`
- `After=network.target` — don't start until networking is ready. Critical for web servers.

`[Service]`
- `User=ssm-user` — run as this user, never as root
- `ExecStart` — exact command to start the app. Must be absolute path.
- `Restart=on-failure` — auto-restart if app crashes
- `RestartSec=10` — wait 10s before restarting (prevents rapid crash loops)

`[Install]`
- `WantedBy=multi-user.target` — start this service on system boot

#### Enable and start:
```bash
sudo systemctl daemon-reload          # reload systemd after creating new service file
sudo systemctl start your-app         # start now
sudo systemctl enable your-app        # auto-start on reboot
sudo systemctl status your-app        # verify: should show active (running)
```

#### View logs:
```bash
sudo journalctl -u your-app -n 50     # last 50 lines
sudo journalctl -u your-app -f        # follow live logs
```

#### Common systemctl commands:
```bash
sudo systemctl stop your-app
sudo systemctl restart your-app
sudo systemctl disable your-app       # remove auto-start
```

---

### Phase 6 — Configure nginx as Reverse Proxy

Edit nginx config:
```bash
sudo nano /etc/nginx/nginx.conf
```

Update the server block:
```nginx
server {
    listen       80;
    listen       [::]:80;
    server_name  _;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**What the headers do:**
- `Host` — tells Spring Boot the original hostname requested
- `X-Real-IP` — passes user's real IP (without this Spring Boot only sees localhost)
- `X-Forwarded-For` — standard header tracking original client IP through proxies

Without these headers, your Spring Boot app thinks every request comes from localhost — you lose all client IP information.

**Test and reload (always in this order):**
```bash
sudo nginx -t                     # test config for errors FIRST
sudo systemctl reload nginx       # reload without downtime
```

**Verify locally:**
```bash
curl http://localhost:80/
curl http://localhost:80/api/quotes
curl http://localhost:80/actuator/health
```

---

## nginx Location Matching — Important Detail

`location /` is a **catch-all** — it matches every URL since all paths start with `/`.

nginx picks the **most specific** match:

| Syntax | Matches | Priority |
|---|---|---|
| `location = /health` | only `/health` exactly | Highest |
| `location /api` | anything starting with `/api` | Medium |
| `location /` | everything (catch-all) | Lowest |

So `location /` proxying to Spring Boot means ALL requests go to Spring Boot — `/`, `/api/quotes`, `/api/status`, everything.

---

## NLB Health Check — Use Actuator

Spring Boot Actuator provides a free health endpoint:
```
GET /actuator/health
→ {"status":"UP"}
```

Configure your NLB Target Group health check to hit `/actuator/health` instead of `/`. This is more meaningful — it tells you the application is actually healthy, not just that nginx is running.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| `systemctl status` shows `exit-code` | Wrong path in ExecStart — verify with `ls -la /path/to/jar` |
| App starts then immediately stops | Check logs: `journalctl -u your-app -n 50` |
| curl localhost:8080 works but curl localhost:80 doesn't | nginx not reloaded or proxy_pass config wrong |
| NLB health check unhealthy | Is systemd service running? Is nginx running? Check both. |
| S3 download fails on EC2 | IAM role missing S3 policy or not attached to EC2 |

---

## Deployment Checklist (for every new version)

When you update your app and need to redeploy:

```bash
# 1. Build new JAR locally
mvn clean package -DskipTests

# 2. Upload to S3
# (via AWS Console or aws cli on laptop)

# 3. On EC2 — download new JAR
aws s3 cp s3://your-bucket/your-app.jar /home/ssm-user/projects/app.jar

# 4. Restart service
sudo systemctl restart your-app

# 5. Verify
sudo systemctl status your-app
curl http://localhost:8080/actuator/health
```

---

## What's Next

- **Route 53 + ACM** — custom domain + HTTPS on the NLB
- **Environment variables** — pass config to Spring Boot without hardcoding (`-Dspring.profiles.active=prod`)
- **CI/CD** — automate the build → S3 → EC2 deploy pipeline (GitHub Actions + AWS CodeDeploy)
- **CloudWatch Logs** — ship Spring Boot logs to CloudWatch for centralised monitoring
- **Multiple EC2s** — register two instances in the Target Group, experience real load balancing
- **RDS** — add a database, same VPC/subnet/Security Group concepts you already know
