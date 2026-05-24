---
_favorite_index: 1
Status: In progress
Date: 2026-05-14
_icon: address-book
---
# VP Platform — First-Round Interview Guide

**Duration:** 60 minutes\
**Panel:** 1-2 interviewers (rotate questions from bank per round)\
**Format:** Structured competency interview with scenario-based questions

***

## Interview Structure

| Segment                  | Duration | Focus                                                             |
| ------------------------ | -------- | ----------------------------------------------------------------- |
| Opening & Resume Context | 5 min    | Set expectations, understand candidate's current context          |
| Competency Block A       | 20 min   | Platform Operations & Target Platform Build (pick 3-4 questions)  |
| Competency Block B       | 20 min   | Migration Engineering & Security/Reliability (pick 3-4 questions) |
| Competency Block C       | 10 min   | Team Building & Strategic Thinking (pick 2 questions)             |
| Candidate Questions      | 5 min    | Reverse Q&A                                                       |

***

## Question Bank

### Block A: Platform Operations & Target Platform Build

#### VP-01: Running Production While Modernizing

> You inherit a stable but aging SaaS platform on Oracle/VMware that serves 1200+ retail brands. You need to keep it running at current reliability while building a Kubernetes/Postgres/Kafka cloud-native target. How do you balance "keep the lights on" with "build the future"? Where do you invest first?

**What to listen for:**

- Separates operational work from transformation work explicitly — doesn't let one consume the other
- Protects on-call and incident response capacity while allocating dedicated time for platform build
- Invests first in observability and CI/CD — these serve both current and future state
- Identifies the biggest production risks (Oracle single point of failure, lack of automation, manual deployments) and addresses them first
- Understands that the first priority is "do no harm" — any modernization that destabilizes current customers is a failure
- Has a concrete mental model for resource allocation (e.g., 60/40 ops-to-transform initially, shifting over time)

**Red flags:**

- "We'll modernize in place" without acknowledging risk to live customers
- No concept of phased investment — everything is equally important
- Cannot articulate what to do in the first 30 days vs. the first year
- Dismisses the legacy as "just legacy" without understanding what keeps it running

***

#### VP-02: Multi-Tenant Platform at Scale

> You're building a multi-tenant platform for 1200+ retail brands with three deployment tiers: shared everything, dedicated compute, and dedicated everything. How do you design the platform to handle a noisy tenant whose festive-season traffic (10x normal) degrades performance for all other tenants on the shared tier?

**What to listen for:**

- Talks about resource isolation at every layer: Kubernetes namespace/resource quotas, PostgreSQL connection pooling (PgBouncer) and statement timeouts, Redis memory limits per namespace, Kafka rate limiting
- Describes auto-scaling strategy: HPA on CPU + custom metrics (requests/sec per tenant), cluster autoscaler for node pools
- Mentions tenant-aware rate limiting at the API gateway (Kong) — per-tenant request quotas
- Has a monitoring/alerting strategy that detects noisy neighbors early (per-tenant latency percentiles, not just aggregate)
- Talks about graceful degradation: throttle the noisy tenant rather than letting them degrade everyone
- Understands the cost implication: shared tier must stay profitable; if a tenant needs dedicated resources, that's a tier upgrade conversation

**Red flags:**

- "We'll just add more capacity" — no concept of isolation or throttling
- Cannot articulate specific Kubernetes, Postgres, or Redis isolation mechanisms
- No awareness that noisy neighbor is primarily a shared-tier problem
- Treats all tenants the same regardless of tier

***

#### VP-03: Platform as a Product

> Domain teams (ERP, OMS, POS, etc.) should not need to understand Kubernetes, Kafka configuration, or Postgres connection pooling. How do you make the platform usable by product teams without each team inventing its own infrastructure? What does the interface between platform and domain teams look like?

**What to listen for:**

- Platform provides golden paths: standardized CI/CD pipelines, Helm charts, deployment templates, service scaffolding
- Self-service where possible: teams can provision environments, deploy services, and configure observability through abstraction layers (not filing tickets)
- Platform team operates as an internal product team with domain teams as customers — has a roadmap, takes feedback, prioritizes
- Defines clear contracts: what platform provides (service mesh, secrets, DB provisioning, Kafka topics) and what teams own (their domain code, their tests, their on-call for domain logic)
- Service catalog / developer portal as the interface
- Does not over-abstract: teams can still access logs, metrics, and traces directly when debugging

