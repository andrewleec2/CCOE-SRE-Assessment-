# Tier 1 Reliability Assessment: AWS

Oct 7, 2026 · @Andrew Lee

## Selected cloud: Amazon Web Services (AWS)

AWS is the selected cloud for every task, service and runbook in this submission; no other cloud provider is used. No hybrid or second-cloud dependency exists. The non-AWS components are open-source software running inside the AWS workload on Amazon EKS, or operational tools rather than hosting dependencies, each justified where it appears: Istio service mesh (Task 3, chosen because AWS App Mesh reached end of support), KEDA for consumer-lag scaling and Fluent Bit for log shipping (Tasks 2 and 3), and the organization's existing paging and chat tools for on-call and incident channels (Tasks 2 and 5).

### Summary

The service detects failures late because alerts watch components, not customer journeys. It turns provider slowdowns into regional incidents because retries are unbounded and stacked across layers. Its recovery is unproven because DR has never run end to end, and it lacks an owner because no team holds an error budget. Technical controls without ownership decay, and ownership without controls cannot stop a retry storm, so this plan fixes both:

- **Detect:** measure three customer journeys at the edge against 99.99% and page only on error budget burn, not on causes.
- **Contain:** allow retries at exactly one layer under a 20% retry budget, add circuit breakers, bulkheads and load shedding, and put both providers behind fallbacks.
- **Recover:** run a warm standby Region driven by an Amazon Application Recovery Controller (ARC) Region switch plan, and prove RTO 45 minutes and RPO 5 minutes in drills, not on paper.
- **Own:** each journey SLO has a named product owner; SRE owns the platform, the standards and the drills, under a graduated error budget policy that protects delivery capacity.

### Assumptions and exclusions

| # | Assumption or exclusion | Why it matters |
| --- | --- | --- |
| A1 | The critical data set for RTO and RPO is the transaction record in Amazon Aurora plus idempotency records in Amazon DynamoDB. Events published to Amazon MSK are written first to a transactional outbox table in Aurora, so Aurora replication also protects in-flight events. | Defines what the 5-minute RPO covers and removes MSK as a separate system of record. |
| A2 | Primary and standby are two AWS Regions inside the same data residency boundary. The pair is confirmed against availability of Aurora Global Database, MSK Replicator and ARC Region switch before build. | Not every AWS feature is in every Region. |
| A3 | Both external providers expose HTTPS APIs with contractual SLAs below 99.99% (assumed 99.9%), support idempotency keys, and allowlist caller IP addresses. | Drives fallback design and pre-registration of standby egress IPs. |
| A4 | Provider calls are asynchronous to the customer: the customer receives "accepted" and fulfilment completes later. | Lets the synchronous journeys hold 99.99% even when a provider is below it. |
| E1 | CI/CD design is out of scope except release gating by error budget and deployment of configuration during failover. | Scope control. |
| E2 | Detailed cost modelling is out of scope; cost trade-offs are stated qualitatively. | Scope control. |

**AI assistance disclosure.** This submission was prepared with two AI assistants: Claude (Anthropic, Claude Opus 5.5) and Gemini (Google). I selected AWS, supplied the assessor's scenario and constraints, provided the initial architecture diagram, set the review criteria and made the final design decisions. Gemini produced an earlier draft and review comments, including the flag that AWS App Mesh reached end of support on 30 September 2026. Claude drafted most of the final text, produced the architecture diagram, and verified service status and metrics against AWS documentation, which led to replacing App Mesh with Istio on EKS, Systems Manager Incident Manager with CloudWatch and SNS alert routing, and ARC readiness checks with ARC Region switch plan evaluation. I cross-checked the two assistants against each other and against the cited sources (see References) before accepting any claim, and I take responsibility for the submitted content.

### Where each required deliverable is

| Required deliverable | Section | Task and weight |
| --- | --- | --- |
| Service reliability model | Service reliability model (Task 1) | Task 1, 20% |
| SLI/SLO specification | SLI/SLO specification and error budget policy (Task 1) | Task 1, 20% |
| Observability and resilience architecture | Parts 1 and 2 (Tasks 2 and 3) | Task 2, 15%; Task 3, 20% |
| DR runbook and game-day plan | DR runbook and game-day plan (Task 4) | Task 4, 20% |
| Incident command model | Incident command model (Task 5) | Task 5, 15% |
| Prioritized 90-day roadmap | Prioritized 90-day roadmap (Task 6) | Task 6, 10% |
| Mandatory architecture diagram | Mandatory architecture diagram (AWS) | All tasks |

## Service reliability model (Task 1)

Three critical user journeys (CUJs) carry the 99.99% target; every dependency, failure mode and owner below is mapped to them.

### Critical user journeys

| ID | Journey | Path | Mode | Target |
| --- | --- | --- | --- | --- |
| CUJ-1 | Submit a transaction | Client → CloudFront + AWS WAF → API Gateway (REST, Regional) → VPC Link → internal NLB → EKS transaction service → RDS Proxy → Aurora writer; idempotency check in DynamoDB; event written to Aurora outbox | Synchronous | 99.99% availability |
| CUJ-2 | View transaction or account status | Client → CloudFront + AWS WAF → API Gateway → EKS query service → Aurora reader | Synchronous | 99.99% availability |
| CUJ-3 | Fulfil the transaction with external providers | Outbox relay → MSK topic → EKS consumer → Network Firewall egress → Provider 1 or 2 → result event → customer notified | Asynchronous | 99.9% completed within 5 minutes |

### Dependency map

