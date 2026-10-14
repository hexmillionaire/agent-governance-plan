# Agent Governance Plan: Human-Gated Inquiry Triage Agent

A governance and safety specification for a small-business LLM agent that reads inbound client inquiries, scopes them, and drafts replies. The agent never sends, books, or commits to anything without a human approving it first.

**Author:** Amado ([@hexmillionaire](https://github.com/hexmillionaire)), founder of Avalon Digital Studio, Trinidad and Tobago
**Status:** Design-stage governance plan. Originally written as the capstone of Microsoft's *MS-AI-102: AI Agents for Everyone* ([verified certificate](https://courses.edx.org/certificates/378c38c17fd248ec95c9f70111a4cc2b)), then revised for public release. Business-specific details (pricing, client data, internal identifiers) have been removed.

---

## Why I'm publishing this

Most published agent-safety thinking assumes large organizations with dedicated risk teams. Most real agent deployments will happen in shops like mine: one or two people, off-the-shelf tooling, and no compliance department. This plan is my attempt to answer a practical question:

> What does responsible deployment of an LLM agent look like when the "governance team" is the founder and one co-owner?

The governing principle throughout is **the agent prepares, the human commits.**

---

## 1. The system

| Component | Choice |
|---|---|
| Orchestration | Power Automate (event-driven pipeline) |
| Reasoning | Copilot Studio (classification, scoping, drafting) |
| Sources | Intake inbox, a pricing doc, a project-history doc, calendar |
| Records | `leads` table and an append-only `agent_audit_log` table (Supabase) |
| Human gate | Power Automate Approvals (Approve / Edit / Reject) |

**Flow:** inquiry arrives → classify → scope against pricing and past projects → draft reply with proposed call slots → **human approval** → send, book, and log.

The orchestrator leads and the LLM supports, because the work is a multi-system pipeline with a mandatory approval gate, not a conversation.

---

## 2. Safety systems

### Scope boundaries

- **Topic:** new-project inquiries only. Support, contractual, and legal messages are auto-escalated, never handled.
- **Action:** read-only across all sources. On approval only, the agent may send one reply, create one calendar hold, write one lead record, and set one follow-up timer per inquiry. It cannot delete, change permissions or settings, add recipients, or send without approval.
- **Data:** a single scoped service account that can reach only the sources listed above.

### Guardrail configuration (extract)

```yaml
in_scope_categories:            [new_project_inquiry]
auto_escalate_categories:       [support, contractual, legal, out_of_scope]
max_emails_per_inquiry:         1
allowed_recipients:             original_sender_only   # no added TO/CC/BCC
price_quote_rule:               must_map_to_pricing_doc_row   # else BLOCK
out_of_band_quote:              hard_block_pending_override
capability_claim_rule:          must_appear_on_services_list
classification_min_confidence:  0.80    # below -> clarify or escalate
require_cited_comparable:       true    # no proposal without >=1 past project
follow_up_after_days:           5
max_followups:                  1
forbidden_actions: [delete_anything, modify_permissions, change_settings, send_without_approval]
audit_retention_months:         24
cold_lead_content_purge_months: 12
```

### Output validation

- **Format:** plain prose, no attachments, addressed only to the original sender, with two or three proposed slots whenever a call is offered.
- **Grounding:** every price band must map to a row in the pricing doc, and every capability claim must appear on the published services list. Ungrounded drafts are flagged, and out-of-band quotes are hard-blocked.
- **Confidence:** classifications below 0.80 route to clarifying questions or escalation. The agent asks rather than asserts. A scope that can't cite at least one comparable past project cannot produce a proposal.

### Audit trail

Each run logs the inbound metadata, the classification and its confidence, the matched tier and price band, the cited comparables, the draft, the approval decision (approve, edit, or reject, by whom, with any edits), and every executed write and escalation. Records go to an append-only table that is retained for 24 months. Raw inquiry content for cold leads is purged at 12 months. Service-role credentials live in a secret store, never inside a flow.

### Emergency stop

The founder, or the co-owner as delegate, can toggle the main flow off, which halts all processing immediately. A partial stop disables only the send action. Because in-flight items rest at the approval gate, **a stop can never cause an auto-send, and no inquiry is lost.** Unprocessed messages stay in the inbox.

Recovery: diagnose from the audit log → fix the cause → re-run held items against a test sample → re-enable.

---

## 3. Human-in-the-loop framework

Each action is assigned to a tier based on its stakes and how often it happens.

| Action | Tier | Rationale |
|---|---|---|
| Read sources | Autonomous | No external effect, high frequency |
| Classify a message | Autonomous | Internal label; errors are caught by escalation rules |
| Archive spam | Autonomous | Reversible (archived, not deleted) |
| Draft a reply | Autonomous | No effect until approved |
| Write or update a lead record | Autonomous | Internal and reversible |
| Mark a lead cold | Autonomous | Internal and reversible |
| Send templated vendor-pitch decline | Notify | Fixed template, not a prospect, logged |
| Escalate support or legal mail | Notify | A hand-off; the human then owns it |
| Send templated 5-day follow-up | Notify | Light touch on an already-approved thread |
| **Send qualified reply to a prospect** | **Approve** | Irreversible, brand-bearing, reads as a commitment |
| **Book discovery call** | **Approve** | Bundled with the reply approval |

**What the approver sees:** the original message, the classification and confidence, the matched tier and price band, the *cited comparable projects* that justify the scope, the full draft, the proposed slots, and any red flags (an out-of-band price or an ungrounded claim).

**Timeout is fail-safe by inaction.** With no decision after 4 business hours, the approver gets a reminder. At 24 hours, the item is held and the approver is pinged again. It never auto-sends. The safe default for an irreversible action is to do nothing.

**Dual approval:** proposals above the standard pricing band require both the founder (commercial owner) and the co-owner (technical and commercial reviewer).

---

## 4. Monitoring

| KPI | Category | Alert threshold | Cadence |
|---|---|---|---|
| Time to first qualified reply | Performance | Median > 2 business hrs over 7 days, or any inquiry unanswered > 4 hrs | Weekly + real-time outlier |
| Draft approve-as-is rate | Performance | < 70% in a week | Weekly |
| Escalation-miss / misclassification rate | **Safety** | **Any single miss (zero tolerance)**, or > 5% reclassified | Real-time / weekly |
| Ungrounded-claim rate | **Safety** | **Any one sent incident** | Real-time |
| Inquiry-to-call conversion | Impact | > 25% below the trailing 4-week average | Weekly |

The safety thresholds are deliberately tighter than the performance ones, because one bad send matters more than one slow week. The co-owner is named as safety co-owner, so the highest-consequence failures always get a second reviewer.

**Weekly review (10–15 min):** check exceptions (escalations and reclassifications), skim 3–5 approved drafts for quality and the edit rate, and compare the funnel week over week. Any real-time safety alert triggers action immediately; it doesn't wait for the weekly review.

---

## 5. Risks and compliance

### Risk 1: Exposure of prospect data (privacy and security)

Third-party personal data flows through an LLM and across several connectors.

**Controls:** a scoped service account, credentials kept in a secret store, inference kept inside the tenant data boundary (no training on content), a retention and purge schedule, the append-only audit log, and a DLP policy on the connectors.

**Residual risk:** the LLM step is still a third-party processing point. Changes to the model, its configuration, or the vendor's subprocessors could shift how data is handled, so the vendor's terms and the DLP posture stay under watch.

### Risk 2: A wrong or over-promising quote reaches a prospect (ethics and commercial)

The agent may confidently scope against the wrong comparable project, and a busy approver may wave it through.

**Controls:** the grounding gate, confidence routing, red flags shown on the approval card, the real-time ungrounded-claim KPI, and dual approval for large quotes.

**Residual risk: approval fatigue.** A human gate only works if the human actually reviews. I track the correction rate and time per approval as proxies for genuine review. If approvals start taking two seconds each, the gate has become theater.

### Compliance mapping

The most relevant requirement is data-protection law: the Trinidad and Tobago Data Protection Act for local prospects, and GDPR/UK-GDPR where applicable.

| Control | Principle satisfied |
|---|---|
| Mandatory human approval gate | No *solely* automated decisions; lawful basis is pre-contractual steps the prospect requested |
| Scoped service account | Data minimization and purpose limitation |
| 12-month purge + deletion on request | Storage limitation and right to erasure |
| Append-only audit log | Accountability |

*This is a design document, not legal advice.*

---

## 6. Known gaps and open questions

Writing this plan surfaced problems that I don't think are fully solved, either here or in general:

1. **Prompt injection from inbound email.** Every inquiry is untrusted text that goes straight into the model. An attacker could write an "inquiry" containing instructions meant to manipulate classification or the draft. Today the approval gate and the hard config limits (one recipient, original sender only, no deletes, grounded prices) cap the blast radius. But the plan doesn't yet include input sanitization, separating instructions from data, or adversarial test cases. That's the next addition.
2. **Approval fatigue is a scalable-oversight problem in miniature.** Human-in-the-loop is the backbone of this design, yet the human is also its weakest link under load. Proxies like edit rate and time per decision are crude. I'd like better ways to tell whether a reviewer is actually reviewing.
3. **Confidence scores aren't calibrated.** The 0.80 threshold assumes the model's confidence means something. I haven't validated calibration against labeled outcomes, so the threshold is a policy choice, not a measured one.
4. **Grounding checks verify *that* a claim maps to a source, not *whether* the mapping is right.** Picking the wrong comparable project passes validation, and only the human catches it.
5. **Small-team governance.** With two people, "separation of duties" is thin. I'd welcome thoughts on what minimum viable oversight looks like for solo founders deploying agents.

Feedback and issues are welcome.