**Red flags:**

- "We'll give teams runbooks" — that's documentation, not a platform
- Every change requires a platform team ticket
- Cannot describe what "self-service" looks like concretely
- Platform is a black box that teams cannot observe into

***

#### VP-04: CI/CD & Deployment Strategy for 1200 Tenants

> Design a deployment pipeline for a platform where a single release must be rolled out across 1200+ tenant schemas, with canary testing on 5% of tenants before full rollout. What happens when the canary reveals a problem at 3 AM?

**What to listen for:**

- Canary deployment at tenant level: route 5% of tenants to new version, monitor for errors/latency/anomalies
- Automated rollback: if error rate exceeds threshold during canary, automatically revert to previous version
- Blue-green or canary at the Kubernetes level with tenant-aware routing
- Schema migrations run before application deployment, backward-compatible (old version works with new schema)
- Feature flags as a safety net: deploy code but disable new behavior until canary validates
- On-call rotation with clear runbooks: who gets paged, what to check, when to roll back vs. fix forward
- Automated canary analysis: not just "wait 30 minutes and check manually" — metrics-driven decision

**Red flags:**

- "Deploy to staging, test, then deploy to production" — no concept of gradual rollout
- No automated rollback — manual intervention required at 3 AM
- Treats all tenants as one deployment unit
- No mention of backward compatibility or schema migration ordering

***

### Block B: Migration Engineering & Security/Reliability

#### VP-05: Oracle to Postgres Migration at Scale

> Walk me through how you'd execute an Oracle to Postgres migration for a 1200-tenant retail ERP. What's the migration pipeline? How do you validate data correctness? How do you handle rollback? What are the top 3 things that will go wrong?

**What to listen for:**

- Phased approach: not a big-bang migration — tenant cohorts (e.g., batches of 50-100)
- Migration pipeline: schema translation (Oracle DDL -> Postgres DDL), data migration (ETL or CDC), validation (row counts, checksums, financial reconciliation), cutover (switch connection strings), rollback (switch back or re-mirror)
- Validation is not just "same number of rows" — includes field-level comparison, stored procedure output parity, query result equivalence for critical reports
- Rollback is tested before cutover — not just theorized
- Top 3 risks they identify should include: (1) stored procedure / PL-SQL translation bugs, (2) query plan differences causing performance regressions, (3) character encoding / numeric precision differences
- Has experience with migration tools or has built custom tooling
- Understands that the application layer needs to work with both Oracle and Postgres during the transition period

**Red flags:**

- "Use AWS DMS and it handles everything" — no awareness of semantic translation issues
- No concept of phased rollout or tenant cohorts
- Cannot name specific Oracle-to-Postgres gotchas (DATE vs TIMESTAMP, NVL vs COALESCE, ROWNUM, etc.)
- No rollback strategy beyond "restore from backup" (which doesn't work once you've taken writes on Postgres)

***

#### VP-06: Incident Response & SLO Design

> Design an incident response framework for a platform where POS transactions cannot be down for more than 15 minutes during business hours without significant revenue impact for 1200+ brands. What SLOs do you set? How does on-call work? What does a post-incident review look like?

**What to listen for:**

- SLOs are per-service and per-tenant-tier: POS has a tighter SLO (99.95%) than analytics (99.5%)
- Defines SLIs concretely: POS transaction completion rate, inventory sync latency, Kafka consumer lag, DB replication lag
- Error budgets: when the budget is exhausted, the team stops shipping features and focuses on reliability
- On-call: rotation is sustainable (not one person), runbooks exist, escalation paths are clear, on-call is compensated
- Incident severity levels: SEV1 (POS down) paged immediately, SEV2 (degraded) paged during business hours, SEV3 (non-critical) handled next business day
- Post-incident review is blameless, produces action items with owners and deadlines, and tracks completion
- Monitoring catches problems before customers do: alert on symptoms (high error rate, high latency) not causes (CPU usage)

**Red flags:**

- "We set 99.99% availability for everything" — unrealistic and unhelpful
- No concept of error budgets or error budget policies
- On-call is ad-hoc or volunteer-based rather than structured
- Post-incident review is blame-oriented or skipped for "minor" incidents

***

#### VP-07: Security & Compliance Foundation

> You're building a security practice from the ground up for a platform processing financial transactions, GST data, and customer PII for 1200+ Indian brands. The platform needs SOC 2 Type 2 and ISO 27001 readiness within 18 months. Where do you start? What are the non-negotiables in month one?