| Dependency | Used by | Failure domain | How failure shows | Design response |
| --- | --- | --- | --- | --- |
| CloudFront + AWS WAF | CUJ-1, 2 | Global edge | Edge 5xx, origin timeouts | Origin resolved through a Route 53 failover record controlled by ARC; WAF rule changes staged in count mode first |
| API Gateway (REST, Regional) | CUJ-1, 2 | Region | 5xx, 429, latency | Method-level throttling, per-route integration timeouts, Region switch |
| EKS services + Istio | CUJ-1, 2, 3 | AZ, Region | Pod crash, saturation, sidecar misconfiguration | Three-AZ topology spread, PodDisruptionBudgets, ARC zonal shift, canary deploys |
| Aurora Global Database | CUJ-1, 2 | AZ, Region | Writer failover, write stalls, replication lag | Multi-AZ writer and readers, RDS Proxy, Global Database secondary in standby Region |
| DynamoDB Global Tables | CUJ-1 | Region | Throttling, replication delay | On-demand capacity, Aurora unique constraint as final duplicate guard |
| Amazon MSK | CUJ-3 | AZ, Region | Broker loss, consumer lag, poison messages | 3 AZs, replication factor 3, min.insync.replicas 2, retry and DLQ topics, MSK Replicator |
| Provider 1, Provider 2 | CUJ-3 | Outside AWS | Timeouts, error spikes, slow responses, full outage | Timeouts, circuit breakers, per-provider bulkheads, fallback, controlled queue drain |

### Failure modes

| # | Failure mode | Impact | Detection | Primary mitigation | Owner |
| --- | --- | --- | --- | --- | --- |
| F1 | Retry storm: a provider slows, callers hold connections, retries at several layers multiply load, pools exhaust and healthy paths fail (root cause of both past regional incidents) | Region-wide CUJ-1 and CUJ-2 failure | Retry ratio, pending requests per upstream, provider p99 latency, journey burn rate | One retry layer under a 20% budget, timeouts below caller deadlines, breakers, bulkheads | Service owner (config), SRE (mesh defaults) |
| F2 | Bad deployment or configuration change (the most frequent cause of incidents in most services) | Any CUJ | Canary analysis on journey SLIs | Progressive delivery with automated rollback, error budget release gate | Service owner |
| F3 | AZ impairment, including partial packet loss | Elevated errors in one AZ | Per-AZ SLIs, ARC zonal autoshift signals | ARC zonal shift, capacity sized to lose one AZ | SRE platform |
| F4 | Aurora writer failure | CUJ-1 write errors during failover | RDS events, write error rate | Multi-AZ failover, RDS Proxy holds client connections through failover, client retry under budget | SRE platform |
| F5 | MSK consumer lag or poison message | CUJ-3 delayed | Consumer lag, DLQ depth | Lag-based scaling, retry and DLQ topics, redrive runbook | Service owner |
| F6 | Provider outage | CUJ-3 delayed | Breaker state, provider canary | Fallback provider or park in retry topic, controlled drain on recovery | Service owner + vendor manager |
| F7 | Region impairment | All CUJs | Synthetics from two external vantage Regions, burn rate, AWS Health | Warm standby and ARC Region switch (Task 4) | SRE (execution), Incident Commander (decision) |
| F8 | Thundering herd on recovery: clients and queues retry at once | Re-failure after recovery | Request rate vs baseline, 429 rate | API Gateway throttling with Retry-After, rate-limited queue drain | Service owner |

### Ownership

Each journey has one accountable product owner; SRE is accountable for the platform and the reliability standards, not for other teams' services.

| Area | Product team | SRE | Platform engineering | Engineering leadership |
| --- | --- | --- | --- | --- |
| Journey SLOs and error budget spend | A, R | C | I | I |
| Service code, timeouts, idempotency, fallbacks | A, R | C | I | I |
| Mesh resilience defaults and guardrails | C | A, R | R | I |
| EKS, MSK, Aurora operations | I | A | R | I |
| DR plan, Region switch drills | R (app validation) | A, R | R | I |
| Provider fallback and contracts | A, R | C | I | C |
| Error budget policy decisions | R | R | I | A |

A = accountable, R = responsible, C = consulted, I = informed.

## SLI/SLO specification and error budget policy (Task 1)

At 99.99% over a rolling 30 days the error budget is 4.32 minutes of full outage, or 1 failed request in 10,000. That budget is too small to absorb even one regional failover at the full 45-minute RTO (about 86% of the 52.6-minute annual budget), so availability must come from in-Region redundancy and fast detection, with Region switch as the last resort.

### SLI and SLO table

| SLI | Journey | Good event | Valid event | Source (AWS) | SLO, 30-day |
| --- | --- | --- | --- | --- | --- |
| Availability | CUJ-1 | Response not 5xx and not 429 | All requests except other client 4xx | CloudFront real-time logs via Kinesis Data Streams, aggregated per CUJ into CloudWatch metrics each minute (alerting); standard logs in S3 via Athena (monthly reporting) | 99.99% |
| Latency | CUJ-1 | Completed under 800 ms at the edge | All successful requests | CloudFront logs (time-taken) | 99.9% |
| Availability | CUJ-2 | Response not 5xx and not 429 | As CUJ-1 | CloudFront logs | 99.99% |
| Latency | CUJ-2 | Completed under 300 ms at the edge | All successful requests | CloudFront logs | 99.9% |
| Synthetic journey success | CUJ-1, 2 | Canary run passes all steps | All canary runs | CloudWatch Synthetics, both Regions, every minute | 99.99% (corroborates low-traffic periods) |
| Fulfilment freshness | CUJ-3 | Provider-confirmed within 5 minutes of acceptance | All accepted transactions | Outbox and result timestamps, metrics in Amazon Managed Service for Prometheus | 99.9% |
| Recovery point (internal) | Critical data | AuroraGlobalDBRPOLag under 60 s | All 1-minute periods | Amazon CloudWatch, standby Region | 99.9% of minutes |
| Provider health (dependency) | CUJ-3 | Call succeeds within its timeout | All provider calls | Istio egress metrics in Amazon Managed Service for Prometheus | Tracked against each contract SLA; not a customer SLO |

