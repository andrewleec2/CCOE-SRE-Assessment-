# AI Usage and Contribution Report: Tier 1 Reliability Assessment

Andrew Lee | 8 October 2026

## Summary

Andrew Lee directed the work, owned the decisions and validated the output; Claude did most of the technical drafting, verification and rendering; Gemini produced an earlier draft and two review passes, parts of which were adopted after verification. The assessment brief (scenario, constraints, tasks and deliverables) was supplied by the assessor, not authored by the candidate or by either AI.

| Item | Detail |
| --- | --- |
| Work produced | Tier 1 Reliability Assessment: AWS (submission document) and the mandatory architecture diagram (PNG and SVG) |
| Session window | 7 October 2026, 23:21 to 8 October 2026, 00:07 (Malaysia time, UTC+8) |
| AI tools | Claude Opus 5.5 (Anthropic), via the Claude app; Google Gemini, used separately by the candidate, whose outputs were pasted into the Claude conversation |
| Method | The candidate cross-checked each AI against the other and asked for source URLs before accepting factual claims |
| Basis of this report | Prepared by Claude from the Claude conversation record. Gemini's contribution is described only from the Gemini outputs pasted into that conversation; any other Gemini sessions are not visible here |

## Contribution breakdown

### User input (Andrew Lee)

- **Selected AWS** as the cloud and set the scope by supplying the assessor's scenario, constraints, six tasks with weights, and the seven required deliverables.
- **Provided the starting artifact:** an initial Mermaid architecture diagram with two Regions, three VPC tiers per Region, cross-Region replication and an ARC failover path. Its origin was not stated in the conversation.
- **Set the quality bar:** asked for validation against the scenario, a submission-ready rendered diagram, source URLs for every disputed claim, and a final check against each required deliverable.
- **Ran a cross-model review:** brought Gemini's review comments and full draft into the conversation and asked for each to be verified. This surfaced the App Mesh end of support date and the 45-minute RTO, which were then checked and applied.
- **Made the decisions:** chose to proceed with the corrected submission, to embed the diagram, and to apply the two final improvements (more detail in the diagram and headings that match the deliverables).

### Claude contribution (Anthropic, Claude Opus 5.5)

- **Reviewed the starting diagram** and identified errors: Route 53 placed as a traffic hop before WAF, API Gateway and DynamoDB drawn inside VPCs, ALB behind a REST API VPC Link, no Availability Zones, few security controls, and an incomplete standby Region.
- **Designed and rendered the final architecture diagram** as a hand-laid SVG and a high-resolution PNG. It shows trust boundaries 0 to 3, live, standby and replication paths, three AZ lanes per Region, security controls, a reliability control plane, and [P1] to [P4] scenario tags.
- **Verified service status against AWS documentation, with URLs:**
  - App Mesh end of support on 30 September 2026: confirmed.
  - Systems Manager Incident Manager closed to new customers on 7 November 2025: found and corrected.
  - ARC readiness checks closed to new customers, with Region switch recommended instead: found and corrected.
  - The Aurora lag metric for evidencing RPO is AuroraGlobalDBRPOLag, not AuroraGlobalDBReplicatedWriteIO: found and corrected.
- **Rejected the VPC Lattice substitution** with reasoning. Lattice lacks retry budgets and outlier detection, so swapping it in would have removed the controls the retry-storm scenario requires. Claude recommended Istio on EKS with an explicit justification instead.
- **Reviewed Gemini's full draft** and listed technical errors and constraint conflicts. These included weighted routing through ARC routing controls (they are on/off), pilot light confused with warm standby, stacked retries, a SEV-1 threshold slower than the error budget, an all-or-nothing deployment freeze that conflicts with the delivery-capacity constraint, and DR proven only by a tabletop exercise.
- **Wrote the submission document:** journey model, SLI/SLO tables, error budget arithmetic (4.32 minutes per 30 days; one 45-minute failover uses about 86% of the annual budget), graduated error budget policy, observability and resilience designs, a timed 12-step DR runbook, reconciliation, failback, game days, the incident command model, a scored 90-day roadmap, and a deliverables index.

### Gemini contribution (Google, as seen in pasted outputs)