**What to listen for:**

- Month one non-negotiables: secrets management (Vault or equivalent), WAF at edge, TLS everywhere (including internal mTLS), access control (SSO + RBAC), audit logging for all sensitive operations, vulnerability scanning in CI
- Tenant isolation as a security property: not just functional — tested and verified (JWT tenant_id enforcement, schema isolation, no cross-tenant queries)
- PCI-DSS awareness for POS payment flows: understands gateway delegation model (SAQ-A vs SAQ-D)
- Compliance roadmap: gap assessment -> control implementation -> evidence collection -> auditor engagement
- Security is not a team — it's a practice: platform embeds security into CI/CD (SAST, DAST, dependency scanning), infrastructure (network policies, pod security), and operations (incident response, access reviews)
- Data classification: knows which data is PII, which is financial, which is GST-regulated, and treats each differently

**Red flags:**

- "We'll hire a security consultant" as the primary strategy — no ownership
- Cannot name specific security controls beyond "encryption and passwords"
- No awareness of Indian data protection requirements (IT Act, DPDP Act)
- Treats compliance as a paperwork exercise rather than technical controls

***

#### VP-08: Observability & Split-Brain Scenarios

> A retail platform at 1200+ brands has POS terminals operating offline, Kafka consumers falling behind, and a database replication lag spike — all happening simultaneously during festive season peak. How do you detect this? What does your observability stack look like? And how do you handle the split-brain reconciliation when systems reconnect?

**What to listen for:**

- Observability three pillars: metrics (Prometheus/Grafana), traces (Jaeger/OpenTelemetry), logs (structured, centralized)
- Per-tenant metrics where it matters: not just aggregate Kafka lag but lag per consumer group per topic
- Dashboards organized by symptom: "POS transaction health," "inventory sync health," not by infrastructure component
- Split-brain handling: primary is source of truth, replica replays; POS queues transactions locally and reconciles on reconnect; Kafka consumers auto-catch-up
- Stock oversell detection: when POS and OMS both sell the same item, the reconciliation flags it, alerts store manager, and triggers warehouse expedited shipment
- Alerting strategy: alert on business impact (POS transaction failure rate > 1%) not just infrastructure metrics (CPU > 80%)

**Red flags:**

- Observability = "we have logs and check them when something breaks"
- No concept of distributed tracing across services
- Cannot articulate what happens when data diverges during a partition
- Monitoring is infrastructure-only with no business-level visibility

***

#### VP-09: Schema Migration at 1200 Tenants

> You need to alter a table that exists in all 1200+ tenant schemas. The migration must be zero-downtime, backward-compatible, and auditable. Walk me through the lifecycle from developer writing the migration to full rollout.

**What to listen for:**

- Migration is additive-first: add column with default, deploy new code, remove old column 3 releases later
- No `ALTER TABLE` that locks for more than 5 seconds — uses `CREATE INDEX CONCURRENTLY`, soft defaults
- Migration runs in parallel batches of tenants (e.g., 50 at a time) with progress tracking
- Atomic per tenant: if one tenant's migration fails, it rolls back; others continue
- Backward compatibility verified: old application version works with new schema
- Lifecycle: write migration -> test in CI with seed data -> test on staging with production snapshot -> dry run -> production batch migration -> smoke test 10 random tenants -> monitor 30 minutes -> deploy new app version
- Has an emergency migration procedure for non-backward-compatible changes (never routine)

**Red flags:**

- "Run the migration and then deploy" — no backward compatibility concept
- No concept of tenant-parallel migration or batching
- Cannot articulate what makes a migration "safe" vs. "dangerous"
- No rollback strategy for individual tenant failures

***

### Block C: Team Building & Strategic Thinking

#### VP-10: Building a Platform Team

> You're inheriting a team of infrastructure, DBA, security, DevOps/SRE, and internal IT people — some strong, some junior, some doing things manually that should be automated. How do you assess the team, identify gaps, and build a platform engineering discipline? What does the team structure look like in 12 months?

**What to listen for:**