Design choices behind the table:

- **Measured at the edge.** API Gateway logs miss failures at CloudFront, DNS and WAF; customers do not.
- **429 counts as bad.** Load shedding protects the system but the customer still failed, so it spends budget honestly.
- **Latency is a separate 99.9% SLO.** A combined 99.99% "fast and successful" target would page on normal tail latency.
- **Providers get dependency SLIs, not customer SLOs.** A provider at 99.9% cannot carry a 99.99% journey, which is why CUJ-3 is asynchronous and has fallbacks.

### Error budget policy

The policy is graduated so a single bad week redirects effort before it stops delivery. Freezes apply to the owning service only, never the whole organization.

| Budget consumed (rolling 30 days) | Action | Effect on delivery |
| --- | --- | --- |
| Under 50% | Normal operation; standing reliability allocation | About 80% feature work, 20% reliability work per team |
| 50% to 75% | Reliability items move to the top of the owning team's backlog | Features continue through canary deployment |
| 75% to 100% | Only low-risk changes ship (behind feature flags, off by default); open postmortem actions come first | Feature throughput reduced |
| Over 100% | Release freeze for the owning service except fixes and security patches | Lifted when the burn rate stays under 1.0 for 7 days and postmortem actions are scheduled |

Two guardrails keep the policy fair: a freeze needs the budget exhausted on both the 30-day and 90-day windows, so one incident does not stop a quarter's delivery; and any incident that consumes over 25% of the 30-day budget requires a blameless postmortem within 5 business days. Exceptions need joint approval from the product director and the SRE lead, recorded in the decision log.

## Observability and resilience architecture, part 1: observability (Task 2)

The goal is to cut time to detect customer impact to under 5 minutes while cutting pages: alerts fire on journey error budget burn, and component signals become diagnostics rather than pages.

### Telemetry standards

- **One standard:** OpenTelemetry, collected by AWS Distro for OpenTelemetry (ADOT) running as an EKS add-on.
- **Required attributes on every signal:** service.name, service.version, deployment.environment, cloud.region, cloud.availability\_zone, cuj.id, and the idempotency key on write paths.
- **Trace context end to end:** W3C traceparent from API Gateway into EKS, through Istio sidecars, into MSK message headers so consumers continue the same trace, and onto provider calls.
- **Logs:** structured JSON carrying trace\_id and span\_id, so any log line opens its trace.
- **Metrics:** RED (rate, errors, duration) per service, per route and per provider, plus saturation (pending requests, retry ratio, breaker state, consumer lag).

### Signal pipeline, storage and retention

| Signal | Collection | AWS store | Retention | Behaviour and trade-off |
| --- | --- | --- | --- | --- |
| Service and mesh metrics | ADOT scraping app and Envoy endpoints | Amazon Managed Service for Prometheus | 150 days default | High-cardinality PromQL without running Prometheus; cost grows with active series, so labels are reviewed |
| SLO aggregates | Recording rules and Athena jobs | Amazon CloudWatch metrics | 15 months (rolled up) | Year-on-year SLO history at low cost; lower resolution after 15 days |
| Traces | ADOT with tail sampling: all errors and slow traces, 5% of the rest | AWS X-Ray | 30 days | Keeps every failing trace while controlling cost; rare successful paths may be unsampled |
| Application logs | Fluent Bit on EKS | CloudWatch Logs, exported to S3 | 30 days hot, 1 year in S3 with lifecycle to archive | Fast queries for recent incidents, cheap audit retention |
| Edge logs | CloudFront standard logs | S3, queried with Athena | 13 months | Source of the journey SLIs |
| Change events | CloudTrail organization trail, EventBridge deployment events | S3 and CloudWatch | 1 year minimum | Correlates incidents with changes (failure mode F2) |

### Correlation and service maps

- **Service map:** CloudWatch Application Signals and the X-Ray trace map show EKS services, Aurora, DynamoDB, MSK and provider calls with live error and latency, so responders see where a failure is spreading.
- **Journey dashboards:** Amazon Managed Grafana joins the CloudFront SLIs, Prometheus metrics, X-Ray traces and business signals on one page per CUJ, with deployment markers from EventBridge.
- **Trade-off:** two consoles (Application Signals for services, Grafana for journeys) is one more tool, accepted because Grafana is the only place the edge SLIs and business signals meet.

### Synthetic checks

| Canary (CloudWatch Synthetics) | What it does | Frequency | Runs from |
| --- | --- | --- | --- |
| CUJ-1 write | Submits a test transaction for a synthetic tenant, with an idempotency key, through the public CloudFront endpoint | 1 minute | Two Regions other than the primary |
| CUJ-2 read | Reads the synthetic transaction status | 1 minute | Same |
| Standby origin | Calls the standby Region's API Gateway directly, bypassing the failover record | 1 minute | Primary and a third Region |
| Provider sandbox | Calls each provider's sandbox endpoint through Network Firewall egress | 5 minutes | Each Region |

The standby origin canary is what makes DR readiness continuous instead of quarterly. Synthetic tenants are excluded from business signals so tests do not distort revenue metrics.

### Business signals

Transactions accepted per minute, fulfilment completions per minute and provider confirmation rate are emitted as CloudWatch metrics using the embedded metric format. CloudWatch anomaly detection compares each against the same time last week, which catches failures that return HTTP 200, such as a silent drop in completions.

### Alert routing