| Contribution | Outcome in the final submission |
| --- | --- |
| Flagged AWS App Mesh end of support | Adopted after verification against the AWS notice |
| Recommended replacing App Mesh with Amazon VPC Lattice and removing Envoy sidecars | Not adopted: would remove retry budgets and outlier detection; Istio on EKS used instead |
| Framed reliability ownership as the root cause of the incidents | Partly adopted: stated as one of two causes alongside missing technical controls |
| Produced a full draft submission with all section headings and a Mermaid diagram | Structure informed the final document; content rewritten to fix retired services, technical errors and constraint conflicts; diagram replaced |
| Draft content adopted in substance | Idempotency keys stored in DynamoDB, warm standby rather than active-active, quarterly game days, a toil threshold for governance |

## Gemini's contribution statement, reviewed

Gemini's statement is useful in shape, but four of its eight claims overstate or misstate what happened. If it were submitted as written, an assessor comparing it with the brief or the final document would find the mismatches. The statement is reproduced below as the candidate received it, followed by an accuracy check of each claim.

### As provided by Gemini

**User Contributions (Human Engineering)**

- Defined the core Tier 1 digital service scenario and all strict mathematical constraints, including the 99.99% availability target, the 45 minute RTO, and the 5 minute RPO.
- Established the foundational synchronous and asynchronous routing logic across API Gateway, Amazon EKS, and Amazon MSK.
- Dictated the iterative pacing of the design process, mandating that each deliverable be mapped against a strict grading rubric before finalization.
- Provided the baseline application flow and deployment conditions that governed the disaster recovery strategy.

**AI Contributions (Gemini Assistance)**

- Translated conceptual routing and generic application flow into strict AWS native terminology and specific SRE governance policies.
- Identified the critical deprecation of AWS App Mesh and recommended the technical pivot to Amazon VPC Lattice to ensure current service compliance.
- Generated the required Mermaid architecture diagram code to visualize trust boundaries and failure domains.
- Structured and formatted the final technical deliverables into a formal Markdown document.

### Accuracy check

| Gemini claim | Assessment | Corrected wording |
| --- | --- | --- |
| User defined the scenario and the 99.99%, 45-minute RTO and 5-minute RPO constraints | Inaccurate: the assessor supplied these in the brief | Worked within the assessor-defined scenario and constraints |
| User established the synchronous and asynchronous routing logic across API Gateway, EKS and MSK | Overstated: the reference services came from the brief; the journey split (CUJ-1 to CUJ-3, with providers kept asynchronous) was drafted by AI and accepted by the user | Selected and approved the journey model and routing design |
| User mandated mapping each deliverable against the rubric before finalization | Accurate in substance | Directed validation of every deliverable against the brief before submission |
| User provided the baseline application flow and deployment conditions for DR | Partly accurate: the user provided the starting diagram; the RTO and RPO conditions came from the brief | Provided the initial architecture diagram and the assessor's recovery targets |
| Gemini translated the design into AWS-native terminology and SRE policies | Partly accurate: true of Gemini's draft, but the final text was rewritten after errors in it were found | Produced an earlier draft that informed the final structure |
| Gemini identified App Mesh deprecation and recommended the pivot to VPC Lattice | First half accurate; second half not adopted | Flagged App Mesh end of support; its VPC Lattice recommendation was assessed and rejected |
| Gemini generated the required Mermaid diagram | Not the submitted diagram: Gemini's diagram lacked the required security controls and was replaced | Produced a draft diagram, superseded by the submitted one |
| Gemini structured the final deliverables into a formal Markdown document | Inaccurate for the final version, which was written in Claude and exported from there | Produced a draft structure that informed the final headings |

## Disclosure statement

The following statement appears in the Assumptions and exclusions section of the submission.

> **AI assistance disclosure.** This submission was prepared with two AI assistants: Claude (Anthropic, Claude Opus 5.5) and Gemini (Google). I selected AWS, supplied the assessor's scenario and constraints, provided the initial architecture diagram, set the review criteria and made the final design decisions. Gemini produced an earlier draft and review comments, including the flag that AWS App Mesh reached end of support on 30 September 2026. Claude drafted most of the final text, produced the architecture diagram, and verified service status and metrics against AWS documentation, which led to replacing App Mesh with Istio on EKS, Systems Manager Incident Manager with CloudWatch and SNS alert routing, and ARC readiness checks with ARC Region switch plan evaluation. I cross-checked the two assistants against each other and against the cited sources (see References) before accepting any claim, and I take responsibility for the submitted content.

Sources used for verification are listed in the References section of the submission.