- Assesses before restructuring: talks to everyone, understands current practices, identifies strengths and gaps
- Identifies critical gaps first: typically security, SRE, and migration engineering are the biggest gaps in a team that grew up doing ops
- Team structure: platform engineering (K8s, CI/CD, IaC), database operations (Oracle + Postgres), security/compliance, SRE/reliability, migration engineering — with clear ownership boundaries
- Grows juniors through pairing with seniors, not just training — embeds them in real work
- Hiring plan is specific: "I need a senior SRE and a security lead in the first 90 days" not "I'll hire as needed"
- Does not create a team that is a bottleneck for every deployment — platform enables, does not gatekeep

**Red flags:**

- "I'll reorganize in the first week" — no assessment phase
- Cannot describe a specific team structure
- Plans to hire a full team of seniors — no plan for growing existing people
- Platform team becomes the deployment bottleneck

***

#### VP-11: Cost Discipline & Cloud Economics

> A platform serving 1200+ brands on Kubernetes/Postgres/Kafka/Redis can easily become very expensive. How do you approach cloud cost management? What are the levers you pull? How do you balance cost optimization against reliability and developer velocity?

**What to listen for:**

- Cost is a first-class concern, not an afterthought: has budget targets, tracks spend per tenant tier, measures unit economics
- Levers: reserved instances for baseline, spot/preemptible for non-critical workloads, right-sizing pods (requests vs. limits), shared infrastructure (connection pooling, Kafka cluster sharing), storage tiering
- Multi-tenant cost amortization: shared tier should have the best unit economics; dedicated tier charges premium
- Does not optimize cost at the expense of reliability: no single-AZ deployments to save money, no skimping on replication
- Measures developer productivity cost: if saving $10K/month in infra costs costs 50 engineer-hours/month in friction, it's not worth it
- Has experience with cloud cost tooling and shows cost dashboards to leadership monthly

**Red flags:**

- "Cloud is cheap, we'll optimize later" — no cost discipline
- Cannot name specific cost optimization techniques
- Willing to compromise reliability for cost savings
- No concept of per-tenant unit economics

***

#### VP-12: Influencing Without Direct Authority

> As VP Platform, you own the platform but domain teams (ERP, OMS, POS) are run by their own VPs. How do you get them to adopt your platform standards, use your CI/CD pipelines, and follow your deployment practices? What happens when a domain team pushes back and says "we have our own way of doing things"?

**What to listen for:**

- Makes the platform the path of least resistance: golden paths that are easier than bespoke solutions
- Demonstrates value before mandating: "we'll migrate your CI/CD first and show you the speed improvement"
- Creates shared incentives: if platform SLOs affect domain team bonuses, alignment is natural
- Has a concrete example of winning over a resistant team
- Escalates to CTO only as a last resort — prefers technical persuasion and demonstrated results
- Standards are not "my way or the highway" — they're negotiated, documented, and have clear rationale
- Understands that some domain-specific tooling is fine as long as it meets platform contract requirements

**Red flags:**

- "I'd mandate it" — command-and-control approach
- No concrete example of influencing peers
- Treats pushback as insubordination rather than a signal to improve the platform
- Cannot articulate why a domain team would want to adopt the platform

***

#### VP-13: First 90 Days Plan

> Based on what you know about this role, what would your first 30/60/90 days look like? What's the first thing you do on day one?

**What to listen for:**

- Day 1: listen — meet the team, understand the current architecture, review recent incidents, understand the biggest risks
- Days 1-30: audit — infrastructure assessment, environment inventory, access review, cloud spend analysis, top production risks, DBA practice assessment. Identify the "if this fails, the company is in trouble" risks.
- Days 31-60: set standards — CI/CD standardization, observability baseline, incident response process, security baseline, environment provisioning. Not building the full platform yet — establishing the operating model.
- Days 61-90: execute — publish platform roadmap, start target platform staging, define migration tooling ownership with QA gates, first hires for critical gaps.
- Prioritizes based on risk: what's the most dangerous thing that could fail right now?
- Balances quick wins (automate the most painful manual process) with foundational work (set up the CI/CD pipeline that everything else depends on)

**Red flags:**

- Wants to rearchitect everything in month one
- No listening phase — comes in with a pre-baked plan
- Plan is all strategy with no concrete deliverables
- Cannot articulate what they'd do differently if they discovered the current infra is worse (or better) than expected

***

## Scoring Rubric

### Rating Scale

