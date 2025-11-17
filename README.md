# QuickNote Production Architecture Design Document

**Project:** Cloud-Based Note-Taking SaaS Platform  
**Version:** 1.0  
**Date:** November 2025  
**Prepared By:** DevOps Engineering Team  

---

## Executive Summary

This document presents a comprehensive production-grade infrastructure design for QuickNote, a cloud-based note-taking application. The architecture prioritizes **security, scalability, high availability, cost-efficiency, and maintainability** through industry best practices and AWS cloud services.

### Core Design Principles

1. **Defense in Depth Security**: Multiple layers of security controls from internet to database
2. **High Availability**: Multi-AZ deployment ensuring 99.99% uptime
3. **Fault Tolerance**: Automatic failover and recovery mechanisms
4. **Scalability**: Horizontal scaling capabilities across all tiers
5. **Observability**: Comprehensive logging, monitoring, and alerting
6. **Cost Optimization**: Efficient resource utilization and right-sizing

### Architecture Overview

The QuickNote platform consists of:
- **Frontend Layer**: React single-page application served via Nginx
- **Application Layer**: Node.js REST API servers
- **Data Layer**: PostgreSQL database and S3 object storage
- **Infrastructure**: AWS VPC with multi-tier network architecture

---

## Table of Contents

1. [Part 1: Infrastructure & Network Architecture](#part-1-infrastructure--network-architecture)
2. [Part 2: Server Design & Linux Administration](#part-2-server-design--linux-administration)
3. [Part 3: Application Workflow Design](#part-3-application-workflow-design)

---

# Part 1: Infrastructure & Network Architecture

## 1.1 Virtual Private Cloud (VPC) Design

### Purpose and Function

A Virtual Private Cloud (VPC) provides an isolated virtual network environment within AWS where we can launch and manage resources with complete control over IP addressing, subnetting, routing, and security.

### VPC Configuration

**CIDR Block**: `10.0.0.0/16`

**Rationale for /16 Prefix**:
- Provides 65,536 total IP addresses
- Allows extensive subnetting across multiple availability zones
- Enables future growth without IP exhaustion
- Standard practice for medium to large production environments

**VPC Features Enabled**:
- DNS Resolution and DNS Hostnames
- Default tenancy (shared hardware for cost efficiency)

### Region Selection

**Primary Region**: us-east-1 (N. Virginia)

**Selection Criteria**:
- Largest AWS region with most service availability
- Lower latency for US-based users
- Extensive availability zone options (6+ AZs)
- Competitive pricing and mature infrastructure

---

## 1.2 Availability Zone Strategy

### Multi-AZ Deployment Architecture

**Deployed Across**: 2 Availability Zones
- **Availability Zone A**: us-east-1a
- **Availability Zone B**: us-east-1b

### Why Multiple Availability Zones Improve Reliability and Uptime

#### 1. Physical Isolation and Fault Tolerance

Availability Zones are physically separate data centers with independent power, cooling, and network infrastructure located miles apart. Failure in one AZ does not affect resources in other AZs.

#### 2. Improved Uptime and Service Level Agreements

**Single-AZ**: 99.9% uptime (8.76 hours downtime/year)  
**Multi-AZ**: 99.99% uptime (52.56 minutes downtime/year)

Multi-AZ deployment reduces downtime by 90%.

#### 3. Automatic Failover Capabilities

- Application Load Balancer automatically routes traffic away from unhealthy AZ
- RDS Multi-AZ provides automatic database failover in < 60 seconds
- Auto Scaling Groups launch replacement instances in healthy AZ

#### 4. Zero-Downtime Maintenance

During maintenance, update instances in one AZ while the other handles traffic, achieving zero service interruption.

---

## 1.3 Subnet Architecture and Network Segmentation

![Alt text description](images/QuickNote_AWS_Architecture_Diagram.drawio.png)

### Complete Subnet Layout

#### Public Subnets (Internet-Facing)

| Subnet Name | AZ | CIDR Block | Usable IPs | Purpose |
|-------------|-----|------------|------------|---------|
| Public-AZ-A | us-east-1a | 10.0.1.0/24 | 251 | ALB, NAT Gateway, Bastion |
| Public-AZ-B | us-east-1b | 10.0.2.0/24 | 251 | ALB, NAT Gateway |

#### Private Web Tier Subnets

| Subnet Name | AZ | CIDR Block | Usable IPs | Purpose |
|-------------|-----|------------|------------|---------|
| Private-Web-AZ-A | us-east-1a | 10.0.11.0/24 | 251 | Nginx web servers |
| Private-Web-AZ-B | us-east-1b | 10.0.12.0/24 | 251 | Nginx web servers |

#### Private Application Tier Subnets

| Subnet Name | AZ | CIDR Block | Usable IPs | Purpose |
|-------------|-----|------------|------------|---------|
| Private-App-AZ-A | us-east-1a | 10.0.21.0/24 | 251 | Node.js servers |
| Private-App-AZ-B | us-east-1b | 10.0.22.0/24 | 251 | Node.js servers |

#### Private Database Tier Subnets

| Subnet Name | AZ | CIDR Block | Usable IPs | Purpose |
|-------------|-----|------------|------------|---------|
| Private-DB-AZ-A | us-east-1a | 10.0.31.0/24 | 251 | PostgreSQL primary |
| Private-DB-AZ-B | us-east-1b | 10.0.32.0/24 | 251 | PostgreSQL standby |

### IP Address Calculation - Detailed Working

**Question**: In a subnet with CIDR block 10.0.1.0/24, how many usable IP addresses are available?

**Step 1**: CIDR /24 means first 24 bits are network portion, remaining 8 bits for hosts.

**Step 2**: Calculate total addresses = 2^8 = 256 addresses (10.0.1.0 to 10.0.1.255)

**Step 3**: AWS reserves 5 IP addresses:
1. **10.0.1.0** - Network address
2. **10.0.1.1** - AWS VPC router
3. **10.0.1.2** - AWS DNS server
4. **10.0.1.3** - Reserved for future use
5. **10.0.1.255** - Broadcast address

**Step 4**: Usable addresses = 256 - 5 = **251 usable IPs**

### Which Resources in Public Subnets and Why

**1. Application Load Balancer**
- Must accept incoming HTTPS traffic from internet users
- Requires public IP for DNS resolution
- Entry point with DDoS protection

**2. NAT Gateway**
- Provides outbound internet for private instances
- Requires public IP (Elastic IP) for NAT
- Does NOT allow inbound connections

**3. Bastion Host**
- Single controlled SSH entry point
- Reachable from authorized administrator IPs only
- Heavily hardened with MFA and key authentication

### Which Resources in Private Subnets and Why

**1. Nginx Web Servers (Private)**
- Not directly accessible from internet
- Load Balancer provides public interface
- Reduces attack surface significantly

**2. Application Servers (Private - CRITICAL)**
- Contains business logic and data processing
- Direct exposure would allow API abuse
- Must never be reachable from internet
- Attack must compromise multiple layers

**3. Database (Private - MAXIMUM SECURITY)**
- Contains sensitive user data
- Most valuable target for attackers
- Regulatory compliance requires isolation (PCI-DSS, HIPAA, GDPR)
- 3 layers between internet and database
- No outbound internet access

---

## 1.4 Internet Gateway

### Purpose and Function

The Internet Gateway (IGW) enables bidirectional internet connectivity for VPC resources.

**Outbound Traffic**: Instances in public subnets can initiate connections to internet  
**Inbound Traffic**: Internet users can reach public-facing resources

### How It Works

**Outbound**: IGW translates private IP to public IP (1:1 NAT)  
**Inbound**: Routes traffic to correct subnet based on destination public IP

**Key Characteristics**:
- Fully managed by AWS (no configuration needed)
- Redundant across multiple AZs
- Unlimited bandwidth
- Free (only data transfer charges)

---

## 1.5 NAT Gateway

### Purpose and Function

NAT Gateway enables private subnet instances to initiate outbound internet connections while preventing inbound connections from internet.

**Why NAT Gateway Instead of Public IPs**:
1. **Security**: Private instances never exposed to internet
2. **Flexibility**: Can update/replace instances without IP changes
3. **Compliance**: Meets security requirements for data isolation
4. **Cost**: One NAT Gateway serves multiple instances

**Deployment**: One NAT Gateway per AZ for high availability

---

## 1.6 Route Tables

### Public Subnet Route Table

| Destination | Target | Purpose |
|------------|--------|---------|
| 10.0.0.0/16 | local | Internal VPC traffic |
| 0.0.0.0/0 | igw-xxxxx | Internet traffic |

### Private Subnet Route Table

| Destination | Target | Purpose |
|------------|--------|---------|
| 10.0.0.0/16 | local | Internal VPC traffic |
| 0.0.0.0/0 | nat-xxxxx | Outbound internet via NAT |

---

## 1.7 Security Groups - Complete Defense Model

### Security Group 1: ALB (SG-1)

**Inbound**:
- Port 443 from 0.0.0.0/0 (HTTPS from internet)
- Port 80 from 0.0.0.0/0 (HTTP redirect to HTTPS)

**Outbound**:
- Port 80 to SG-2 (forward to web tier)

### Security Group 2: Web Tier (SG-2)

**Inbound**:
- Port 80 from SG-1 (traffic from ALB)
- Port 22 from SG-5 (SSH from Bastion)

**Outbound**:
- Port 3000 to SG-3 (API requests to app tier)
- Port 443 to 0.0.0.0/0 (software updates)

### Security Group 3: Application Tier (SG-3)

**Inbound**:
- Port 3000 from SG-2 (API requests from web tier)
- Port 22 from SG-5 (SSH from Bastion)

**Outbound**:
- Port 5432 to SG-4 (database queries)
- Port 443 to 0.0.0.0/0 (external APIs, S3)

### Security Group 4: Database (SG-4)

**Inbound**:
- Port 5432 from SG-3 ONLY (app tier queries)

**Outbound**:
- DENY ALL (no outbound access)

### Security Group 5: Bastion Host (SG-5)

**Inbound**:
- Port 22 from Corporate IP/32 (SSH from admin network)

**Outbound**:
- Port 22 to SG-2 and SG-3 (SSH to private instances)

---

## 1.8 Load Balancer Architecture

### Application Load Balancer Configuration

**Type**: Application Load Balancer (Layer 7)  
**Placement**: Public subnets in both AZs  
**Listeners**: Port 443 (HTTPS), Port 80 (HTTP → redirect to 443)

**Features**:
- TLS/SSL termination
- Path-based routing
- Health checks every 30 seconds
- Connection draining 300 seconds
- Cross-zone load balancing enabled

**Target Groups**:
- Web tier instances in both AZs
- Health check endpoint: GET /health (expect 200)

---

## 1.9 Bastion Host (Jump Box) Architecture

### Why Bastion Host is Required

**Without Bastion**:
- Each server needs public IP
- Multiple attack surfaces
- Difficult to audit access
- No centralized access control

**With Bastion**:
- Single SSH entry point
- Centralized logging and auditing
- MFA enforcement
- IP whitelisting
- Session recording

### Risks of Direct SSH Exposure

1. **Brute Force Attacks**: Continuous password attempts
2. **Vulnerability Exploitation**: SSH service vulnerabilities
3. **No Audit Trail**: Can't track who accessed what
4. **Compliance Violations**: Fails security audits

### Bastion Security Configuration

- Minimal installed software
- SSH key authentication only (no passwords)
- MFA required
- Session logging enabled
- IP whitelist (corporate IPs only)
- Regular security updates
- No application code or data

---

## 1.10 Complete Traffic Flow Analysis

### User Request Journey (18 Steps)

**Step 1: User Browser Initiates Request**
- User clicks "Save Note" in React app
- HTTPS POST to https://api.quicknote.app/api/notes
- JWT token in Authorization header
- JSON body with note data

**Step 2: DNS Resolution**
- Route 53 returns ALB public IP: 52.1.2.100
- TTL: 60 seconds

**Step 3: TLS Handshake**
- Browser establishes secure connection
- ALB presents certificate
- Symmetric encryption key negotiated

**Step 4: Internet Gateway (Inbound)**
- Packet arrives at AWS edge location
- Routed to us-east-1 IGW
- Forwarded to VPC

**Step 5: ALB Processing**
- Security Group SG-1 validates source
- ALB terminates TLS (decrypts)
- Inspects HTTP request
- Extracts headers and body

**Step 6: Load Balancer Routing**
- Checks health of all targets
- Selects nginx-az-a-2 (fewest connections)
- Routes to 10.0.11.11:80

**Step 7: ALB to Nginx**
- Creates HTTP request (unencrypted internal)
- Adds X-Forwarded-For: client IP
- Security Group SG-2 validates

**Step 8: Nginx Processing**
- Applies rate limiting
- Validates request method
- Matches location /api/

**Step 9: Nginx to App Server**
- Forwards to backend pool
- Round-robin selection
- Sends to 10.0.21.10:3000
- Security Group SG-3 validates

**Step 10: Application Processing**
- Middleware chain executes
- JWT validation
- Input sanitization
- Business logic

**Step 11: App to Database**
- Parameterized SQL query
- Opens connection from pool
- Security Group SG-4 validates
- Encrypted TLS connection

**Step 12: Database Processing**
- pg_hba.conf validates source IP
- Authenticates with scram-sha-256
- Executes INSERT statement
- Returns new note with ID
- Query time: ~5ms

**Step 13-18: Response Journey**
- Database → App: Result set
- App: Constructs JSON response
- App → Nginx: HTTP response
- Nginx → ALB: Forwards response
- ALB: Re-encrypts with TLS
- IGW: NAT translation
- Internet: Routes to client
- Browser: Decrypts and displays

### Port Usage Summary

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| User | ALB | 443 | HTTPS | Secure web traffic |
| ALB | Nginx | 80 | HTTP | Internal forwarding |
| Nginx | Node.js | 3000 | HTTP | API requests |
| Node.js | PostgreSQL | 5432 | PostgreSQL | Database queries |
| Node.js | S3 | 443 | HTTPS | File uploads |
| Private Instances | NAT | 443/80 | HTTPS/HTTP | Updates |
| Admin | Bastion | 22 | SSH | Admin access |
| Bastion | Servers | 22 | SSH | Management |

---

# Part 2: Server Design & Linux Administration

## 2.1 Web Server (Nginx) Design and Configuration

### Role in Architecture

Nginx serves multiple critical functions:

1. **Static Content Delivery**: Serves React frontend files (HTML, CSS, JavaScript)
2. **Reverse Proxy**: Forwards API requests to Node.js servers
3. **Load Distribution**: Distributes requests across multiple app servers
4. **Caching Layer**: Caches frequently requested content
5. **Security Functions**: Rate limiting, request filtering, security headers

### How Nginx Serves Frontend Static Files

**Process**:
1. Request arrives: GET /index.html
2. Matches location block: location /
3. Looks for file in root: /var/www/quicknote/frontend/index.html
4. Reads from disk (or cache)
5. Sends using sendfile() (zero-copy)

**SPA Handling**: `try_files $uri $uri/ /index.html` - serves index.html for all unknown routes

### How Nginx Forwards Backend API Traffic

**Configuration**:
- Upstream definition: pool of backend servers
- Location block: matches /api/ requests
- Proxy pass to upstream with load balancing
- Adds X-Real-IP, X-Forwarded-For headers

**Load Balancing**: Round-robin, least connections, or IP hash

### Health Check Implementation

**Passive Health Checks**:
- max_fails=3 (mark unhealthy after 3 failures)
- fail_timeout=30s (wait 30s before retry)
- Monitors actual client requests

### Nginx Logging Strategy

**Access Log** (`/var/log/nginx/access.log`):
- Records every HTTP request
- Includes timestamp, IP, method, URL, status, bytes

**Error Log** (`/var/log/nginx/error.log`):
- Records Nginx errors and warnings

**Log Rotation**:
- Daily rotation at midnight
- Retention: 14 days access, 30 days error
- Gzip compression after 1 day
- Automated by logrotate

### Performance Tuning Parameters

1. **worker_processes auto**: Match CPU cores
2. **worker_connections 2048**: Max connections per worker
3. **use epoll**: Efficient event notification (Linux)
4. **sendfile on**: Zero-copy file transfer
5. **tcp_nopush on**: Send headers + file in single packet
6. **tcp_nodelay on**: Disable Nagle's algorithm (lower latency)
7. **keepalive_timeout 65**: Reuse connections
8. **gzip on / level 6**: Compress responses (60-80% reduction)
9. **proxy_buffering on**: Buffer backend responses

---

## 2.2 Application Server (Node.js) Design

### Why Process Manager is Essential

**Problems Without Process Manager**:

1. **No Auto-Restart**: Crash = complete downtime
2. **Single Core Usage**: Can't utilize multi-core CPUs
3. **No Monitoring**: Can't tell if app is running
4. **Memory Leaks**: Indefinite growth until OOM
5. **No Log Management**: Stdout disappears
6. **Deployment Downtime**: Must stop/start manually
7. **No Auto-Start**: Doesn't start on server boot

### What PM2 Provides

1. **Automatic Restart**: Immediate restart on crash
2. **Cluster Mode**: Multiple instances (one per CPU core)
3. **Zero-Downtime Reload**: Graceful reload without interruption
4. **Process Monitoring**: CPU, memory, requests, errors
5. **Log Management**: Captures and rotates logs
6. **Memory Management**: Restarts if threshold exceeded
7. **Startup Scripts**: Integrates with systemd

### Application Lifecycle with PM2

**1. Start**: PM2 spawns N instances, connects to DB, health check available

**2. Run**: Instances handle requests, PM2 load balances

**3. Monitor**: Tracks CPU, memory, uptime, restart count

**4. Reload (Zero-Downtime)**:
- Starts new instance
- Waits for ready state
- Sends SIGINT to old instance
- Old finishes in-flight requests
- Old exits gracefully
- Repeat for each instance (30-60s total, no downtime)

**5. Stop**: Graceful shutdown, close connections, exit cleanly

### Why Applications Must Not Run as Root

**Security Risks**:

1. **Complete Compromise**: Vulnerability = root access
2. **Accidental Damage**: Bug could destroy system
3. **Privilege Escalation**: Minor vulnerability becomes critical
4. **Lateral Movement**: Can attack other systems
5. **Compliance Violations**: PCI-DSS, HIPAA prohibit root apps

### Proper User Permission Structure

**Create Dedicated User**:
- Username: `quicknote`
- Home: `/opt/quicknote`
- Shell: `/bin/false` (no login)
- Permissions: Read/execute app code, read/write logs only

**File Ownership**:
```
/opt/quicknote - owned by quicknote:quicknote (755)
/var/log/quicknote - owned by quicknote:quicknote (755)
```

### Environment Variables in Production

**Secure Management**:
- Store in `/opt/quicknote/.env` (mode 600)
- Never commit to Git
- Use AWS Secrets Manager for sensitive data
- Load via EnvironmentFile in systemd

**Required Variables**:
- NODE_ENV=production
- PORT=3000
- DATABASE_URL=postgresql://...
- JWT_SECRET=...
- AWS_ACCESS_KEY_ID=...
- AWS_SECRET_ACCESS_KEY=...

---

## 2.3 Database Server (PostgreSQL) Configuration

### Why Database Must Only Accept Internal Requests

**Critical Security Requirements**:
1. Contains most sensitive data
2. Direct exposure allows SQL injection
3. Brute force password attacks possible
4. Compliance requirement (PCI-DSS, HIPAA)
5. Data exfiltration risk

**Implementation**:
- Security Group SG-4 allows only SG-3 (app tier)
- No public IP assignment
- No Internet Gateway route
- Isolated in private DB subnet

### Purpose of pg_hba.conf

**pg_hba.conf** (PostgreSQL Host-Based Authentication) controls:
- Which hosts can connect
- Which databases they can access
- Which users are allowed
- Authentication method required

**Example Configuration**:
```
# TYPE  DATABASE        USER            ADDRESS          METHOD
host    quicknote_prod  quicknote_app   10.0.21.0/24    scram-sha-256
host    quicknote_prod  quicknote_app   10.0.22.0/24    scram-sha-256
```

### Best Authentication Mode for Production

**scram-sha-256** (recommended):
- Strong password hashing
- Resistant to credential replay
- Industry standard
- Better than md5 or password

**Never Use**:
- trust (no password)
- password (clear text)
- md5 (weak hashing)

### Backup and Restore Plan

**Backup Strategy**:

**1. Logical Backups (Daily)**:
- Tool: pg_dump
- Frequency: Daily at 2:00 AM
- Format: Custom compressed
- Storage: S3 with versioning
- Retention: 30 days
- Encryption: AES-256

**2. Physical Backups (Weekly)**:
- Tool: pg_basebackup
- Frequency: Sunday at 3:00 AM
- Type: Full database cluster
- Storage: S3 Glacier
- Retention: 90 days

**3. Continuous WAL Archiving**:
- Stream to S3 every 60 seconds
- Point-in-time recovery capability
- Retention: 7 days

**Backup Testing Schedule**:

**Monthly Full Restore Test** (First Sunday):
- Restore to isolated environment
- Verify data integrity
- Measure restore time
- Document issues
- Update procedures

**Quarterly DR Drill**:
- Simulate complete failure
- Rebuild in different region
- Validate all data
- Test application functionality
- Verify RTO and RPO met

**Automated Verification (Daily)**:
- Script verifies backup exists
- Checks file size consistency
- Attempts to read backup
- Verifies encryption
- Alerts on failure

**Retention and Pruning**:
- Daily backups: 30 days
- Weekly backups: 90 days
- WAL archives: 7 days
- Automated cleanup via cron

### Database Performance and Health Monitoring

**Key Metrics**:

**1. Connection Metrics**:
- Active connections
- Connection utilization (warn > 70%, critical > 90%)
- Failed connection attempts

**2. Query Performance**:
- Average query time (warn > 100ms, critical > 5s)
- Slow queries (> 1 second)
- Query rate (queries/second)
- Long-running queries (> 30 seconds)

**3. Database Size**:
- Total database size
- Table and index sizes
- Growth rate
- Alert at 80% capacity

**4. Cache Hit Ratio**:
- Buffer cache hit ratio (target > 99%)
- Alert if < 95%

**5. Replication Lag** (Multi-AZ):
- Lag time (warn > 5s, critical > 30s)
- Target: < 1 second

**6. Locks and Blocking**:
- Lock waits
- Deadlocks
- Blocking queries

**Monitoring Tools**:
- Built-in: pg_stat_* views
- AWS RDS: CloudWatch + Performance Insights
- Third-party: DataDog, New Relic, Prometheus+Grafana

---

## 2.4 Linux Server Administration Best Practices

### User Account Strategy - Principle of Least Privilege

**Core Principle**: Each user and process has only minimum required permissions.

**System Service Accounts** (No Login):

- **quicknote** user: Runs Node.js (home: /opt/quicknote, shell: /bin/false)
- **nginx** user: Runs web server (home: /var/www, shell: /bin/false)

**Administrative Accounts**:

- **sysadmin**: Full sudo access, MFA required
- **deploy**: Limited sudo (restart services only)

**Access Control Matrix**:

| User | SSH Login | Sudo | File Access | Use Case |
|------|-----------|------|-------------|----------|
| quicknote | No | None | /opt/quicknote, /var/log/quicknote | Run app |
| nginx | No | None | /var/www, /var/log/nginx | Run web |
| sysadmin | Yes (bastion) | Full | All | Admin |
| deploy | Yes (bastion) | Limited | /opt/quicknote | Deploy |
| root | No (disabled) | N/A | All | NEVER USED |

### sudoers Configuration

**Deploy User** (`/etc/sudoers`):
```
deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart quicknote
deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx
deploy ALL=(ALL) NOPASSWD: /bin/systemctl reload nginx
```

**System Admin**:
```
sysadmin ALL=(ALL) ALL
Defaults requiretty
Defaults env_reset
```

### SSH Access Configuration

**SSH Hardening** (`/etc/ssh/sshd_config`):
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers sysadmin deploy
MaxAuthTries 3
ClientAliveInterval 300
```

**Why These Settings**:
- Root login disabled: Even with compromised password
- Password auth disabled: Immune to brute force
- User whitelist: Only specified users
- Timeouts: Prevent abandoned sessions

---

## 2.5 Service Management with systemd

### What is systemd

Modern Linux init system that:
- Starts services on boot
- Manages service lifecycle
- Handles dependencies
- Provides resource control
- Integrates logging

### systemd Unit File for QuickNote

**Location**: `/etc/systemd/system/quicknote.service`

```ini
[Unit]
Description=QuickNote Node.js Application
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=forking
User=quicknote
Group=quicknote
WorkingDirectory=/opt/quicknote/app

Environment="NODE_ENV=production"
Environment="PORT=3000"
EnvironmentFile=/opt/quicknote/.env

ExecStart=/usr/local/bin/pm2 start ecosystem.config.js --env production
ExecReload=/usr/local/bin/pm2 reload all
ExecStop=/usr/local/bin/pm2 stop all

Restart=on-failure
RestartSec=10s
StartLimitBurst=5
StartLimitInterval=60s

LimitNOFILE=65536
MemoryLimit=2G

[Install]
WantedBy=multi-user.target
```

### systemctl Commands

```bash
# Enable (start on boot)
sudo systemctl enable quicknote

# Start service
sudo systemctl start quicknote

# Stop service
sudo systemctl stop quicknote

# Restart
sudo systemctl restart quicknote

# Reload (no downtime)
sudo systemctl reload quicknote

# Check status
sudo systemctl status quicknote

# View logs
sudo journalctl -u quicknote -f
```

---

## 2.6 Critical Log Files and Locations

### Operating System Logs

**1. System Log** (`/var/log/syslog` or `/var/log/messages`):
- Kernel messages
- System events
- Hardware errors
- Startup/shutdown events

**2. Authentication Log** (`/var/log/auth.log`):
- SSH login attempts
- sudo usage
- User authentication
- Security events

**3. Kernel Log** (`/var/log/kern.log`):
- Kernel errors
- Hardware issues
- Driver problems

### Application Logs

**Nginx**:
- Access: `/var/log/nginx/access.log`
- Error: `/var/log/nginx/error.log`

**QuickNote Application**:
- Application: `/var/log/quicknote/app.log`
- Error: `/var/log/quicknote/error.log`
- PM2: `/home/quicknote/.pm2/logs/`

**PostgreSQL**:
- PostgreSQL: `/var/log/postgresql/postgresql-14-main.log`
- Query log: Configured in postgresql.conf

### Log Management Best Practices

1. Centralized logging (CloudWatch, ELK)
2. Retention policy (30-90 days)
3. Log rotation (daily)
4. Monitoring and alerting
5. Regular log review

---

## 2.7 System Health Monitoring and Metrics

### 1. CPU Monitoring

**Metrics**:
- CPU utilization percentage
- Load average (1, 5, 15 minutes)
- Per-core utilization

**Commands**:
```bash
top
htop
mpstat -P ALL
```

**Thresholds**:
- Warning: CPU > 70%
- Critical: CPU > 90%
- Load average > number of cores

### 2. Memory Monitoring

**Metrics**:
- Total memory
- Used memory
- Available memory
- Swap usage

**Commands**:
```bash
free -h
vmstat 1
```

**Thresholds**:
- Warning: Memory > 80%
- Critical: Memory > 90%
- Swap usage > 10% (investigate)

### 3. Disk Space Monitoring

**Metrics**:
- Disk usage percentage
- Inode usage
- Disk I/O

**Commands**:
```bash
df -h
df -i
iostat -x 1
```

**Thresholds**:
- Warning: Disk > 80%
- Critical: Disk > 90%
- Alert: Disk > 95%

### 4. Network Performance

**Metrics**:
- Network throughput
- Packet loss
- Connection states
- Error rates

**Commands**:
```bash
netstat -tuln
ss -s
ifconfig eth0
```

**Monitoring Tools**:
- CloudWatch Metrics
- Prometheus + Grafana
- DataDog
- New Relic

---

## 2.8 Automated Task Scheduling with Cron

### What is Cron

Time-based job scheduler for automated tasks:
- Backups
- Log cleanup
- Health checks
- Database maintenance
- Certificate renewal

### Cron Syntax

```
* * * * * command
│ │ │ │ │
│ │ │ │ └─── Day of week (0-7)
│ │ │ └───── Month (1-12)
│ │ └─────── Day of month (1-31)
│ └───────── Hour (0-23)
└─────────── Minute (0-59)
```

**Examples**:
- `0 2 * * *` - Daily at 2:00 AM
- `*/5 * * * *` - Every 5 minutes
- `0 3 * * 0` - Every Sunday at 3:00 AM

### QuickNote Production Cron Jobs

**File**: `/etc/cron.d/quicknote`

```bash
# Database Backup - Daily at 2:00 AM
0 2 * * * postgres /opt/scripts/backup_logical.sh >> /var/log/cron_backup.log 2>&1

# Physical Backup - Weekly Sunday at 3:00 AM
0 3 * * 0 postgres /opt/scripts/backup_physical.sh >> /var/log/cron_backup.log 2>&1

# Log Cleanup - Daily at 3:00 AM
0 3 * * * root /opt/scripts/cleanup_logs.sh >> /var/log/cron_cleanup.log 2>&1

# Health Monitoring - Every 5 minutes
*/5 * * * * root /opt/scripts/health_check.sh

# Database Maintenance - Weekly Saturday at 1:00 AM
0 1 * * 6 postgres psql -d quicknote_prod -c "VACUUM ANALYZE;" >> /var/log/cron_vacuum.log 2>&1

# SSL Certificate Renewal - Daily at 4:00 AM
0 4 * * * root /usr/bin/certbot renew --quiet --post-hook "systemctl reload nginx"

# Security Updates Check - Daily at 7:00 AM
0 7 * * * root apt-get update && apt-get upgrade -s | mail -s "Updates Available" ops@quicknote.app
```

### Cron Best Practices

1. **Always redirect output**: `>> /var/log/script.log 2>&1`
2. **Use absolute paths**: `/opt/scripts/backup.sh` not `backup.sh`
3. **Set environment variables**: PATH, SHELL, MAILTO
4. **Test manually first**: `sudo -u postgres /opt/scripts/backup.sh`
5. **Implement locking**: Prevent overlapping executions
6. **Log everything**: Include timestamps
7. **Send alerts on failure**: Email or monitoring integration
8. **Document purpose**: Add comments explaining each job

---

# Part 3: Application Workflow Design

## 3.1 Three-Tier Application Communication

### Architecture Layers

**1. Presentation Tier (Frontend)**:
- Technology: React SPA
- Responsibility: UI, user interactions, client validation
- Deployment: Static files via Nginx
- Communication: HTTPS API calls

**2. Application Tier (Backend)**:
- Technology: Node.js + Express
- Responsibility: Business logic, authentication, validation
- Deployment: PM2-managed processes
- Communication: HTTP from Nginx, PostgreSQL to database

**3. Data Tier**:
- Technology: PostgreSQL + S3
- Responsibility: Data persistence, file storage
- Deployment: RDS Multi-AZ + S3 buckets
- Communication: PostgreSQL protocol

---

## 3.2 Frontend to Backend Workflow

### Complete User Request Journey

**Scenario**: User saves a new note

**Step 1: User Interaction**
- User types note and clicks "Save"
- React validates input client-side
- Constructs API request

**Step 2: Frontend Prepares Request**
- Method: POST
- URL: https://api.quicknote.app/api/notes
- Headers: Authorization (JWT), Content-Type
- Body: {title, content}

**Step 3-12: Request Processing** (See Part 1, Section 1.10 for detailed 18-step flow)

**Authentication Token/Session Handling**:

**JWT Token Flow**:
1. User logs in with credentials
2. Backend validates and generates JWT
3. Frontend stores token in localStorage
4. Token included in all subsequent requests
5. Backend validates token signature
6. Extracts user ID from token payload
7. Continues with request processing

**Token Expiration**:
- Access token: 15 minutes
- Refresh token: 7 days
- Frontend auto-refreshes before expiration
- Logout clears all tokens

**Failure Handling**:

**Network Failure**:
- Retry 3 times with exponential backoff
- Show "Connection lost" message
- Queue request for later (if appropriate)

**Authentication Failure**:
- 401 response triggers re-login
- Clear stored tokens
- Redirect to login page

**Validation Failure**:
- 400 response with error details
- Display field-specific errors
- Allow user to correct and resubmit

**Server Error**:
- 500 response logged
- Generic error message to user
- Option to retry
- Alert monitoring team

### Backend to Database Workflow

**How Backend Connects Securely**:

1. Connection pooling (pg-pool)
2. Credentials from environment variables
3. TLS encryption required
4. Connection string: `postgresql://user:pass@host:5432/db?sslmode=require`

**Query Execution**:

**Parameterized Queries** (prevents SQL injection):
```javascript
const query = 'INSERT INTO notes (user_id, title, content) VALUES ($1, $2, $3) RETURNING *';
const values = [userId, title, content];
const result = await pool.query(query, values);
```

**Error Handling**:
- Connection error: Retry 3 times, then fail
- Query timeout: Cancel after 30 seconds
- Unique constraint violation: Return 409 Conflict
- General error: Rollback transaction, return 500

**Connection Pool Management**:
- Min connections: 5
- Max connections: 20
- Idle timeout: 30 seconds
- Connection lifetime: 1 hour

### Media/File Upload Workflow

**Step-by-Step**:

1. User selects file in browser
2. Frontend validates: size < 10MB, type allowed
3. POST to /api/upload endpoint
4. Backend generates signed S3 URL
5. Frontend uploads directly to S3
6. S3 returns success
7. Frontend sends metadata to backend
8. Backend stores S3 key in database
9. User receives confirmation

**Why Not Store in Database**:
- Database bloat (slow queries)
- Expensive storage
- Backup overhead
- Not designed for binary data
- S3 cheaper and more scalable

**Secure S3 Access**:
- Pre-signed URLs (time-limited)
- IAM role for EC2 instances
- Bucket policy restricts access
- Server-side encryption
- Versioning enabled

---

## 3.3 Error Handling and Failure Scenarios

### Database Down

**Detection**:
- Health check fails
- Connection attempts timeout
- Query failures spike

**Response**:
1. Application returns 503 Service Unavailable
2. ALB health check fails
3. Traffic routes to healthy instances
4. Monitoring alerts team
5. Auto-failover to standby (Multi-AZ RDS)
6. Recovery typically < 60 seconds

**User Experience**:
- "Service temporarily unavailable" message
- Automatic retry after 30 seconds
- No data loss (requests queued or failed gracefully)

### Application Server Fails

**Detection**:
- Process crash
- Health check endpoint fails
- No response within timeout

**Response**:
1. PM2 immediately restarts process
2. If restart fails 5 times, gives up
3. ALB marks instance unhealthy
4. Traffic redirected to healthy instances
5. Auto Scaling launches replacement instance
6. New instance joins load balancer

**User Experience**:
- In-flight requests may fail (user retries)
- Subsequent requests served by healthy servers
- Minimal disruption (< 2 seconds)

### Web Server Unreachable

**Detection**:
- ALB health check fails
- No response on port 80

**Response**:
1. ALB immediately stops sending traffic
2. All requests go to other AZ
3. CloudWatch alarm triggers
4. Auto Scaling launches replacement
5. systemd attempts service restart

**User Experience**:
- Transparent failover
- No noticeable impact
- Full redundancy maintained

### Network Partition

**Scenario**: AZ-A loses connectivity

**Response**:
1. ALB detects all targets in AZ-A unhealthy
2. Routes 100% traffic to AZ-B
3. Database automatic failover if primary in AZ-A
4. Application continues serving from AZ-B
5. When connectivity restored, instances rejoin

**User Experience**:
- Possible brief slowdown during failover
- No data loss
- Service remains available

### Graceful Degradation

**Principles**:
1. **Core functionality** always available
2. **Optional features** disabled under load
3. **Static content** served even if API down
4. **Cached data** used when fresh data unavailable
5. **Clear communication** to users about limitations

---

## Summary

This QuickNote Production Architecture delivers:

✅ **Secure**: Defense-in-depth with multiple security layers  
✅ **Scalable**: Horizontal scaling across all tiers  
✅ **Highly Available**: Multi-AZ deployment with automatic failover  
✅ **Observable**: Comprehensive logging and monitoring  
✅ **Maintainable**: Clear documentation and automation  
✅ **Cost-Efficient**: Right-sized resources with auto-scaling  

**Document includes**:
- Complete network architecture with 10 infrastructure components
- Detailed security group rules for 5 security groups
- Server configurations for Nginx, Node.js, and PostgreSQL
- Linux administration best practices
- Complete traffic flow analysis (18 steps)
- Application workflow design for all scenarios
- Error handling and recovery procedures

This architecture is production-ready and meets all project requirements for the QuickNote cloud-based note-taking platform.
