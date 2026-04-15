<!-- GitAds-Verify: EB35LK4F5GZV5FU37AW9CCHVH9I87M4R -->

# AWS Hidden Costs

[![Sponsored by GitAds](https://gitads.dev/v1/ad-serve?source=atryx/aws-hidden-costs@github)](https://gitads.dev/v1/ad-track?source=atryx/aws-hidden-costs@github)

> The AWS costs nobody warns you about — data transfer, NAT gateways, CloudWatch, and 30+ more billing surprises that will blow your budget.

**Your AWS bill is higher than it should be. This repo explains exactly why.**

---

## Table of Contents

- [The Big Picture](#the-big-picture)
- [Networking Costs](#networking-costs)
- [Compute Costs](#compute-costs)
- [Storage Costs](#storage-costs)
- [Database Costs](#database-costs)
- [Observability Costs](#observability-costs)
- [Security Costs](#security-costs)
- [Serverless Costs](#serverless-costs)
- [Container Costs](#container-costs)
- [AI/ML Costs](#aiml-costs)
- [Cost Optimization Checklist](#cost-optimization-checklist)
- [Related Repos](#related-repos)
- [Contributing](#contributing)

---

## The Big Picture

Most AWS pricing pages show per-unit costs. What they don't show is how those units **multiply** in production. Here's a real-world example:

### A "simple" web app's monthly bill breakdown

| Service | What You Expected | What You Got | Why |
|---------|------------------|-------------|-----|
| EC2 (2x t3.medium) | $60 | $60 | As expected |
| RDS (db.t3.medium) | $50 | $50 | As expected |
| NAT Gateway | $0 (forgot about it) | $180 | Data processing fees |
| Data Transfer | $10 | $95 | Cross-AZ + internet egress |
| CloudWatch | $0 (free tier) | $45 | Custom metrics + log storage |
| ALB | $16 | $35 | LCU charges from traffic patterns |
| EBS Snapshots | $5 | $40 | Never cleaned up old snapshots |
| Secrets Manager | $0 | $15 | Per-secret + per-API-call pricing |
| **Total** | **$141** | **$520** | **3.7x over estimate** |

> **The predictable services (EC2, RDS) are usually right. The "invisible" services are where your money goes.**

---

## Networking Costs

### 1. NAT Gateway — The Silent Budget Killer

**The surprise:** NAT Gateway charges $0.045/GB for data **processed** (in AND out). If your private subnets talk to the internet at all — pulling Docker images, calling APIs, downloading packages — it all flows through NAT and you pay per GB.

**Typical monthly impact:** $100–$2,000+

**How to avoid:**
- Use VPC endpoints for AWS services (S3, DynamoDB, ECR, etc.) — $7.30/month per endpoint vs $0.045/GB through NAT
- Use S3 Gateway Endpoint (free) for S3 traffic
- Pull container images from ECR through VPC endpoint
- Monitor NAT Gateway data processing in Cost Explorer → filter by "NatGateway"
- Consider Squid proxy or NAT instances for cost-sensitive workloads

### 2. Cross-AZ Data Transfer — $0.01/GB Adds Up Fast

**The surprise:** Traffic between availability zones costs $0.01/GB in each direction ($0.02 round trip). Microservices chatting across AZs generate massive cross-AZ bills.

**Typical monthly impact:** $50–$500

**How to avoid:**
- Colocate chatty services in the same AZ (use pod topology constraints in K8s)
- Use AZ-aware routing — ALB can prefer same-AZ targets
- Cache aggressively to reduce cross-AZ API calls
- Monitor with VPC Flow Logs → filter cross-AZ traffic

### 3. Elastic IP — Charged When NOT in Use

**The surprise:** An Elastic IP attached to a running instance is free. An EIP **not** attached to anything costs $0.005/hour ($3.60/month). AWS also charges for the **first** EIP on a running instance if it has >1 EIP.

**Typical monthly impact:** $5–$50 (adds up with forgotten resources)

**How to avoid:**
- Audit unused EIPs monthly: `aws ec2 describe-addresses --filters "Name=association-id,Values="`
- Automate cleanup with Lambda + EventBridge rules
- Use ALB/NLB instead of assigning EIPs to individual instances

### 4. Data Transfer Out to Internet — The Egress Tax

**The surprise:** All data leaving AWS to the internet costs $0.09/GB (first 10TB). Serving images, APIs, downloads, or backups to external systems adds up.

**Typical monthly impact:** $50–$5,000+

**How to avoid:**
- Use CloudFront for static content (first 1TB/month free, then $0.085/GB — cheaper than direct egress)
- Compress API responses (gzip/brotli saves 70-80% on transfer costs)
- Keep data processing inside AWS — don't pull large datasets to on-prem for analysis
- Use S3 Transfer Acceleration only when needed (additional $0.04/GB)

### 5. VPC Peering / Transit Gateway — Hidden Per-GB Charges

**The surprise:** VPC peering itself is free, but cross-AZ peered traffic costs $0.01/GB. Transit Gateway costs $0.02/GB per attachment + $0.05/hour per attachment.

**Typical monthly impact:** $100–$1,000 (multi-VPC setups)

**How to avoid:**
- Consolidate VPCs where possible — fewer peering connections = less cross-VPC traffic
- Use Transit Gateway only when you have 5+ VPCs (the per-hour cost justifies itself at scale)
- Monitor per-attachment data processing in Cost Explorer

---

## Compute Costs

### 6. EC2 — Stopped Instances Still Cost Money

**The surprise:** You stop an EC2 instance to save money. You still pay for: attached EBS volumes, Elastic IPs, and any associated snapshots.

**Typical monthly impact:** $20–$200

**How to avoid:**
- Terminate (not just stop) instances you don't need
- Detach and delete EBS volumes from stopped instances
- Use ASGs with scheduled scaling (scale to 0 at night for dev environments)
- Automate dev environment shutdown with Lambda: 7pm stop, 8am start

### 7. EC2 — T-series Burstable CPU Credits

**The surprise:** t3/t3a instances have "burstable" performance. You get baseline CPU credits. When you burst beyond baseline, you burn credits. When credits run out, your instance is throttled to 20% performance — or you pay $0.05/vCPU-hour for "unlimited" mode.

**Typical monthly impact:** $10–$100

**How to avoid:**
- Monitor `CPUCreditBalance` in CloudWatch — if it hits zero, you're being throttled
- If sustained CPU > 20% (t3.medium baseline), switch to m-series (fixed performance, often cheaper)
- Enable T3 Unlimited only if you understand the cost implications
- Right-size: `compute-optimizer` recommendations are actually useful here

### 8. EC2 — Graviton Savings You're Leaving on the Table

**The surprise:** Not a hidden cost, but a hidden **savings**. Graviton (ARM) instances are 20-40% cheaper than x86 equivalents with better performance. Most teams don't switch because "we haven't tested on ARM."

**Typical monthly savings:** 20-40% on compute

**How to capture:**
- Most languages (Node.js, Python, Java, Go, .NET 6+) work on ARM with zero code changes
- Use multi-arch Docker images (`docker buildx build --platform linux/amd64,linux/arm64`)
- Start with non-production workloads, then migrate
- Graviton4 (R8g, M8g, C8g) delivers even better price/performance

---

## Storage Costs

### 9. EBS Snapshots — The Growth You Never Notice

**The surprise:** Snapshots are incremental but never automatically cleaned up. A daily snapshot schedule for 10 volumes generates 3,650 snapshots/year. At $0.05/GB-month for snapshot storage, this compounds.

**Typical monthly impact:** $50–$500

**How to avoid:**
- Use Data Lifecycle Manager (DLM) to automatically expire old snapshots
- Audit snapshot age: `aws ec2 describe-snapshots --owner-ids self --query 'sort_by(Snapshots, &StartTime)'`
- Set retention policies: 7 days for dev, 30 days for staging, 90 days for prod
- Tag snapshots at creation for easy cost allocation

### 10. S3 — It's Not Just Storage Costs

**The surprise:** S3 storage is cheap ($0.023/GB for Standard). But S3 also charges for: PUT requests ($0.005/1000), GET requests ($0.0004/1000), lifecycle transitions, and data retrieval (for Glacier/IA).

**Hidden S3 charges:**

| Operation | Cost | Surprise Factor |
|-----------|------|----------------|
| PUT/POST/LIST | $0.005 per 1,000 | High write volume = expensive |
| GET/SELECT | $0.0004 per 1,000 | CDN origin pulls multiply this |
| Lifecycle transition | $0.01 per 1,000 objects | Moving millions of files to IA/Glacier |
| S3 Select | $0.002/GB scanned | Scanning large objects adds up |
| Glacier retrieval | $0.01/GB (expedited: $0.03/GB) | Restoring from Glacier is shockingly expensive at scale |

**How to avoid:**
- Use S3 Intelligent-Tiering for unpredictable access patterns (small monitoring fee, but auto-optimizes)
- Consolidate small objects — 1 million 1KB objects costs more in requests than 1 GB file
- Set lifecycle policies to expire incomplete multipart uploads (they're invisible but billed)
- Monitor S3 request metrics in CloudWatch — high request counts = unexpected bills

### 11. EBS — GP3 vs GP2: Free Performance You're Missing

**The surprise:** GP2 volumes scale IOPS with size (3 IOPS/GB). Need 3000 IOPS? You need a 1TB volume. GP3 gives 3000 IOPS and 125 MB/s baseline at any size, and it's 20% cheaper.

**Typical monthly savings:** 20% on EBS costs

**How to capture:**
- Convert all GP2 volumes to GP3: `aws ec2 modify-volume --volume-type gp3`
- No downtime, no detach required
- GP3 also lets you provision additional IOPS/throughput independently from size

---

## Database Costs

### 12. RDS — Multi-AZ Doubles Your Cost (Obviously, But Also Hidden Charges)

**The surprise:** Multi-AZ costs 2x the instance price (expected). But you also pay for: cross-AZ data transfer between primary and standby, storage for both instances, and automated backup storage beyond the free allocation.

**Additional hidden charges:**

| Item | Cost |
|------|------|
| Cross-AZ replication traffic | $0.01/GB |
| Backup storage beyond DB size | $0.095/GB-month |
| Snapshot export to S3 | $0.01/GB |
| Performance Insights (detailed) | $0 for 7 days, then per-vCPU pricing |

### 13. RDS — Read Replicas Transfer Costs

**The surprise:** Creating a read replica in a different AZ incurs cross-AZ data transfer for all replication traffic. Cross-region replicas are even more expensive.

**How to avoid:**
- Place read replicas in the same AZ as your application where possible
- Use RDS Proxy for connection pooling instead of additional replicas for connection scaling
- For global reads, evaluate DynamoDB Global Tables vs RDS cross-region replicas

### 14. DynamoDB — On-Demand Mode Can Be 7x More Expensive

**The surprise:** On-Demand mode costs $1.25/million WCUs vs $0.00065/WCU-hour for Provisioned. For consistent workloads, On-Demand can be 5-7x more expensive.

**When On-Demand makes sense:** Unpredictable, spiky traffic with long idle periods.

**When Provisioned wins:** Any workload with baseline traffic you can predict.

**How to optimize:**
- Start with On-Demand to understand your traffic pattern
- Switch to Provisioned + Auto Scaling once you know baseline
- Use Reserved Capacity for predictable workloads (save additional 50-75%)
- Monitor consumed vs provisioned capacity — over-provisioning is waste

### 15. ElastiCache / MemoryDB — Reserved Instance Trap

**The surprise:** ElastiCache has no auto-scaling for Redis (only Memcached). You provision for peak. At 3 AM, you're paying for 100% capacity serving 5% traffic.

**How to avoid:**
- Use ElastiCache Serverless (launched 2023) for variable workloads
- For Redis: use smaller node types with more shards (horizontal scaling)
- Buy Reserved Instances for baseline, use on-demand for burst

---

## Observability Costs

### 16. CloudWatch Logs — The Unexpected Top-5 Line Item

**The surprise:** CloudWatch Logs charges $0.50/GB **ingested** + $0.03/GB-month stored. A busy cluster logging at 10GB/day = $150/month ingestion + growing storage costs.

**Typical monthly impact:** $50–$1,000+

**How to avoid:**
- Set log retention policies (default is **forever** — logs grow indefinitely)
- Filter logs before ingestion — don't send DEBUG level to CloudWatch
- Use subscription filters to route logs to cheaper storage (S3) for archival
- Consider self-hosted logging (Loki + S3 backend) for large volumes
- Use CloudWatch Logs Insights only when needed — it charges $0.005/GB scanned

### 17. CloudWatch Metrics — Custom Metrics Are Expensive

**The surprise:** The first 10 custom metrics are free. After that: $0.30/metric/month. If you instrument 100 microservices with 20 custom metrics each, that's 2,000 metrics = $600/month. Add high-resolution (1-second) metrics at $0.30/metric and it doubles.

**How to avoid:**
- Use EMF (Embedded Metric Format) to embed metrics in logs (cheaper)
- Aggregate metrics before sending — one metric with dimensions, not many separate metrics
- Use Prometheus + Grafana (self-hosted) for detailed metrics
- CloudWatch dashboards cost $3/month each — clean up unused dashboards

### 18. X-Ray / Distributed Tracing — Sampling Matters

**The surprise:** X-Ray charges $5/million traces recorded + $0.50/million traces retrieved. At 100% sampling with moderate traffic (1000 req/s), that's $13,000/month.

**How to avoid:**
- Sample at 1-5% in production (you rarely need 100% of traces)
- Use adaptive sampling — sample more for errors, less for 200 OK
- Consider OpenTelemetry + Jaeger/Tempo for self-hosted tracing
- AWS Distro for OpenTelemetry is free — it's the backends that cost

---

## Security Costs

### 19. Secrets Manager — Per-Secret AND Per-Call Pricing

**The surprise:** $0.40/secret/month + $0.05/10,000 API calls. If your app reads 20 secrets on every container startup, and you have 50 containers restarting frequently, the API call charges multiply.

**Typical monthly impact:** $10–$100

**How to avoid:**
- Cache secrets in your app (don't call Secrets Manager on every request)
- Use SSM Parameter Store (SecureString) for simpler secrets — it's cheaper
- Batch secret reads with `GetSecretValue` for multiple values in one secret

### 20. AWS WAF — Rule Evaluation Charges

**The surprise:** WAF costs $5/month per web ACL + $1/month per rule + $0.60/million requests evaluated. With managed rule groups (each counting as a rule) and high traffic, this adds up.

**Typical monthly impact:** $50–$500

**How to avoid:**
- Consolidate web ACLs where possible (one per ALB, not one per service)
- Use rate-based rules instead of paying for third-party bot protection
- Monitor rule match rates — remove rules that never trigger

### 21. GuardDuty — Charged Per Event Volume

**The surprise:** GuardDuty charges based on CloudTrail events, VPC Flow Logs, and DNS query logs analyzed. In a busy account with many API calls, this can be $100+/month.

**How to manage:**
- Keep it enabled (the security value outweighs cost) but monitor spending
- Use `Usage` tab in GuardDuty console to see per-source costs
- Suspend in dev/test accounts if not needed

---

## Serverless Costs

### 22. Lambda — Memory/Duration Trap

**The surprise:** Lambda charges per GB-second. A 512MB function running for 5 seconds costs the same as a 1024MB function running for 2.5 seconds. But the 1024MB function is faster (more CPU) and often completes in less than half the time — making it **cheaper**.

**How to optimize:**
- Use AWS Lambda Power Tuning tool to find the optimal memory setting
- Higher memory = more CPU = faster execution = often cheaper
- Monitor with `aws lambda get-function-configuration` for memory/duration stats
- Set timeout wisely — runaway functions burn money

### 23. API Gateway — Per-Request Pricing Surprises

**The surprise:** API Gateway REST API: $3.50/million requests. At 100 req/s that's $9,072/month just for the gateway. WebSocket API: $1/million connection-minutes + $1/million messages.

**How to avoid:**
- Use ALB ($0.008/LCU-hour) instead of API Gateway for simple HTTP routing
- Use API Gateway HTTP API ($1/million requests) instead of REST API when possible
- Enable caching ($0.02/hour per 0.5GB) to reduce backend invocations
- Use CloudFront in front of API Gateway (can be cheaper with caching)

### 24. Step Functions — Standard vs Express

**The surprise:** Standard Step Functions: $25/million state transitions. A workflow with 10 states processing 100K events/day = $750/month. Express: $0.00001/5-minute window, significantly cheaper for high-volume.

**How to avoid:**
- Use Express workflows for high-volume, short-duration workflows (< 5 minutes)
- Standard only for long-running, low-volume orchestration
- Monitor state transition counts — they multiply with parallel states and map iterations

---

## Container Costs

### 25. ECS/Fargate — You're Probably Over-Provisioned

**The surprise:** Fargate charges per vCPU-hour ($0.04048) and per GB-hour ($0.004445). You must round up to the nearest valid combination. A service needing 0.3 vCPU and 0.7 GB RAM must use 0.5 vCPU / 1 GB — you pay for 40% waste.

**Valid Fargate configurations (not linear):**

| vCPU | Memory Options |
|------|---------------|
| 0.25 | 0.5, 1, 2 GB |
| 0.5 | 1, 2, 3, 4 GB |
| 1 | 2, 3, 4, 5, 6, 7, 8 GB |
| 2 | 4–16 GB (1 GB increments) |
| 4 | 8–30 GB (1 GB increments) |

**How to optimize:**
- Use Fargate Spot for non-critical workloads (save 70%)
- Monitor actual utilization with Container Insights — right-size aggressively
- For predictable workloads, EC2 + Savings Plans is 50% cheaper than Fargate
- Use Graviton-based Fargate (20% cheaper): `--runtime-platform '{"cpuArchitecture":"ARM64"}'`

### 26. ECR — Image Storage Accumulates

**The surprise:** ECR charges $0.10/GB-month for image storage. If CI/CD pushes a new image on every commit and you never clean up, storage grows fast. 100 images × 500MB = 50 GB = $5/month (seems small, but across 20 repos...).

**How to avoid:**
- Enable ECR lifecycle policies: keep last N images, expire untagged after 1 day
- Use multi-stage builds to minimize image size
- Share base layers across images in the same repository

---

## AI/ML Costs

### 27. SageMaker Endpoints — Idle Inference Is Expensive

**The surprise:** A `ml.m5.large` inference endpoint costs $0.115/hour ($83/month) whether it serves 0 or 10,000 requests. Most ML models are idle 90% of the time.

**How to avoid:**
- Use SageMaker Serverless Inference for sporadic traffic (pay per invocation)
- Use auto-scaling with scale-to-zero capability
- Use SageMaker Multi-Model Endpoints to share a single endpoint across models
- For batch: use SageMaker Batch Transform instead of real-time endpoints

### 28. Bedrock — Token Costs Scale Non-Linearly

**The surprise:** Bedrock charges per input/output token. A chatbot with long conversation history sends the full context on every turn. Token count grows quadratically with conversation length.

**Typical cost per 1M tokens (2026):**

| Model | Input | Output |
|-------|-------|--------|
| Claude 3.5 Sonnet | $3 | $15 |
| Claude 3.5 Haiku | $0.25 | $1.25 |
| Llama 3.1 70B | $0.72 | $0.72 |

**How to avoid:**
- Implement conversation summarization (compress history periodically)
- Use prompt caching where available
- Route simple queries to cheaper models (Haiku), complex to expensive (Sonnet)
- Set max_tokens limits to prevent runaway output costs

---

## Cost Optimization Checklist

### Quick Wins (Do These First)

- [ ] Convert all GP2 EBS volumes to GP3 (20% savings, zero downtime)
- [ ] Set CloudWatch Logs retention policies (default = forever)
- [ ] Delete unused EBS snapshots older than 90 days
- [ ] Release unattached Elastic IPs
- [ ] Enable S3 Intelligent-Tiering on large buckets
- [ ] Add ECR lifecycle policies to all repositories
- [ ] Right-size RDS instances (most are over-provisioned by 2x)

### Medium Effort (This Month)

- [ ] Add VPC endpoints for S3, DynamoDB, ECR (eliminate NAT Gateway costs)
- [ ] Switch to Graviton instances where possible (20-40% savings)
- [ ] Buy Savings Plans for predictable compute (save 30-50%)
- [ ] Implement Lambda Power Tuning for all functions
- [ ] Set up Cost Anomaly Detection in AWS Cost Explorer
- [ ] Tag all resources for cost allocation (you can't optimize what you can't measure)

### Strategic (This Quarter)

- [ ] Migrate to Fargate Spot for non-critical workloads
- [ ] Implement self-hosted logging (Loki + S3) for high-volume services
- [ ] Set up kubecost or OpenCost for K8s cost visibility
- [ ] Create per-team AWS budgets with alerts
- [ ] Audit cross-AZ traffic patterns and colocate chatty services

### Cost Monitoring Tools

| Tool | Type | What It Shows |
|------|------|--------------|
| AWS Cost Explorer | Native | Service-level spend, trends |
| AWS Budgets | Native | Alerts on spend thresholds |
| Cost Anomaly Detection | Native | Unusual spend patterns |
| Compute Optimizer | Native | Right-sizing recommendations |
| kubecost / OpenCost | Open source | Per-namespace K8s costs |
| Infracost | Open source | Terraform cost estimation before deploy |
| Vantage | SaaS | Multi-cloud cost analytics |

---

## Related Repos

| Repo | Description |
|------|-------------|
| [what-they-dont-tell-you-about-kubernetes](https://github.com/atryx/what-they-dont-tell-you-about-kubernetes) | K8s gotchas including cost section |
| [terraform-gotchas](https://github.com/atryx/terraform-gotchas) | Terraform pitfalls for infrastructure |
| [devops-learning-path](https://github.com/atryx/devops-learning-path) | Full DevOps roadmap including cloud costs |
| [production-ready-snippets](https://github.com/atryx/production-ready-snippets) | Production configs for AWS, Docker, K8s |
| [docker-vs-podman](https://github.com/atryx/docker-vs-podman) | Container runtime comparison |

---

## Contributing

Found a hidden AWS cost we missed? [See CONTRIBUTING.md](CONTRIBUTING.md) to add it.

---

## License

MIT — Share freely, attribute kindly.

⭐ **If this saved you money on your AWS bill, star this repo.**