| Score | Label               | Description                                                                                                                                                                      |
| ----- | ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **5** | Exceptional         | Demonstrates deep expertise well beyond the role requirement. Provides novel insights based on battle scars. Could set strategy independently. Sets the bar for this competency. |
| **4** | Strong              | Clearly meets the requirement with concrete, relevant examples. Demonstrates hands-on depth in infrastructure/platform. Minor gaps that are closeable.                           |
| **3** | Meets Bar           | Adequate understanding and relevant experience. Can handle the role with some support. Answers are correct but lack depth, specificity, or scale awareness.                      |
| **2** | Below Bar           | Gaps in understanding or experience. Answers are theoretical without practical grounding. Would need significant ramp-up on critical platform concerns.                          |
| **1** | Significantly Below | Fundamental misunderstanding or no relevant experience. Cannot articulate basic concepts for this competency.                                                                    |

### Competency Scoring Matrix

| Competency                                      | Weight | Question(s)         | Score (1-5) | Notes                                                                      |
| ----------------------------------------------- | ------ | ------------------- | ----------- | -------------------------------------------------------------------------- |
| **Platform Operations at Scale**                | 20%    | VP-01, VP-02        |             | Running 1200+ tenants, noisy neighbor, multi-tier operations               |
| **Platform Engineering & Developer Experience** | 15%    | VP-03, VP-04        |             | Platform as product, CI/CD, deployment strategy, golden paths              |
| **Migration Engineering (Oracle → Postgres)**   | 20%    | VP-05, VP-09        |             | Migration pipeline, validation, rollback, schema migration at 1200 tenants |
| **Security & Reliability**                      | 15%    | VP-06, VP-07, VP-08 |             | Incident response, SLOs, security baseline, observability, split-brain     |
| **Team Building & Leadership**                  | 15%    | VP-10, VP-12        |             | Building platform discipline, hiring, influencing without authority        |
| **Strategic Thinking & Cost**                   | 15%    | VP-11, VP-13        |             | Cloud economics, first 90 days, prioritization under uncertainty           |

### Decision Thresholds

| Weighted Score | Recommendation                                                                                            |
| -------------- | --------------------------------------------------------------------------------------------------------- |
| **4.0+**       | Strong Hire — advance to final round immediately                                                          |
| **3.5 - 3.9**  | Hire — advance to final round, note development areas                                                     |
| **3.0 - 3.4**  | Lean Hire — advance only if strong in migration engineering + platform operations; otherwise discuss      |
| **2.5 - 2.9**  | Lean No Hire — significant gaps in critical areas; reconsider only if exceptional in migration experience |
| **Below 2.5**  | No Hire — does not meet the bar for the role                                                              |

### Critical Gates (Must Score ≥ 3 on These)

Regardless of weighted total, a candidate **must** score at least 3 on:

1. **Platform Operations at Scale** — this is 1200+ tenants, not a startup
2. **Migration Engineering** — Oracle-to-Postgres migration is the #1 program risk
3. **Team Building & Leadership** — this is a VP role, not an IC role

If a candidate scores below 3 on any of these, the recommendation should be **No Hire** regardless of weighted total.

***

## Interviewer Rotation Guide

### Panel A Set (Operations + Migration Focus)

- VP-01 (Running Production While Modernizing)
- VP-02 (Multi-Tenant Scale)
- VP-05 (Oracle to Postgres Migration)
- VP-09 (Schema Migration at 1200 Tenants)
- VP-10 (Building a Platform Team)
- VP-13 (First 90 Days)

### Panel B Set (Security + Platform Engineering Focus)

- VP-03 (Platform as a Product)
- VP-04 (CI/CD & Deployment at 1200 Tenants)
- VP-06 (Incident Response & SLOs)
- VP-07 (Security & Compliance)
- VP-08 (Observability & Split-Brain)
- VP-13 (First 90 Days)

### Panel C Set (Leadership + Economics Focus)

- VP-01 (Running Production While Modernizing)
- VP-05 (Oracle to Postgres Migration)
- VP-10 (Building a Platform Team)
- VP-11 (Cost Discipline)
- VP-12 (Influencing Without Authority)
- VP-13 (First 90 Days)

> **Note:** VP-13 (First 90 Days) is included in all panels as a closing question. It tests synthesis, prioritization, and self-awareness.

***

## Post-Interview

After the interview, complete the scoring matrix and write 3-5 sentences on:

1. **Biggest strength** — what impressed you most
2. **Biggest risk** — what concerns you most (especially: can they actually run a 1200-tenant platform, or are they a 50-tenant person in a 1200-tenant role?)
3. **Recommendation** — with specific reasoning

Submit scores within 24 hours to maintain calibration across panels.
