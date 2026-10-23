# Config Management Workflow

Turn governed security and operational standards into reusable Ansible automation — from baseline documents through AI-augmented playbook generation to scheduled execution, verified assets and evidence-backed governance.

**In one line:** Governed intent lives in GitHub, actual state is collected by a scan playbook, drift is what the comparison between the two reports in ClickHouse, and drift is fixed automatically by a parameterized remediation playbook — unless an Archer finding covers that asset and setting.

## Core principles

- **The scanner does not produce findings.** Executing the Config Scan Playbook collects raw actuals. Drift is reported only by comparing those actuals against expected state.
- **Expected and actual live in different places, and meet in ClickHouse.** Expected values live in GitHub as machine-readable baselines. Scan results (actuals) land in the Data Store. Drift is evaluated where the two can be joined.
- **No per-drift playbook generation.** Two playbooks are generated per platform, once: a scan playbook and a remediation playbook. Remediation at runtime is a parameterized execution of an already-approved playbook, not new code and not a new approval.
- **No per-drift human approval gate.** The human-in-the-loop step is playbook onboarding into Ansible Automation Platform (AAP). Once a playbook is onboarded, drift fixes run automatically, subject only to the Archer exception.
- **The Data Store is not evidence.** The Data Store (ClickHouse) is just the store and the compare point. Evidence is the evaluation output — drift records and compliance results — held on the KPIs / tracking / governance side.
- **Every result stays traceable to its baseline version.** A drift record carries the baseline commit it was measured against, so "the asset moved" stays separate from "the yardstick moved" when a baseline changes.

## The workflow at a glance

Govern → generate → execute → compare → remediate → verify.

1. Governed documents establish configuration intent.
2. Security and operational baselines express that intent as machine-readable expected state, per platform, in GitHub.
3. AI-augmented generation produces two playbooks per platform from the baseline.
4. The scan playbook runs on a cadence through AAP against assets; actuals land in the Data Store.
5. ClickHouse compares expected vs actual and materializes drift records.
6. KPIs / tracking / governance holds the evidence and the exception register.
7. On drift, the remediation loop invokes the remediation playbook with parameters against the drifted asset — unless an Archer finding is in place.
8. A rescan verifies the fix and closes the drift item.

## Stages

The canvas carries 10 stages. Stages 4A and 4B are the two generation outputs; the 4B playbook appears again downstream where the remediation loop invokes it — it is the same playbook, carrying its 4B identity, not a separate stage or box.

| Stage | Name | What it does |
|---|---|---|
| 01 | Document Governance Process | Source of authority. Approves and maintains configuration intent before automation begins. |
| 02 | Security & Operational Baselines | One merged baseline per platform: TSS secure-state + Configs operating-state, as YAML on GitHub, PR-approved. |
| 4A | Config Scanning — generation | AI-augmented generation produces the Config Scan Playbook, per platform. Read-only collection. |
| 4B | Config Remediation — generation | AI-augmented generation produces the Config Remediation Playbook, per platform. Parameterized. |
| 05 | Scan execution | Runs the scan playbook on its governed cadence — e.g. daily at 5 AM — with a defined scope and window. |
| 06 | Ansible Automation Platform | Execution plane. Runs only the approved playbook version, against the authorized scope and window. |
| 07 | Assets | Target scope. Receive governed configuration, and are the assets whose actual state is scanned. |
| 08 | Data Store — ClickHouse | Expected vs actuals. Scan results land here; drift is compared here. |
| 09 | KPIs / Tracking / Governance | Evidence side. Measures posture, progress, control and exceptions. |
| 4B | Config Remediation Playbook — invocation | Invoked with parameters: drifted setting + expected value. Connects directly to 07 Assets. |

There is no separate Ansible Playbooks box and no separate Remediation Loop box on the canvas, and no standalone Config Scanner box. Scanning is the execution of the 4A playbook, and the remediation loop is the route from 09 to 4B to 07 — not a stage of its own.

### Stage notes

**01 — Document Governance Process.** Maintains authoritative configuration requirements and routes approved content into the baseline. Only governed baseline content moves into playbook generation; draft or superseded intent stays outside the execution path. Output: approved baseline inputs with clear ownership and lineage.

**02 — Security & Operational Baselines.** Security Baselines (TSS) and Operational Baselines (Configs) are merged into one baseline per platform — not two separate boxes. Each entry states an explicit, testable expected value the generation stages can consume directly.

**4A / 4B — AI-augmented playbook generation (stage 04).** Generation interprets the baseline statements once, at generation time. AI assists authoring; both playbooks are reviewable artifacts. See "The two playbooks" below.

**05 — Scan execution.** Binds the scan playbook to its scope and cadence (e.g. daily at 5 AM) and hands the timed run to AAP. Missing scope, timing or approval keeps a run outside the executable path.

**06 — Ansible Automation Platform.** Launches the approved playbook, monitors task-level progress, and retains run status, output and exceptions.

