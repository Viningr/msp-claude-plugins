# Compliance Drift Reporter — CIPP expansion

**Date:** 2026-05-19
**Status:** Approved design — ready for implementation plan
**Affects:** `wyre-technology/msp-claude-plugins` (Advanced Workflows docs + live routine)

## Summary

The batch-1 **Compliance Drift Reporter** routine (`trig_01KdNPXeYMep1SFEDEzCrPQV`)
currently reports only Liongard configuration-change detections. As WYRE rolls out
compliance baselines in CIPP, the routine should also report **baseline drift from
CIPP** — tenants failing their assigned CIPP Standards, plus a supporting Best
Practice Analyzer score per tenant.

This is an **expansion of the existing routine**, not a new one: one routine, one
weekly cadence, one combined Slack report covering both signals.

## Motivation

Liongard and CIPP report drift in fundamentally different shapes:

- **Liongard** — a stream of *change events* ("this config changed"). The current
  routine treats the week's detections as the drift signal.
- **CIPP Standards** — a point-in-time *pass/fail against a baseline* ("this tenant
  fails its assigned standard"). This is true baseline drift, the missing half the
  current doc's "Extending it" section already anticipates.

The current routine's docs explicitly name this as the next step: *"true baseline
pass/fail: the routine could evaluate each system against its compliance baseline
and flag failures, not just report that something changed."* CIPP Standards is
that baseline.

## Decisions (resolved during brainstorming)

| Question | Decision |
|---|---|
| New routine or expand existing? | **Expand the existing routine** — one combined report |
| Which CIPP signal? | **Both** — CIPP Standards compliance (primary) + Best Practice Analyzer score (supporting) |
| Tenant scope? | **All CIPP tenants** — tenants with no baseline assigned surface as an "unmanaged" finding |
| Fresh data or cached? | **Read last computed results** — read-only, no `cipp_run_standards_check` trigger; the routine stays a pure reporter |
| Report structure? | **Approach C** — two-section report (Liongard / CIPP) with a computed posture scorecard header |

## Scope of changes

Four artifacts change in `wyre-technology/msp-claude-plugins`:

| Artifact | Change |
|---|---|
| Live routine `trig_01KdNPXeYMep1SFEDEzCrPQV` | Add `cipp_list_tenants`, `cipp_list_standards`, `cipp_list_bpa` to the WYRE MCP Gateway connector's `permitted_tools`; replace the routine prompt with the two-phase version |
| `msp-claude-plugins/docs/src/pages/advanced-workflows/compliance-drift-reporter.astro` | Rewrite: cover both signals, new build prompt + routine prompt, new CIPP multi-tenant gotchas |
| `msp-claude-plugins/docs/src/pages/advanced-workflows/agent-routine-catalog.astro` | Update the `compliance-drift-reporter`/liongard row note; note CIPP posture coverage is folded in (touches the `security-posture-reviewer`/cipp row) |
| `CHANGELOG.md` | New `[Unreleased]` entry |

**Unchanged:** weekly cron `0 12 * * 1` (Monday 08:00 America/New_York), Slack
delivery (canvas + one-line summary to channel `C0931CKJ75X`), and the entire
Liongard collection logic.

## Run flow

The routine runs four phases per invocation:

### Phase 1 — Liongard collection (unchanged)

Compute a 7-day window. Call `liongard_detections_list` at `pageSize` 5, paginating
until 25 detections collected or `HasMoreRows` is false. Read only `ID`,
`SystemName`, `Name`, `Date`, `SystemType` per detection — never `ChangeDetection`
(5KB–44KB each). Record `Data.Pagination.TotalRows` as the true weekly total.

### Phase 2 — CIPP collection

1. `cipp_list_tenants` — the full tenant list. This is the denominator.
2. `cipp_list_standards` — which tenants have a standard assigned and their
   compliance state. A tenant present in the tenant list but absent here =
   **unmanaged** (no baseline assigned).
3. `cipp_list_bpa` — per-tenant Best Practice Analyzer score.
4. Classify each tenant: **pass** / **fail** / **unmanaged**; capture its BPA score.

Read only small per-tenant fields (tenant name/id, standard name, compliance
status, BPA score). Never read per-tenant config payloads — the same "read the
metadata, skip the bloat" discipline as Liongard's `ChangeDetection` rule.

### Phase 3 — Posture scorecard

Compute from numbers already collected:

- Configuration changes this week (Liongard `TotalRows`)
- Tenants failing CIPP Standards
- Tenants with no baseline assigned ("unmanaged")
- Average BPA score across tenants

### Phase 4 — Deliver

Publish a Slack canvas titled `Compliance Drift — <YYYY-MM-DD>` with the scorecard
header followed by two sections:

- **Configuration changes (Liongard)** — detections grouped by system, as today.
- **Baseline compliance (CIPP)** — per-tenant pass/fail table, plus a list of
  unmanaged tenants.

Post one summary line to `C0931CKJ75X` linking the canvas.

## Error handling — per-source degradation

With two independent data sources, "if it fails, stop" is wrong: one vendor's
outage should not blank a report the other half is fine to deliver. Silent partial
data is equally wrong. The rule: **degrade per-source, and make the gap visible in
the report itself.**

| Condition | Behavior |
|---|---|
| Liongard fails, CIPP OK | Deliver CIPP section; canvas notes *"Liongard data unavailable this run."* |
| CIPP fails, Liongard OK | Deliver Liongard section; canvas notes *"CIPP data unavailable this run."* |
| Both fail | Post a "needs a human" line to `C0931CKJ75X`, then stop (batch-1 behavior preserved) |
| Both OK, zero findings | Post a single summary line, skip the canvas (batch-1 behavior preserved) |

## Build & verification

The doc uses the established **one-shot build-prompt** pattern: a single prompt
pasted to Claude that confirms connectors, updates the routine, and verifies it
end to end.

**Shape probing.** The Liongard envelope (`Data.Detections` + `Data.Pagination`)
is known. The CIPP envelopes for `cipp_list_standards` and `cipp_list_bpa` are
**not yet verified**. The build prompt's Step 1 probes all three `cipp_*` tools
against a couple of real tenants and records the actual response shape; the
routine prompt's field names are finalized against that probe. No unverified
field names are baked into the live routine — the same approach the batch-1 doc
uses for Liongard.

**Build prompt steps:**

1. Confirm the gateway is reachable. Probe `liongard_detections_list` (known
   shape) plus `cipp_list_tenants`, `cipp_list_standards`, `cipp_list_bpa` —
   record envelope shapes and the per-tenant fields that distinguish
   pass / fail / unmanaged.
2. Confirm the Slack connector and destination channel `C0931CKJ75X`.
3. Update the existing routine `trig_01KdNPXeYMep1SFEDEzCrPQV`: add the three
   `cipp_*` tools to the gateway connector's `permitted_tools`; install the
   two-phase routine prompt.
4. Manual run + verify: the canvas has the scorecard header and both sections;
   the Liongard count reconciles with `TotalRows`; CIPP tenant counts
   (pass + fail + unmanaged) sum to the `cipp_list_tenants` total.

**Testing.** A scheduled routine has no unit-test harness — verification is the
manual run in Step 4, plus a follow-up run a week later confirming the cron fired
and the partial-degradation messaging renders correctly when a source is down.

## Known gotchas (new, CIPP-specific)

- **A 34+ tenant CIPP sweep can be large.** Read only the small per-tenant fields;
  never read per-tenant config payloads. Cap detail rows if a response is
  unexpectedly large, the same way the Liongard phase caps at 25 detections.
- **`permitted_tools` must list every `cipp_*` tool the routine calls.** A
  connector with an empty or incomplete `permitted_tools` list runs with no tools
  and silently does nothing.
- **The routine is read-only by design.** It must not call
  `cipp_run_standards_check` — that is a write-capable, minutes-long operation.
  The routine reads whatever CIPP last computed on its own schedule.

## Out of scope

- Triggering fresh CIPP standards checks.
- Correlating Liongard systems to CIPP tenants into a unified per-entity table
  (no shared key — fragile mapping; rejected as Approach B).
- Any remediation action (opening PSA tickets for repeat offenders) — noted as a
  possible future extension, not part of this work.
