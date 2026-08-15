# Cloud Security Hardening Mini-Project — AWS

A small AWS environment built from a default, unhardened state to a documented, defensible security posture — public/private network segmentation, encrypted storage, least-privilege IAM for separate developer and admin roles, and centralized logging at both the API and network level.

Every control was built permissively first, evidenced, then hardened and re-evidenced. Three genuine incidents occurred during the build (two account self-lockouts and one blocked security-group edit, all from the same root cause) — they're documented as they happened rather than edited out, because the incident-and-recovery record is more informative than a clean run would have been.

Full write-up with all evidence screenshots, before/after tables, and the complete threat narrative is in [`Cloud_Security_Hardening_Report.docx`](./Cloud_Security_Hardening_Report.docx).

## Architecture

- **VPC** (10.0.0.0/16) with one public subnet and one private subnet, single AZ
- **public-web** — EC2 instance (Amazon Linux 2023, t2.micro), public subnet, public IP
- **private-app** — EC2 instance (Amazon Linux 2023, t2.micro), private subnet, no public IP, no outbound internet route (no NAT Gateway)
- **devin-lab-bucket-2026** — S3 bucket, SSE-KMS encrypted, public access blocked
- **devin-lab-s3-key** — customer-managed KMS key, usage restricted to admins
- **devin-lab-trail** — CloudTrail, multi-region, log file validation enabled
- **devin-flow-log** — VPC Flow Logs (all traffic), delivered to a centralized S3 log bucket

## Controls Implemented

| Control | Before | After |
|---|---|---|
| Network segmentation | Flat network | VPC with public + private subnets, no NAT Gateway |
| Security groups | SSH open to `0.0.0.0/0`; private tier open to entire VPC CIDR | SSH restricted to a single IP; private tier restricted to a security-group reference |
| Storage access | S3 bucket publicly readable | Block Public Access on, no bucket policy |
| Storage encryption | Default AWS-managed key (SSE-S3) | Customer-managed KMS key (SSE-KMS) |
| IAM — developer | `AmazonEC2FullAccess` (account-wide) | Custom least-privilege policy, tag-scoped, MFA-gated denies |
| IAM — admin | `AdministratorAccess` only | `AdministratorAccess` + custom MFA-gated policy, tested for gaps |
| API-level logging | None | CloudTrail, multi-region |
| Network-level logging | None | VPC Flow Logs to a centralized bucket |

## Incidents

Three real incidents occurred while building the IAM controls, all sharing the same root cause:

1. **Self-lockout via premature policy removal** — `AdministratorAccess` was detached from the Admins group before the replacement policy had successfully attached, leaving the account with zero IAM permissions and no way to self-recover. Resolved via root.
2. **MFA-gated explicit deny blocking a legitimate admin action** — the custom admin policy's explicit-deny statement blocked a CloudTrail setup step because the active session wasn't flagged as MFA-authenticated, despite MFA being enabled on the account.
3. **Same pattern, different action** — the identical MFA-session issue blocked a security group edit later in the build, confirming this wasn't a one-off but a consistent gap worth investigating further.

Full diagnosis, evidence, and recovery steps for each are in the report, Section 7.

## Threats Faced

Within hours of `public-web` receiving a public IP, its VPC Flow Logs captured real, unsolicited port-22 scanning traffic from multiple unrelated external IPs — the internet's constant background noise looking for open SSH. The security groups built in this project are what stopped it, and the flow logs prove it: every one of those attempts shows up as `REJECT`.

## Tools Used

| Category | Tools |
|---|---|
| Compute | Amazon EC2 (t2.micro), Amazon Linux 2023 |
| Networking | Amazon VPC, security groups, Internet Gateway (no NAT Gateway) |
| Storage | Amazon S3 |
| Encryption | AWS KMS (customer-managed key) |
| Identity | IAM users, groups, customer-managed policies, IAM Policy Simulator |
| Logging | AWS CloudTrail, VPC Flow Logs |
| Cost control | AWS Budgets |

## Repo Contents

- `Cloud_Security_Hardening_Report.docx` — full report: architecture, every control's before/after evidence, all three incidents, and the complete threat narrative