**07 — Assets.** Configured, failed, skipped or unreachable status is recorded per target. Targets outside the authorized scope are excluded; failed or unreachable assets stay visible as exceptions.

**08 — Data Store.** See "Drift evaluation" below.

**09 — KPIs / Tracking / Governance.** See "KPIs, control and exceptions" below. This is also where the remediation loop is triggered from: the drift rule sits on the line from 09 to 4B.

## Baselines in GitHub

Expected state is governed input, held as code — not as an edited document.

| Aspect | Decision |
|---|---|
| Location | GitHub, per platform |
| Format | YAML — machine-readable config |
| Layout | One merged baseline per platform: security and operational entries together (e.g. `baselines/<platform>/security.yaml` + `operational.yaml`) |
| Change flow | GitHub pull request approval flow, per platform; required reviewers from security and platform ownership |
| Versioning | Version controlled; every change is a reviewed commit |
| Why YAML | PR-readable diffs, comments for rationale, Ansible-native, JSON Schema validation in CI |

Each baseline entry needs, from day one:

- the expected value, in a canonical form that matches how the scan reports the actual
- the comparison operator (not just a value — equals, at-least, one-of, and so on)
- the scope it applies to, by asset profile
- severity
- SLA target for remediation

### Publishing expected state

On merge, expected state is published from GitHub into ClickHouse. Every published row is stamped with the baseline commit SHA / baseline version it came from. Drift is then evaluated inside ClickHouse rather than across two stores at query time.

A baseline registry supports this: baseline id, product / platform, owner, version, and review due date.

## The two playbooks

Stage 04 generates exactly two playbooks per platform. Both carry their generation identity downstream wherever they appear in the flow.

| Playbook | Generated at | Behaviour |
|---|---|---|
| Config Scan Playbook (4A) | Stage 04 | Read-only. Reads each baselined setting on an asset and returns its actual value. Changes nothing, judges nothing. Executing it **is** the scanning. |
| Config Remediation Playbook (4B) | Stage 04 | Parameterized. Takes asset, drifted setting and expected value as inputs, and returns only that setting to its expected value. Compliant settings are untouched. |

**Bring your playbooks (4A enabler).** A platform-owned existing execution procedure can be brought into the governed cycle instead of using a generated one — the yellow 4A card on the canvas ("4A · Bring Your Playbooks / Ansible · Execution" on the standalone page; "4A · Config Scan Playbook" on the governance site). Scan results flow into the Data Store from this yellow 4A box.

**Onboarding into AAP.** Playbooks enter AAP only through a human-in-the-loop playbook onboarding process. A playbook is onboarded once. After that, invoking it — including per-drift remediation with different parameters — needs no new onboarding and no new approval.

## Drift evaluation — in ClickHouse (stage 08)

The Data Store is ClickHouse. It holds both sides of the comparison:

- **Expected:** published from the GitHub baselines on every merge, stamped with the baseline commit.
- **Actuals:** raw scan results, landed from the yellow 4A scan playbook execution against assets.

Per scan cycle, drift is a join in ClickHouse: the latest actual per asset and setting against the expected value for that asset profile and setting, using the operator and severity from the baseline entry.

The join materializes **drift records**:

| Field | Meaning |
|---|---|
| Asset | The asset that drifted |
| Setting | The configuration setting that drifted |
| Expected | Value from the baseline |
| Actual | Value collected by the scan |
| Severity | From the baseline entry |
| Baseline version | Commit SHA the expected value was measured against |
| First seen | When this drift was first detected |

Materialized drift records feed KPIs and governance (stage 09) and open one remediation item per drift.

Because every drift record carries its baseline version, reporting separates:

- **Drift by asset change** — the asset moved away from an unchanged baseline.
- **Drift by baseline change ("drift by decree")** — the baseline moved, so previously compliant assets now report drift.

## The remediation loop

The loop is orchestrated as a **Temporal workflow — one instance per drift item**. It is drawn on the canvas as a route, not a box: **09 → 4B → 07 Assets**, and back through rescan.

The rule, carried as a comment on the line from 09 to 4B:

> If drift is detected, the remediation playbook kicks off — unless an Archer finding is in place.

### On drift detection

1. **Notify the app custodian.** Always.
2. **Check Archer.** If an active Archer finding exists for that asset + that setting, do not auto-remediate. The drift stays tracked as excepted (stage 09 exception register).
3. **Otherwise, auto-remediate.** Invoke the 4B Config Remediation Playbook through AAP with parameters — asset, drifted setting, expected value. Only the drifted setting is changed.
4. **Rescan to verify.** The 4A scan playbook runs again on the asset.
5. **Close or loop.** If the rescan shows the setting at its expected value, the drift item closes. If not, the item stays open and the failure is visible for follow-up.

There is no human approval gate per drift item — the same Archer-exception pattern as Patch Journey stage 07. Human judgement sits earlier, at playbook onboarding, and later, in exception handling.