Pages fire on multi-window, multi-burn-rate alerts against each journey SLO, following the [Google SRE workbook](https://sre.google/workbook/alerting-on-slos/) method:

| Alert | Condition (99.99% SLO) | Budget consumed | Action |
| --- | --- | --- | --- |
| Fast burn | Burn rate 14.4 over 1 hour and 5 minutes (error rate above 0.144%) | 2% in 1 hour | Page product on-call, SRE secondary |
| Slow burn | Burn rate 6 over 6 hours and 30 minutes | 5% in 6 hours | Page product on-call |
| Budget trend | Burn rate 1 over 3 days and 6 hours | 10% in 3 days | Ticket to the owning team |
| Platform saturation | Breaker open, consumer lag, RPO lag above 60 s | Not budget based | Page SRE platform on-call if it threatens an SLO, otherwise ticket |

Routing path: Amazon Managed Service for Prometheus alert manager and CloudWatch alarms publish to Amazon SNS, which feeds the organization's paging tool; AWS Health events reach the incident channel through EventBridge. AWS Systems Manager Incident Manager is not used because it closed to new customers on 7 November 2025. Cause-based alerts such as CPU, memory and pod restarts are demoted to dashboards or tickets, with a target of at least 50% fewer pages and at least 80% of pages actionable within 30 days.

## Observability and resilience architecture, part 2: resilience (Task 3)

The two regional incidents were retry storms, so the design rule is that load from a failing dependency must stay bounded: one retry layer, a 20% retry budget, deadlines that shrink at each hop, and shedding before saturation.

### Service mesh choice: Istio on EKS

AWS App Mesh reached end of support on 30 September 2026 ([AWS notice](https://docs.aws.amazon.com/app-mesh/latest/userguide/concepts.html)), so it is not an option. AWS positions Amazon VPC Lattice as a migration path ([AWS blog](https://aws.amazon.com/blogs/containers/migrating-from-aws-app-mesh-to-amazon-vpc-lattice/)); Lattice removes sidecars and adds IAM-based service authorization, but it does not provide retry budgets or outlier detection, which are the controls failure mode F1 needs.

Istio runs on EKS with Envoy sidecars and integrates with AWS-native identity (IAM Roles for Service Accounts), telemetry (ADOT to Amazon Managed Service for Prometheus and X-Ray) and load balancing (AWS Load Balancer Controller). The trade-off is that the team operates the Istio control plane: upgrades are kept within one minor version of upstream, rehearsed in staging, and sidecars add CPU and memory overhead per pod. Istio ambient mode, which removes sidecars, is reviewed after the 90 days. VPC Lattice remains an option for future cross-VPC or cross-account service connectivity, as a complement rather than a replacement.

### Controls by layer

| Control | Where | Setting | Behaviour | Trade-off |
| --- | --- | --- | --- | --- |
| Timeouts and deadlines | API Gateway integration, Istio VirtualService, app clients, Aurora | CUJ-1: API Gateway 5 s, service route 4 s, Aurora statement 2 s, provider call 2 s; remaining deadline propagated in a request header | Each hop gives up before its caller does, so no work continues for a caller that has gone | Tight timeouts fail slow-but-valid requests; values set from measured p99.9 plus margin and reviewed monthly |
| Retries | Caller's Istio sidecar only; library and SDK retries disabled for service-to-service calls | Up to 2 retries on connect failure, reset, 503; jittered backoff 25 ms to 250 ms; non-idempotent requests retried only with an idempotency key | One layer means retries add, never multiply | Developers lose control of retries in code; documented as a platform standard |
| Retry budget | Envoy cluster circuit\_breakers.retry\_budget, applied with an Istio EnvoyFilter (VirtualService sets attempts but not budgets) | budget\_percent 20, min\_retry\_concurrency 3 | Once retries exceed 20% of active requests, further retries fail fast | EnvoyFilter is version-sensitive; covered by an upgrade test in staging |
| Retries to AWS APIs | AWS SDKs | Adaptive retry mode, 3 attempts | Client-side rate limiting backs off under throttling | Separate dependency from service calls, so not a stacked layer |
| Circuit breaking | Istio DestinationRule outlier detection | 5 consecutive 5xx, 10 s interval, 30 s base ejection, at most 50% of endpoints ejected | Unhealthy endpoints leave the pool; the 50% cap avoids ejecting everything | A noisy but healthy pod can be ejected briefly |
| Bulkheads | Istio DestinationRule connection pools, one per provider and per critical upstream | Max connections, pending requests and concurrent requests per provider | A slow provider exhausts only its own pool, not the service | Limits must be re-tuned when traffic grows |
| Provider fallback | Application, with breaker state from the mesh | Interchangeable providers fail over to each other; otherwise work is parked in an MSK retry topic and the customer sees "pending" | CUJ-1 still returns success while CUJ-3 degrades gracefully | Fulfilment delay counts against the CUJ-3 freshness SLO |
| Idempotency | API Gateway requires an Idempotency-Key header on POST and PUT; DynamoDB conditional write (attribute\_not\_exists) with 24-hour TTL; Aurora unique constraint on client transaction ID | Keys also forwarded to providers that support them | Retries, client resubmits and replays cannot create duplicates | DynamoDB Global Tables are last-writer-wins, so after an unplanned Region switch the Aurora constraint is the final guard and reconciliation covers provider-side duplicates |
| Backpressure | Envoy pending-request limits, MSK consumers | Consumers pause partitions while a provider breaker is open; scaling on consumer lag (KEDA reading Amazon Managed Service for Prometheus) capped at each provider's rate limit | Load waits in MSK, where it is durable, instead of in memory | Fulfilment latency rises during provider trouble |
| Queue recovery | MSK retry topics and DLQ topic | Retry after 1, 5 and 30 minutes; DLQ after 3 attempts; topic retention 7 days; producer idempotence on; offsets committed after processing | Poison messages leave the main path; a redrive runbook replays the DLQ at a controlled rate after a fix | DLQ needs an owner and a daily check |
| Load shedding | AWS WAF rate-based rules; API Gateway usage plans and method throttling (token bucket rate plus burst) returning 429 with Retry-After; AWS AppConfig feature flags | CUJ-2 and non-critical routes throttled before CUJ-1 writes; brownout flags switch off non-essential features | Overload becomes fast rejections instead of slow failures for everyone | Rejected requests still spend error budget, by design |
| Outbox relay | Aurora outbox table, relay service to MSK | Event and transaction committed together | No lost or phantom events when MSK or the relay fails | Extra write per transaction |

### Capacity strategy

- **Survive one AZ:** each tier runs across 3 AZs with enough capacity to lose one at peak, which means about 150% of peak in total. HPA minimum replicas are sized for this, and topology spread constraints keep pods even across AZs.
- **Compute:** Karpenter provisions EKS nodes across several instance types to reduce the risk of capacity shortfall in one type.
- **Data:** Aurora writer plus two readers across 3 AZs, kept under 60% CPU at peak; MSK with brokers spread evenly across 3 AZs, replication factor 3, min.insync.replicas 2 and storage auto scaling; DynamoDB on-demand.
- **Standby Region:** compute at 20% of production, with On-Demand Capacity Reservations for the node count needed to reach 100%, so scale-out does not compete for capacity while many customers fail over at once. The data tier runs at full size because the Aurora secondary must carry production load the moment it is promoted.
- **AZ impairment:** ARC zonal shift (with zonal autoshift enabled) moves traffic away from an impaired AZ for the load balancer and EKS, without a Region switch.
- **Verification:** quarterly load tests with the Distributed Load Testing on AWS solution at 2x forecast peak, plus scale-out timing in every DR drill.

## DR runbook and game-day plan (Task 4: regional recovery)

Recovery uses a warm standby Region and a scripted ARC Region switch plan, targeting 30 minutes from decision to recovered traffic so the 45-minute RTO holds with margin, and an RPO measured in seconds against a 5-minute limit.

### Strategy and trade-offs

| Option | RTO | RPO | Why chosen or rejected |
| --- | --- | --- | --- |
| Pilot light | 1 hour or more | Seconds | Rejected: building compute during an event risks the 45-minute RTO |
| **Warm standby (chosen)** | Under 45 minutes | Seconds | Compute at 20% scales out in minutes; data tier already replicated at full size |
| Active-active | Minutes | Near zero | Rejected for now: Aurora has one writer, so active-active writes need application-level conflict handling that would consume most of the 90-day delivery capacity |

### Failure detection and authority

- **Signals that start the clock:** fast-burn alerts on CUJ-1 or CUJ-2; CloudWatch Synthetics failing from two external vantage Regions; AWS Health events for the primary Region.
- **First response is in-Region:** ARC zonal shift, rollback of recent changes and load shedding. A Region switch is considered when these have not restored service within 15 minutes, or when impact is projected to exceed 30 minutes.
- **Authority:** the Incident Commander recommends; one named approver (Head of SRE, with two named delegates) authorizes. If no approver answers within 5 minutes during a SEV-1, the Incident Commander is pre-authorized to proceed. The decision and its timestamp go in the decision log.
- **Readiness:** ARC Region switch plan evaluation runs continuously and the standby origin canary runs every minute. ARC readiness checks are not used because they are closed to new customers and AWS recommends Region switch plan evaluation instead ([AWS notice](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-readiness-availability-change.html)).

### DR runbook: unplanned Region switch

| Step | Target time | Action | Evidence |
| --- | --- | --- | --- |
| 0 | T+0 | Detection: fast-burn page | Alarm history |
| 1 | T+5 | Incident Commander declares SEV-1 and starts the decision clock | Decision log |
| 2 | T+15 | Decision point: approver authorizes the ARC Region switch plan, executed from outside the impaired Region | Region switch execution record |
| 3 | T+15 | Pre-checks: plan evaluation green, standby canary green; record current AuroraGlobalDBRPOLag as the expected data loss window | Screenshot and metric value in log |
| 4 | T+16 | Stop writes in the primary if it is reachable: pause outbox relay and MSK consumers, set API Gateway maintenance responses through AppConfig | AppConfig deployment ID |
| 5 | T+17 | Scale standby EKS to 100% (raise HPA minimums; Karpenter uses the capacity reservations) | Node and pod counts |
| 6 | T+18 | Promote data: if the primary is reachable, Aurora Global Database switchover (no data loss); if not, managed failover with data loss accepted, which promotes the standby cluster to writer | RDS event IDs |
| 7 | T+25 | Messaging: standby consumers start on the MSK Replicator target cluster using translated consumer group offsets; producers repointed through AppConfig | Consumer lag graph |
| 8 | T+25 | DynamoDB Global Tables are already writable in the standby Region; confirm writes succeed | Test write |
| 9 | T+27 | Dependency validation (list below) | Checklist signed in log |
| 10 | T+30 | Traffic shift: ARC routing control switches the Route 53 api-origin record to the standby API Gateway; CloudFront keeps serving with a 60-second origin DNS TTL | Routing control state change |
| 11 | T+35 | Verify: synthetic canaries pass, burn rate falling, business signals recovering | Dashboard snapshot |
| 12 | T+45 | RTO limit. Declare recovered; record measured RTO and RPO | Incident timeline |

### Dependency validation checklist

- Provider 1 and Provider 2 allowlist the standby Region's NAT Gateway Elastic IPs (registered in advance and re-tested in every drill).
- TLS certificates for the API and origin exist in AWS Certificate Manager in the standby Region.
- Secrets Manager replica secrets and the KMS multi-Region key decrypt correctly in the standby Region.
- Network Firewall FQDN allowlist in the standby Region matches the primary (managed as code).
- WAF web ACL, API Gateway throttles and authorizer configuration match the primary (deployed from the same templates).
- Provider sandbox canary passes from the standby Region.

### Reconciliation after an unplanned switch

Transactions committed in the primary during the replication lag window (normally seconds, alarmed at 60 seconds, limited to 5 minutes) may be missing from the promoted database.

1. Preserve the former primary's data before it is rebuilt as a secondary: snapshot it as soon as the Region is reachable.
2. Compare that snapshot with the new primary by client transaction ID over the lag window.
3. Replay missing transactions through the normal API using their original idempotency keys, so nothing is applied twice.
4. Compare provider records (provider confirmation IDs) with local records to find work that reached a provider but not the database; reverse or confirm with the provider.
5. Contact affected customers if any transaction outcome changed, and record totals in the postmortem.

### Failback

Failback is a planned change in a low-traffic window, at least 24 hours after the original Region is healthy:

1. Re-add the original Region to the Aurora Global Database as a secondary and wait until replication lag is under 1 second.
2. Reverse MSK Replicator direction and confirm consumer offsets are in sync.
3. Scale the original Region's EKS to 100%.
4. Run Aurora Global Database switchover (no data loss), then the Region switch plan in reverse.
5. Return the former standby to 20% compute after 24 hours of stable SLIs.

Writes run in one Region at a time, so traffic moves back in one step after switchover rather than gradually; canaries and burn-rate alerts gate each step.

### Game-day plan

Every experiment runs through AWS Fault Injection Service (FIS) with CloudWatch alarm stop conditions tied to the journey burn rate, so an experiment halts itself if real customers start to suffer.

| Game day | Method | Environment and frequency | Pass criteria |
| --- | --- | --- | --- |
| AZ loss | FIS AZ availability power interruption scenario | Staging monthly; production quarterly | ARC zonal shift completes, CUJ SLIs stay within SLO, no page above SEV-3 |
| Retry storm reproduction | Istio fault injection adds 2 s latency to a provider at 2x peak load | Staging monthly until it passes 3 times, then quarterly | Retries stay at or under 20% of requests, breakers open within 30 s, other services unaffected |
| Provider outage | Network Firewall rule drops traffic to one provider | Staging monthly; production quarterly in low traffic | Fallback or parking works, CUJ-1 stays within SLO, queue drains at the provider rate limit after recovery |
| Full Region switch | ARC Region switch plan executed end to end, including reconciliation | Staging twice in Month 3; first production run at day 90 in a low-traffic window, then quarterly | Measured RTO at or under 45 minutes (target 30), measured RPO at or under 5 minutes, zero duplicate transactions after reconciliation |
| Failback | Planned switchover back | After each Region switch drill | Completed with no data loss and no SLO breach |
| Decision tabletop | Facilitated exercise with the Incident Commander pool and approvers | Quarterly | Decision made within 15 minutes of the scenario start |

Each drill produces evidence for the assessment trail: the FIS experiment report, the Region switch execution record, measured RTO and RPO, and a short findings report whose actions enter the roadmap backlog.

## Incident command model (Task 5)

Severity is set by customer impact measured against the error budget, not by which component broke, so a SEV-1 is declared within 5 minutes of a fast burn instead of after a fixed outage duration.

### Severity criteria

| Severity | Criteria (any one) | Response | Communication cadence |
| --- | --- | --- | --- |
| SEV-1 | Fast-burn alert on CUJ-1 or CUJ-2 sustained for 5 minutes; journey success under 99%; suspected data loss or integrity risk; Region switch under consideration | Page Incident Commander, owning team and SRE within 5 minutes; executive liaison informed within 15 minutes | Status page and stakeholders every 15 minutes; executive summary every 30 minutes |
| SEV-2 | Slow-burn alert; partial degradation of one journey; CUJ-3 freshness SLO breached; one provider down with fallback active | Page owning team on-call; Incident Commander joins on request | Stakeholders every 60 minutes |
| SEV-3 | Redundancy lost with SLIs intact: AZ shifted away, breaker open with fallback holding, DLQ growing | Ticket for the owning team, handled in business hours | Daily update in the ticket |

Any responder can raise the severity; only the Incident Commander lowers it, with the reason in the decision log.

### Roles

| Role | Filled by | Responsibilities |
| --- | --- | --- |
| Incident Commander (IC) | Trained rotation drawn from SRE and senior product engineers | Owns the incident, sets priorities, makes or escalates decisions including Region switch recommendation; does not debug |
| Operations lead | Owning team's on-call engineer | Directs technical investigation and mitigation; runs runbooks |
| Communications lead | Product or support duty manager | Status page, customer and stakeholder updates on the cadence above |
| Scribe | Any trained engineer not on operations | Keeps the timeline and decision log in real time |
| Provider liaison | Vendor manager or owning team lead | Opens and tracks provider escalations under contract terms |
| Executive liaison (SEV-1) | Engineering director on call | Business decisions, customer commitments, Region switch approval if delegated |

### Decision log and evidence

- **Channel:** a bot opens one incident channel per incident in the organization's chat tool and pins the timeline document.
- **Decision log:** each entry records time, decision, who decided, options considered and the evidence used. Region switch decisions also record the approver and the measured AuroraGlobalDBRPOLag at the time.
- **Immutable record:** chat messages can be edited, so at close the timeline, decision log and channel export are written to an Amazon S3 bucket with S3 Object Lock in compliance mode, retained for at least one year.
- **Evidence collected automatically:** CloudWatch alarm history, Grafana dashboard snapshots, X-Ray traces for failing requests, CloudTrail change events, ARC routing control and Region switch execution history, FIS experiment reports, and AWS Health events.

### Escalation

| Trigger | Escalate to |
| --- | --- |
| Page not acknowledged in 5 minutes | Secondary on-call, then the team's engineering manager at 10 minutes |
| SEV-1 unresolved at 30 minutes | Head of Engineering and executive liaison |
| AWS service suspected | AWS Support case at the highest severity the support plan allows, with the incident ID |
| Provider suspected | Provider's contractual escalation path, opened by the provider liaison |
| Region switch considered | Named approver per Task 4, with the pre-authorization rule after 5 minutes |

### Handover

Handover happens at shift change or when an IC has led for 4 hours. The outgoing IC writes a handover note in the timeline (current impact, active hypotheses, actions in flight, next decision and its deadline, open escalations), then holds a 10-minute call with the incoming IC. The incoming IC posts "I have command" in the channel and the scribe logs the time.

### After the incident

SEV-1 and SEV-2 incidents, and any incident that spends over 25% of a 30-day budget, get a blameless postmortem within 5 business days. Actions are tracked in the owning team's backlog with an owner and due date, and overdue actions are reviewed in the monthly reliability review (Task 6).

## Prioritized 90-day roadmap (Task 6)

The 90 days are ordered by risk to the journey SLOs: Month 1 fixes detection and stops retry amplification, Month 2 builds platform resilience, Month 3 proves recovery. Product teams commit 20% of sprint capacity to roadmap items, so feature delivery continues at about 80%.

### How items are prioritized

Each item is scored before it enters the backlog:

- **Risk score:** likelihood (incidents in the last 12 months, game-day findings) × impact (estimated error budget minutes lost per occurrence). The two retry-storm incidents and the untested DR path score highest.
- **Toil saved:** engineer hours per week removed, from pager data (pages per week, share actionable) and a two-week toil log kept by SRE and on-call engineers in weeks 1 and 2.
- **Order:** highest risk first; among similar risks, highest toil saved first.

### Backlog

| Month | Item | Problem fixed | Risk | Toil saved | Owner |
| --- | --- | --- | --- | --- | --- |
| 1 | Name an owner per CUJ, sign the RACI and the error budget policy | Ownership | High | Low | Engineering leadership |
| 1 | Publish CUJ SLIs and SLOs; build edge SLI pipeline and Grafana journey dashboards | Detection | High | Medium | SRE + product teams |
| 1 | Burn-rate alerts live; demote cause-based alerts | Detection, alert noise | High | High | SRE |
| 1 | CloudWatch Synthetics canaries for CUJ-1, CUJ-2, standby origin and providers | Detection, DR readiness | High | Low | SRE |
| 1 | Retry hotfix: disable stacked library retries, set timeouts, API Gateway throttles with Retry-After | Retry storms | Very high | Medium | Product teams |
| 1 | Baseline metrics: MTTD, MTTR, pages per week, toil hours | Governance | Medium | Low | SRE |
| 2 | Istio rollout to all services with retry budgets, outlier detection and per-provider bulkheads | Retry storms | Very high | Medium | SRE platform |
| 2 | Idempotency keys on all writes; Aurora outbox; MSK retry and DLQ topics with redrive runbook | Data integrity, queue recovery | High | High | Product teams |
| 2 | Provider fallbacks and AppConfig brownout flags | External dependencies | High | Low | Product teams |
| 2 | Retry storm game day in staging (FIS and Istio fault injection) | Retry storms | High | Low | SRE |
| 2 | Start standby Region build as code: VPC, EKS at 20%, Aurora Global Database, MSK Replicator, DynamoDB replica | DR | Very high | Low | Platform engineering |
| 3 | Complete standby Region; dependency validation (provider allowlists, certificates, secrets) | DR | Very high | Low | Platform engineering |
| 3 | ARC Region switch plan with plan evaluation; reconciliation tooling | DR | Very high | Medium | SRE |
| 3 | Two staging Region switch drills plus failback; fix findings | DR | Very high | Low | SRE + product teams |
| 3 | First production Region switch drill at day 90 in a low-traffic window, after two clean staging runs | DR | Very high | Low | SRE, approved by engineering leadership |

### Day-90 success metrics

| Metric | Baseline | Day-90 target |
| --- | --- | --- |
| Time to detect customer-impacting failure | Measured in weeks 1 and 2 | Under 5 minutes |
| Pages per on-call week | Measured in weeks 1 and 2 | At least 50% fewer; at least 80% actionable |
| Retry share of requests during a provider slowdown | Unbounded (caused two incidents) | At or under 20%, shown in game day |
| Measured RTO and RPO | Never tested | RTO at or under 45 minutes (target 30), RPO at or under 5 minutes, in 2 staging and 1 production drill |
| CUJs with SLO, owner and error budget tracking | 0 of 3 | 3 of 3 |
| SRE time on operational work | Measured in weeks 1 and 2 | At or under 50%, trending to 40% |

### Governance: keeping SRE from becoming an operations dumping ground

- **Engagement model:** SRE supports a service only after a production readiness review: an SLO, an owner, runbooks, dashboards and alerts that pass the burn-rate standard. Until then the product team carries the pager alone.
- **Pager ownership:** product teams are primary on-call for their services; SRE is primary for the shared platform (EKS, Istio, MSK, Aurora) and secondary for services.
- **Toil cap:** SRE operational work is capped at 50% of team time, measured weekly from tickets and pages. If the cap is exceeded for 2 consecutive weeks, the excess work goes back to the owning team until it is automated.
- **Recurring work rule:** any manual task done more than twice a month gets an automation ticket, or is returned to its owner.
- **Monthly reliability review:** product, SRE and engineering leadership review budget spend per CUJ, top risks, toil trend and overdue postmortem actions, and re-rank the backlog.
- **Quarterly reset:** the risk and toil scores are recalculated from the quarter's incidents and game days, and the next 90-day plan is agreed with product leadership so reliability work competes openly with features.

## Mandatory architecture diagram (AWS)

The diagram shows the AWS reference workload as one active primary Region and one warm standby Region, with every control from Tasks 2 to 4 placed where it acts and tagged \[P1\] to \[P4\] against the scenario problems.

![Tier 1 digital service: AWS multi-Region reliability architecture](diagrams/tier1-aws-architecture.png)

&#91;image: Tier 1 digital service: AWS multi-Region reliability architecture\]

A full-resolution copy (4800 x 3600 px) is attached separately as tier1-aws-architecture.png so the labels stay readable when the document is printed or exported to PDF.

| Requirement | Where it appears on the diagram |
| --- | --- |
| Trust boundaries | Boundary 0: internet (customers and both providers); Boundary 1: AWS global edge; Boundary 2: private VPC per Region; Boundary 3: isolated data subnets with no route to the internet |
| Traffic flow | Solid arrows: live path CloudFront + WAF → Route 53 origin record → API Gateway → VPC Link → internal NLB → EKS → RDS Proxy, Aurora, MSK, DynamoDB; egress only through Network Firewall; outbox relay from Aurora to MSK; MSK retry and DLQ topics; CloudFront real-time logs to Kinesis Data Streams for edge SLIs. Dashed arrows: standby path after an ARC switch. Thick blue arrows: asynchronous replication |
| Failure domains | Global edge, each Region, and three Availability Zone lanes inside each VPC |
| Security controls | AWS Shield Advanced, AWS WAF, TLS 1.2+, JWT authorizer, private VPC Link, Security Groups, Istio mTLS, IAM Roles for Service Accounts, RDS Proxy IAM auth, MSK TLS + IAM, KMS encryption, Secrets Manager, VPC endpoints, Network Firewall FQDN allowlist |
| Key service names | CloudFront, AWS WAF, Route 53, ARC Region switch and routing controls, API Gateway, EKS, Istio, NLB, Aurora Global Database, RDS Proxy, Amazon MSK and MSK Replicator, DynamoDB Global Tables, CloudWatch Synthetics, ADOT, Amazon Managed Service for Prometheus, X-Ray, Managed Grafana, AWS Fault Injection Service, AWS AppConfig, Kinesis Data Streams (CloudFront real-time logs) |

Scenario tags: \[P1\] slow detection of customer impact, \[P2\] retry storms, \[P3\] DR never exercised, \[P4\] unclear reliability ownership.

## Validation, limitations and references

### How each claim is validated

| Claim | Evidence | AWS source of evidence |
| --- | --- | --- |
| 99.99% journey availability | Monthly SLO report per CUJ | CloudFront logs aggregated into CloudWatch metrics; Grafana SLO dashboards |
| Detection under 5 minutes | Time from first failed request to page, per incident | CloudFront real-time logs vs SNS publish time |
| RPO under 5 minutes | Continuous lag record and drill measurements | AuroraGlobalDBRPOLag in the standby Region ([AWS metric reference](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html)) |
| RTO under 45 minutes | Measured time from decision to recovered traffic in drills | ARC Region switch execution history, synthetic canary results |
| Retry storms contained | Retry share stays at or under 20% under injected provider latency | FIS experiment reports, Istio metrics in Amazon Managed Service for Prometheus |
| Load shedding works | Throttled requests return 429 quickly at 2x peak | Distributed Load Testing on AWS runs, API Gateway metrics |

### Known limitations

- **Cost:** the standby Region runs compute at 20% plus a full-size data tier and capacity reservations, an estimated third or more of production infrastructure cost (a rough estimate, to be confirmed with the AWS Pricing Calculator). Cost controls already in the design are tail-based trace sampling, Prometheus label review, log lifecycle to archive, and a 20% compute baseline instead of full duplication.
- **Unplanned failover loses the replication lag window:** normally seconds, always under the 5-minute RPO, but reconciliation (Task 4) is still required.
- **A regional failover is expensive against 99.99%:** one switch at the full 45-minute RTO uses about 86% of the annual error budget, so the design depends on in-Region resilience and fast detection.
- **Istio is customer-operated:** it is not an AWS managed service, so upgrades and sidecar overhead are the team's responsibility (Task 3 explains why it is still the right choice).
- **Providers bound CUJ-3:** fulfilment freshness cannot be better than the providers allow; fallbacks reduce but do not remove this.
- **Region pair:** service availability for Aurora Global Database, MSK Replicator and ARC Region switch must be confirmed for the chosen Regions before build.

### References

- [AWS App Mesh end of support notice](https://docs.aws.amazon.com/app-mesh/latest/userguide/concepts.html)
- [Migrating from AWS App Mesh to Amazon VPC Lattice (AWS Containers blog)](https://aws.amazon.com/blogs/containers/migrating-from-aws-app-mesh-to-amazon-vpc-lattice/)
- [Getting started with Istio on Amazon EKS (AWS Open Source blog)](https://aws.amazon.com/blogs/opensource/getting-started-with-istio-on-amazon-eks/)
- [ARC readiness check availability change and Region switch recommendation](https://docs.aws.amazon.com/r53recovery/latest/dg/arc-readiness-availability-change.html)
- [AWS Systems Manager Incident Manager availability change](https://docs.aws.amazon.com/incident-manager/latest/userguide/incident-manager-availability-change.html)
- [Amazon CloudWatch metrics for Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraMonitoring.Metrics.html)
- [Alerting on SLOs, Google SRE workbook](https://sre.google/workbook/alerting-on-slos/)

* [Region switch in ARC (AWS documentation)](https://docs.aws.amazon.com/r53recovery/latest/dg/region-switch.html)
* [Choosing a Service Mesh on EKS (third-party comparison of VPC Lattice and Istio)](https://barkingiguana.com/writing/choosing-a-service-mesh-on-eks/)
* AI assistance: Claude (Anthropic) and Gemini (Google), as disclosed under Assumptions and exclusions
