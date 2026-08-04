# AWS CSPM Tool — Project Overview

**Subject:** Cloud Security (Semester Assignment)
**Type:** Cloud Security Posture Management (CSPM) tool for AWS

---

## 1. Project Summary

A tool that automatically scans an AWS account for security misconfigurations against industry best practices, assigns a risk score to each finding, uses an LLM API to generate plain-English impact explanations and remediation steps, and presents everything on an interactive dashboard. A scheduled-scan mode compares each run against the last one and pushes alerts (email + Telegram) only for new or worsening findings.

**Why this isn't just "a smaller Prowler/Checkov":** the scanning engine, risk scoring formula, and alerting logic are custom-written (boto3 + Python), not wrapped calls to an existing tool. Prowler/Checkov are used only as an optional validation layer to sanity-check results — a good discussion point in the report, not the core engine.

---

## 2. Architecture — Four Layers

```
AWS Account
   |  (boto3 API calls, read-only)
   v
1. Scanner Engine        -> boto3 checks across IAM, S3, EC2/SG, RDS, CloudTrail/Config
   |  outputs structured JSON findings
   v
2. Risk Scoring Engine   -> custom formula (severity x exposure x data sensitivity), 0-100
   |  scored findings
   v
3. AI Explanation Layer  -> LLM API call per finding -> plain-English impact + remediation
   |  enriched findings
   v
4. Dashboard             -> Streamlit + pandas/plotly, filterable findings table, score gauges
```

### Optional extension — Scheduled Scans + Alerts

```
Task Scheduler (Windows Task Scheduler / Python `schedule` lib)
   v
Scanner reruns (same engine, timestamped output)
   v
Alert Dispatcher (diffs current scan vs last scan -> new/worsening findings only)
   v
Notification Channels: smtplib (email) + Telegram Bot API
```

---

## 3. Tools & Libraries by Layer

| Layer | Tool / Library | Notes |
|---|---|---|
| AWS access | AWS CLI + IAM user | Use `SecurityAudit` or `ReadOnlyAccess` managed policy — least privilege, since the scanner only needs to read config |
| Scanner engine | `boto3` (Python SDK) | Your own check functions, e.g. `check_public_s3_buckets()`, `check_open_sg_ports()` |
| Risk scoring | Plain Python | No library needed — your own weighting logic |
| AI layer | Anthropic or OpenAI API | You write the prompts and parsing glue; the model isn't the "engine," it's a component |
| Dashboard | `streamlit`, `pandas`, `plotly` | Fastest path to a working UI for a solo assignment |
| Validation (optional) | Prowler, Checkov | Run alongside your scanner as a comparison, not as the core |
| Scheduling | `schedule` (Python lib) or Windows Task Scheduler | No need for cron on Windows |
| Email alerts | `smtplib` (built-in) + Gmail app password | No external API needed |
| Telegram alerts | `requests.post()` to Telegram Bot API, or `python-telegram-bot` | Create a bot via BotFather, get a chat ID |

**Skip WhatsApp** for this — it needs Twilio or WhatsApp Business API approval, too much overhead for a semester project. Telegram covers the same "instant mobile alert" demo value.

---

## 4. Suggested Checks (Own boto3 Code)

**IAM**
- Root account MFA disabled
- Unused/old access keys
- Overly permissive policies (`*:*` actions)
- Weak password policy

**S3**
- Public buckets / public ACLs
- Missing default encryption
- Versioning or access logging disabled

**EC2 / Security Groups**
- SSH (22) or RDP (3389) open to `0.0.0.0/0`
- Unused security groups
- Unencrypted EBS volumes

**RDS**
- Publicly accessible instances
- Unencrypted storage
- Automated backups disabled

**Logging / Monitoring**
- CloudTrail disabled or not multi-region
- AWS Config not enabled
- GuardDuty disabled

15–25 checks total is a reasonable scope for one semester.

---

## 5. Risk Scoring — Approach

A simple, defensible formula works better than something overly complex you can't explain in a viva:

```
risk_score = severity_weight x exposure_factor x data_sensitivity_factor
```

- **Severity weight**: base score per check (e.g. public S3 bucket = high, unused SG = low)
- **Exposure factor**: is it internet-facing? (1.0 if public, 0.5 if internal-only)
- **Data sensitivity factor**: does the resource likely hold sensitive data? (tag-based or heuristic)

Normalize to 0–100 per finding, then aggregate to an overall account score. Document the formula clearly in your report — this is the part faculty will ask you to defend.

---

## 6. Demo Idea

Open port 22 to `0.0.0.0/0` live during the presentation, trigger a scan, and show the Telegram alert arriving in real time. This is a strong, memorable way to demonstrate the diff-based alerting actually works, not just the scan/dashboard.

---

## 7. Open Items / Next Steps

- [ ] Scaffold folder structure (`scanner/`, `scoring/`, `ai_layer/`, `dashboard/`, `alerts/`)
- [ ] Write `aws_client.py` — shared boto3 session/auth module (fold in the existing `test_auth.py` connectivity check)
- [ ] Implement first 3–5 checks (start with S3 + Security Groups — highest demo impact)
- [ ] Design JSON schema for findings (resource, check, severity, region, timestamp)
- [ ] Build risk scoring function
- [ ] Wire up AI explanation API call + prompt template
- [ ] Build Streamlit dashboard (findings table + score gauge)
- [ ] Add scheduler + diff-based alert dispatcher
- [ ] Write report section defending the risk scoring formula