## Baseline governance

The Document Governance Process (stage 01) governs the baselines themselves, not just the documents behind them.

**Annual review cycle.** Baseline owners are notified to review and update their base configuration. Reviews are attested, and overdue reviews escalate. Baseline currency is a tracked KPI (see below).

**New baseline requirements.** A governed intake for new requirements — for example, a new product:

request → assess → approve → pull request into GitHub

**Propagation of a baseline change.** A merged baseline change triggers, in order:

1. Expected state republished to ClickHouse under the new commit.
2. Ansible playbooks regenerated from the updated baseline, with a human review gate on the playbook diff.
3. Regenerated playbooks re-onboarded to AAP through the human-in-the-loop onboarding process before they execute.

Baseline-governance measures: review completion % and overdue aging, intake cycle time, propagation lag (merge to republished expected state / regenerated playbooks), and yardstick-drift volume (drift caused by baseline change).

## KPIs, control and exceptions — stage 09

Stage 09 holds the evidence: drift records, compliance results and run outcomes. The Data Store holds the data; stage 09 is where it becomes posture, progress, control and exceptions.

| Measure | What it shows |
|---|---|
| Compliance rate | Share of asset / setting checks at expected state |
| Open drift, aging & SLA burn-down | How much drift is open, how old it is, and how much SLA remains |
| Drift by severity | Open drift grouped by the severity in the baseline entry |
| Time to detect | Scan cadence to drift record |
| MTTR | Drift detection to verified closure |
| Auto-remediation rate | Share of drift fixed by the loop without manual action |
| Remediation success rate | Fixes verified by rescan, vs attempted |
| Recurrence rate | Settings that drift again after closure |
| Scan coverage & blind spots | Assets / settings scanned vs in scope, and what is never reached |
| Baseline currency | Baselines within their review cycle vs overdue |
| Exception load | Open exceptions, by owner and age |

**Control & exceptions.** Every drift item that cannot auto-remediate is tracked as an exception with an owner and an expiry / review date — including Archer-excepted drift, accepted risk, and failed remediations. No exception stays open without an owner and a review date. Governance views distinguish verified outcomes from missing, failed or exempt results.

## Data foundation

| Dataset | Holds |
|---|---|
| Asset inventory | Assets in scope, their platform and profile |
| Expected state | Baseline entries published from GitHub, stamped per commit |
| Actuals | Raw scan results per asset and setting, per scan cycle |
| Evaluations & drift lifecycle | Drift records from first seen through verified closure or exception |
| Remediation / workflow records | Temporal workflow events, exported to ClickHouse and joined on drift id |
| Baseline registry | Baseline id, product / platform, owner, version, review due date |
| Exception register | Excepted drift, owner, reason (including Archer finding), expiry |

## Canvas terminology

The canvas legend names what the lines and marks actually are. It does not use "completed route" or "upcoming route."

| Term | Means |
|---|---|
| Governed input | Governed documents and baselines feeding the flow |
| Ansible automation | Execution through Ansible playbooks and AAP |
| AI-augmented | A step where AI assists generation (marked with a borderless star / sparkle) |
| Bring your playbook | An existing platform-owned playbook brought into the governed cycle (the yellow 4A card) |
| Remediation loop | The 09 → 4B → 07 drift-response route |
| Config scanning | The 4A scan playbook collecting actuals |
| Config remediation | The 4B remediation playbook fixing a drifted setting |
| Evidence | The evaluation output at stage 09 — not the Data Store itself |

## Key design decisions

| Decision | Settled position |
|---|---|
| What the scanner produces | Raw scan results, not findings. Drift comes from the comparison. |
| Where expected state lives | GitHub, as machine-readable YAML baselines, per platform, PR-approved. |
| Where drift is evaluated | In ClickHouse, as a join of published expected state vs latest actuals. |
| How many playbooks per platform | Two — one scan (4A), one remediation (4B) — generated once at stage 04. |
| How remediation fixes drift | Parameterized invocation of 4B (drifted setting + expected value). No runtime generation. |
| Approval model | No per-drift gate. Human-in-the-loop sits at playbook onboarding into AAP. |
| When remediation does not run | When an active Archer finding covers that asset + that setting. Drift is tracked as excepted instead. |
| How a fix is proven | Rescan with the 4A playbook; closure only on a verified expected value. |
| Loop orchestration | Temporal workflow, one instance per drift item. |
| Config Scanner as a product entry | Removed from Features on the governance site. Scanning lives inside this workflow as the 4A playbook — there is no separate scanner stage or menu entry. |

## What this workflow is not

- Not a findings engine: nothing in it labels a scan result a finding before the baseline comparison runs.
- Not per-drift code generation: no new playbook is written when drift is found.
- Not approval-per-fix: routine drift on an onboarded playbook remediates automatically, with the Archer exception as the only hold.
- Not a separate Config Scanner product surface: the scan is one playbook inside this governed lifecycle.
