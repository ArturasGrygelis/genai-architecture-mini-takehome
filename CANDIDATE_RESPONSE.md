# Candidate Response

## Your Name

Artūras Grygelis

---

## Section A: Top 5 Risks (Ranked)

1. **No role-based SQL enforcement** — JWT role is passed into the system but never enforced. A SupportAgent can query `contact_email` or `mrr` and Snowflake will execute it. Violates the role access matrix and customer DPAs.

2. **Debug Logger persists raw PII** — Full Snowflake result sets (including `contact_email`, `mrr`) are stored 30 days in a shared logging cluster with likely weaker access controls than the source database.

3. **Unauthenticated admin endpoint with published URL** — `POST /admin/rerun_query` has no access control and its URL is explicitly documented in the design doc. Anyone on the internal network can replay any prior query with zero effort.

4. **Hallucination is the designed fallback** — The spec states the system will "estimate reasonable values based on industry benchmarks" with no user-visible indicator. Decisions get made on fabricated numbers.

5. **Prompt injection via RAG ordering** — RAG chunks are placed above the system prompt. Adversarial text in any indexed document can override system instructions before they are evaluated.

---

## Section B: Mitigations

1. Three role-scoped Snowflake accounts enforced at the DB layer: SupportAgent — SELECT on aggregate views only; Analyst — events/sessions with `contact_email`/`mrr` masked via Dynamic Data Masking; Executive — Metrics API only, SQL fallback disabled. Orchestrator picks credentials from verified JWT before any query.

2. Log metadata only: query ID, role, table names, row count, latency. Append-only audit log, security team access only. Replace prompt logs with template ID + parameter hash.

3. Require `ops-admin` JWT on every `/admin/rerun_query` call. Replay must re-check caller permissions — reject if the caller cannot access the original data.

4. Remove estimation fallback entirely. Return "I don't have data for this" with an optional dashboard link. Never fabricate metrics.

5. Reorder: `[System prompt] → [RAG context] → [History] → [User query]`. Wrap RAG in `<doc>...</doc>` and instruct the LLM to treat it as reference only, never as instructions.

---

## Section C: First Architecture Change

**Change:** Enforce RBAC at the Snowflake credential layer, not the LLM layer.

**Rationale:** This is the only risk exploitable with zero effort and legally non-negotiable under GDPR and customer DPAs. A DB-layer fix is a hard guarantee — no jailbreak bypasses a database permission.

---

## Section D: Clarifying Questions

1. **What privileges does the Snowflake service account hold?** If it has full read access, PII exposure is already live in production for every role right now.

2. **Who can read the logging cluster, and is it under the same DPA controls as Snowflake?** Shared observability stacks typically have much broader access — making the logger an exfiltration path. No session revocation is implemented — if a user's role is downgraded, their JWT (and elevated access) stays valid for up to 24 hours.

---

## Section E: Success Metrics (Optional)

1. **Role violation rate:** SQL queries per day accessing tables or columns beyond the requesting user's role permissions — target: 0.
2. **PII presence in logs:** Automated scan of log entries for email address patterns or MRR-shaped values — target: 0 matches per 30-day retention window.
3. **Fabrication rate:** Percentage of responses flagged as hallucinated by adding a structured output field  which has binary output values 'hallucinated' | "real", llm checks that by relevance of an answer, was retrieved information used correctly — target: 0% estimated responses served to users.

---

*End of response.*
