# RBC Operating Model Transformation Deck — Context Document

**Purpose.** This is the bootstrapping context for continuing work on Dhwanil's 62-slide RBC operating model transformation deck. Drop this entire document at the start of a new Claude session to restore full context. The deck is built with python-pptx and rendered as a Word/PowerPoint file plus rendered slide images via LibreOffice + pdftoppm.

The user is **Dhwanil**, Director-level technology leader at RBC running Technology Infrastructure (~2,500 FTE), owner of the Zephyr On-Prem IaC program (34 platforms). He's a Max plan user, direct, comfortable iterating heavily, expects analysis before execution on novel content but will say "go" or "looks good" as the build trigger.

---

## TL;DR — what to do at start of a new session

1. Read this entire document.
2. Ask Dhwanil to attach `Operating_Model_Transformation_RBC.pptx` (the current state of the deck) if he wants to keep iterating on the same artifact, or confirm he wants to start from scratch.
3. If iterating: set up the build environment (below), copy the pptx into the workdir, and announce what slide(s) he'd like to modify.
4. Always discuss before building novel content — Dhwanil values pre-build analysis. Once he says "go" / "yes" / "lets try it", build, render, verify, deliver.

---

## The argument the deck makes

This is a horizontal-operating-model proposal for RBC Technology Infrastructure. The core claim: dissolve four vertical functional towers (Architecture, Engineering, Operations, Intelligent Ops) into flat peer squads that each own a domain end-to-end with all four roles inside the squad. Then weave three cross-cutting peer squads — Zephyr Hybrid Cloud (Control Plane), Compliance, Optimization — across all capability squads.

**Story beats in order:**

1. **Problem.** Four towers, three handoff seams between them, costs measured (rework, MTTR, reactive Ops time).
2. **Solution.** Virtual pod removes the seams. Every role works on one backlog, same standup, one definition of done.
3. **Force multiplier.** Coding agents inside every pod — workflow (Issues In, PRs Out) plus full AI-augmented SDLC (Plan → Code → Test → Deploy → Monitor).
4. **Squads.** Database, Compute, Storage, Middleware, Network. Each contains technology pods (8-12 engineers each, full Eng/Dev/Ops/IntelOps mix).
5. **Cross-cutting squads.** Zephyr Hybrid Cloud ships the Control Plane (substrate). Compliance ships Zephyr Assurance. Optimization ships Zephyr Optimize. Pods publish IaC modules into the Control Plane; gates run as code.
6. **Bridge.** 90-day pilot with 3 squads, no org change. Wave 1 priority: stand up agent-assisted development.
7. **Target state.** Team Topologies framing. Seven tribes aligned to value streams. Communities of Practice preserve craft.
8. **Proof.** AWS / Netflix / JPMC case studies. Quantified benefits. Risks and mitigations.
9. **Decision.** Fund Wave 2 / formalize 3b reorg with evidence.
10. **Appendix.** Persona slides for stakeholders (internal only).

---

## Build environment

```bash
cd /home/claude/work
# Working file (gets modified)
ls updated.pptx
# Deliverable lives in /mnt/user-data/outputs/Operating_Model_Transformation_RBC.pptx
```

The main builder is `/home/claude/work/update_deck.py` — a ~5700-line python-pptx file with one builder function per slide plus shared helpers. To iterate:

```bash
# 1. Edit a builder function in update_deck.py
# 2. Rebuild target slide(s) in place with a small script:
python3 -c "
import sys
sys.path.insert(0, '/home/claude/work')
from update_deck import Presentation, Inches, SW, SH, clear_slide, build_<NAME>_slide
prs = Presentation('/home/claude/work/updated.pptx')
prs.slide_width = Inches(SW); prs.slide_height = Inches(SH)
clear_slide(prs.slides[<IDX>])
build_<NAME>_slide(prs.slides[<IDX>])
prs.save('/home/claude/work/updated.pptx')
"

# 3. Render to inspect:
python3 /mnt/skills/public/pptx/scripts/office/soffice.py --headless --convert-to pdf updated.pptx
rm -f slide-*.jpg
pdftoppm -jpeg -r 110 updated.pdf slide -f <N> -l <N>
# view slide-NN.jpg

# 4. Deliver:
cp updated.pptx /mnt/user-data/outputs/Operating_Model_Transformation_RBC.pptx
# Then use present_files
```

For inserting a new slide rather than rebuilding in place, the pattern is:

```python
blank_layout = prs.slide_layouts[0]
s = prs.slides.add_slide(blank_layout)  # adds at end
clear_slide(s)
build_<NAME>_slide(s)
# Then move to target position via OOXML manipulation:
sldIdLst = prs.slides._sldIdLst
current = list(sldIdLst)
last = current[-1]
sldIdLst.remove(last)
sldIdLst.insert(<TARGET_INDEX>, last)
```

---

## Design conventions (must follow)

### Dimensions
- Widescreen 13.333" × 7.5" (`SW = 13.333`, `SH = 7.5`)
- Layout 0 (blank) used for everything

### Colors (RGBColor)
```python
RBC_BLUE        = 0x003DA5
RBC_BLUE_DARK   = 0x002B7F
RBC_BLUE_DEEP   = 0x001F5C
RBC_BLUE_MED    = 0x1A55BC
RBC_YELLOW      = 0xFEDF01
RBC_YELLOW_SOFT = 0xFFF1A8
ICE_BLUE        = light blue used for muted text on dark backgrounds
SUCCESS         = 0x10B981
DANGER          = RGBColor(0xDC, 0x26, 0x26)
TEXT_DARK       = near-black
TEXT_MUTED      = mid-gray
BORDER          = light gray
LIGHT_BG        = near-white background tint
WHITE           = 0xFFFFFF
```

### Typography
- Headings: Georgia, bold
- Body: Calibri
- Eyebrow text (small label above title): all caps, char_spacing 3-4, RBC_BLUE, 9-11pt
- Title: Georgia 28pt bold — **must fit on one line**
- Footer: light gray "Operating Model Transformation | Technology Infrastructure"

### Standard slide chrome
Every content slide has:
- 0.08" yellow accent stripe at top (added by `add_header`)
- RBC logo top-right (~0.55" tall)
- Eyebrow + Title via `add_header(slide, eyebrow, title)`
- Footer "Operating Model Transformation | Technology Infrastructure"

### Section dividers
Full-bleed RBC_BLUE_DEEP background, left yellow stripe, logo top-left. Built via `build_section_divider_slide(slide, eyebrow, title_line1, title_line2, description, closing_note)`. Title splits across two lines: italic Georgia + bold Georgia yellow.

### Persona slides (appendix, internal only)
Built via `build_persona_slide(slide, config)`. Layout: avatar (initials in yellow circle) + name + role on left, italic reaction quote, yellow underlying-concern box, red-rail SPECIFIC BLOCKERS, green-rail HOW WE RESPOND on right.

### Cross-cutting squad bands
RBC_BLUE_DEEP background with yellow top stripe and yellow left stripe.

---

## Key helper functions in update_deck.py

- `add_header(slide, eyebrow, title)` — top yellow stripe, eyebrow, title, RBC logo top-right
- `add_rect(slide, x, y, w, h, fill, line_color=None, line_width=None)`
- `add_oval(slide, x, y, w, h, fill)`
- `add_text(slide, x, y, w, h, text, size=N, bold=, italic=, color=, align=, valign=, char_spacing=, font=)`
- `_set_char_spacing(run, value)` — character spacing on a run
- `clear_slide(slide)` — wipe a slide before rebuilding
- `build_section_divider_slide(slide, eyebrow, title_line1, title_line2, description, closing_note)`
- `build_persona_slide(slide, config)` — persona config has name, role, scope, initial_reaction, blockers (list of dicts with title/detail), underlying_concern, our_response (list of strings)

---

## Current 62-slide structure (deck v62)

| # | Eyebrow | Title |
|---|---|---|
| 1 | OPERATING MODEL TRANSFORMATION | From Vertical Silos (title slide) |
| 2 | EXECUTIVE SUMMARY | The Case for a Horizontal Operating Model |
| 3 | PART 1 · THE PROBLEM | Four towers, (divider) |
| 4 | THE CHALLENGE | Where Today's Vertical Model Creates Friction |
| 5 | THE SHIFT | Vertical vs. Horizontal — At a Glance |
| 6 | PARTS 2 & 3 · THE MODEL & THE BRIDGE | The model, (divider) |
| 7 | OPERATING MODEL EVOLUTION | Where the Handoffs Happen — and What Breaks |
| 8 | OPERATING MODEL EVOLUTION | How the Virtual Pod Removes the Handoffs |
| 9 | OPERATING MODEL EVOLUTION | Every Role Gains a Coding Agent |
| 10 | WORKED EXAMPLE | SQL Server Through the Five Stages |
| 11 | BRIDGE STEP | Why Virtual Cross-Squad — and Why Now |
| 12 | CROSS-SQUAD COMPOSITION | Why Ops Must Be in Every Cross-Squad |
| 13 | VIRTUAL SQUAD FORMATION | How Cross-Squads Are Built — No Org Change |
| 14 | SQUAD STRUCTURE | Squads Contain Pods — One Pod Per Technology (Database example) |
| 15 | POD ANATOMY | SQL Pod — One Technology, All Roles, One Team |
| 16 | SQUAD STRUCTURE | Compute Squad — Linux / Windows / Legacy pods |
| 17 | SQUAD STRUCTURE | Storage Squad — Block / Object-NAS / Backup-DR pods |
| 18 | SQUAD STRUCTURE | Middleware Squad — Web-App / Messaging / Integration pods |
| 19 | SQUAD STRUCTURE | Zephyr Hybrid Cloud — Ships the Control Plane |
| 20 | OPERATIONS TRANSFORMATION | How On-Call & Ops Work Differently |
| 21 | OPERATING MODEL CAPABILITY | Agent-Assisted Development (SECTION DIVIDER) |
| 22 | AGENT-ASSISTED DEVELOPMENT · THE WORKFLOW | Issues In, PRs Out — A Coding Agent in Every Pod |
| 23 | AI-AUGMENTED SDLC | Coding Agents Accelerate Pods Across the Lifecycle |
| 24 | OPERATIONS TRANSFORMATION | Ops Shifts Right — From Firefighting to Engineering |
| 25 | BRIDGE EXECUTION | Priorities for the 90-Day Pilot |
| 26 | CROSS-CUTTING SQUADS | Three Peer Squads Serve Every Capability Pod |
| 27 | COMPLIANCE SQUAD | Policy-as-Code · Assurance Automation |
| 28 | OPTIMIZATION SQUAD | FinOps · Performance · Capacity |
| 29 | ZEPHYR HYBRID CLOUD SQUAD | Ships the Zephyr Control Plane |
| 30 | INTERACTION MECHANICS | Pods ↔ Zephyr · Compliance · Optimization |
| 31 | PART 4 · TARGET STATE | Where this (divider) |
| 32 | TARGET OPERATING MODEL | One Squad, Full Lifecycle · Peer Squads |
| 33 | WORKED EXAMPLE | Where Does SQL Server Live? |
| 34 | FOUNDATIONAL FRAMEWORK | Four Team Types · Three Interaction Modes |
| 35 | STRUCTURE | Seven Tribes Aligned to Value Streams |
| 36 | TRANSFORMATION | Where the Four Verticals Land in the New Model |
| 37 | GOVERNANCE BY DESIGN | What Stays Vertical — Cross-Cutting Functions |
| 38 | PRESERVING CRAFT EXCELLENCE | Communities of Practice — the Horizontal Weave |
| 39 | PART 5 · PROOF & DECISION | Proven at scale, (divider) |
| 40 | PROVEN AT SCALE | Who Is Already Running This Model |
| 41 | PRESERVED BY DESIGN | What We're NOT Changing |
| 42 | DECISION RIGOR | Alternatives We Considered |
| 43 | WHAT IT FEELS LIKE | A Week in the Life — SQL Server Engineer |
| 44 | WHAT SUCCESS LOOKS LIKE | Year-One Scorecard — Month 12 |
| 45 | CASE STUDY · 1 OF 3 | Amazon / AWS — Two-Pizza Teams |
| 46 | CASE STUDY · 2 OF 3 | Netflix — Full-Cycle Developers + Paved Road |
| 47 | CASE STUDY · 3 OF 3 | JPMorgan Chase — Platform-as-Product |
| 48 | QUANTIFIED BENEFITS | What the Data Says — Realistic Targets |
| 49 | PROGRAM ACCELERATION | Powering Programs Already in Flight |
| 50 | TRANSITION ROADMAP | Three Waves Over 18–24 Months |
| 51 | RISK MANAGEMENT | Top Risks & How We Mitigate Each |
| 52 | HOW WE MEASURE SUCCESS | Leading & Lagging Indicators |
| 53 | DECISIONS REQUESTED | Recommended Next Steps |
| 54 | (closing) | From four towers to flow. |
| 55 | PART 6 · APPENDIX | For our team, (divider, internal only) |
| 56 | STAKEHOLDER PREP · INTERNAL ONLY | Jikin Shah — SVP, Technology Infrastructure |
| 57 | STAKEHOLDER PREP · INTERNAL ONLY | Rob Vanchanahem — SVP, Platform Services |
| 58 | STAKEHOLDER PREP · INTERNAL ONLY | Scott Stanley — Managing Director, Platforms |
| 59 | STAKEHOLDER PREP · INTERNAL ONLY | Sara O'Neil — Director, Database Operations |
| 60 | STAKEHOLDER PREP · INTERNAL ONLY | Paul Gage — Director, Compute Operations |
| 61 | STAKEHOLDER PREP · INTERNAL ONLY | Dan Considine — Director, Storage Operations |
| 62 | STAKEHOLDER PREP · INTERNAL ONLY | Ernani — Director, Zephyr Hybrid Cloud |

---

## Hard-won design decisions (don't re-litigate these)

These are settled decisions from previous iterations. Don't propose alternatives unless Dhwanil explicitly asks:

1. **Flat peer-squad model.** Every infrastructure capability owned by ONE persistent squad doing ALL: build IaC, engineer the platform, operate (SLO + on-call), build AI agents, automate.
2. **Squad contains Pods.** Each pod has Eng/Dev/Ops/IntelOps roles (8-12 engineers per pod).
3. **Terminology lock:** "AI/ML" → "Intelligent Ops" (long form); "AI" role chip → "IntelOps" (short form). But products like "AI agents" or "coding agents" stay as-is.
4. **"Zephyr Control Plane"** is the umbrella product the Zephyr Hybrid Cloud Squad ships. Replaces older terms like "Backstage."
5. **Zephyr Assurance** = Compliance Squad's flagship. **Zephyr Optimize** = Optimization Squad's flagship.
6. **On-call STAYS WITH OPS** but transformed: Ops in pod, backed by AI agents on first response.
7. **Bridge step is 3 platforms** (SQL Server, Kubernetes, Storage) in Q1, no org change.
8. **Zephyr Hybrid Cloud is the THIRD CROSS-CUTTING SQUAD** (peer to Compliance and Optimization). Led by Ernani. Not embedded in pods; ships the Control Plane that pods publish IaC modules into and consume primitives from.
9. **Architecture → Engineering handoff language softened** to system-level framing ("design context lives in one team, build context in another") — not blame-the-people framing.
10. **Slide 7 (Handoffs) shows 4 teams, 3 seams.** Compliance is NOT in the handoff chain — it's a peer cross-cutting squad, not a downstream consumer.
11. **Slide 8 (Pod removes handoffs)** has 4 yellow role chips with 3 green checks at former seam positions. Four outcome cards: SHARED CONTEXT · PARALLEL WORK · OPS UPLIFT · COMMON GOAL.
12. **Coding agents term retained** (Dhwanil specifically preferred this over "engineering agent" or "AI pair engineer"). Workflow: tickets in GitHub Issues / ServiceNow / PagerDuty / Slack get auto-created or assigned to coding agents for doc uplift, bug fix, or capability enhancement.
13. **Agent-Assisted Development section** = slides 21-23. Section divider (21) → workflow (22, "Issues In, PRs Out") → lifecycle (23, "Coding Agents Across the Lifecycle"). Eyebrows on the two content slides differentiate: "THE WORKFLOW" vs "AI-AUGMENTED SDLC."
14. **AI SDLC framing:** NOT "what industry is measuring" — instead "how coding agents accelerate Pods across the lifecycle." Five canonical stages: Plan/Code/Test/Deploy/Monitor with feedback loop. Three bands: pod context (top), pipeline (middle), cross-cutting squads supply runtime (bottom). No industry stats. Acceleration callout: "WHY IT COMPOUNDS: multi-week cycle → multi-day cycle."
15. **Slide 9 (Every Role Gains a Coding Agent)** is the operating-model-level emphasis on agentic dev: pod with role chips + "+ coding agent" green badges, plus 4 role-specific leverage cards (what does each role's coding agent do), plus AGENTIC LEVERAGE yellow callout.
16. **Slide 30 (Interaction Mechanics)** lane order: Zephyr Control Plane (top) → Compliance/Assurance (middle) → Optimization/Optimize (bottom). Matches the substrate-then-overlay conceptual hierarchy.

---

## Persona slide content (slides 56-62)

For reference if rebuilding personas. Each persona is a config dict for `build_persona_slide`. Schema:

```python
{
    'name': str,
    'role': str,
    'scope': str,  # 1-2 sentences on what they own
    'initial_reaction': str,  # italic quote, in voice
    'blockers': [
        {'title': str, 'detail': str},
        ...  # 3 blockers typically
    ],
    'underlying_concern': str,  # one sentence in yellow box
    'our_response': [str, str, str, ...],  # 3-4 responses
}
```

Personas in current deck: Jikin Shah (SVP, Tech Infrastructure), Rob Vanchanahem (SVP, Platform Services), Scott Stanley (MD, Platforms), Sara O'Neil (Director, DB Ops), Paul Gage (Director, Compute Ops), Dan Considine (Director, Storage Ops), Ernani (Director, Zephyr Hybrid Cloud).

---

## Visual conventions for common slide types

### Squad-contains-pods slide (slides 14, 16-19)
Uses `build_squad_pod_structure_generic(slide, config)`. Layout: squad container on left (squad name, OWNS bullets, ENG TOTAL pill at bottom, "contains" yellow arrow), 3 numbered pod cards on right with 2x2 role chip grid (Eng/Dev/Ops/IntelOps) and "8-12 engineers · end-to-end" footer.

Config dict:
```python
{
    'eyebrow': 'SQUAD STRUCTURE',
    'title': 'X Squad — One Pod Per Technology',
    'lead': str,
    'squad_name': str,
    'squad_name_size': int,  # optional, override if name is long
    'squad_subtitle': str,
    'squad_owns': [str, str, str, str],  # 4 bullets
    'squad_size': 'NN-NN ENG TOTAL',
    'pods': [
        {'num': '1', 'name': str, 'tech': str},
        ...  # 3 pods
    ],
}
```

### Squad deep-dive slide (slides 27, 28, 29)
Left card: THE SQUAD identity (name, headcount, role mix with yellow count chips). Right top: WHAT THEY SHIP with products. Right bottom: OUTCOMES strip with 4 outcome cards.

### Cross-cutting overview slide (slide 26)
Stacked horizontal bands top-to-bottom: Compliance band → arrows → capability squads middle band → arrows → Optimization band → arrows → Zephyr Hybrid Cloud band (substrate at bottom). Arrows describe what flows between layers.

---

## Communication style notes (for working with Dhwanil)

- **Pre-build analysis is mandatory for novel content.** When he asks "how should we add X?" do NOT just build. Discuss options first. He'll say "go" or "yes" or "lets try it" when he wants execution.
- **For tactical edits ("change this label", "reorder these"), just do it.** No need to over-discuss.
- **Push back when wrong.** He'll correct misreads bluntly — "I asked for X, not Y" — and expects you to recover quickly with the right thing. Don't grovel; just fix.
- **Numbers matter.** When he says "first 30-day plan", agent-assisted dev goes as the *first* bullet, not just inserted somewhere.
- **Don't over-format.** When he reviews a slide and says "Looks good", don't add 8 paragraphs of post-build explanation. A succinct delivery with the slide rendered is enough.
- **Keep deck coherent across sessions.** Terms (Intelligent Ops, IntelOps, Zephyr Control Plane, etc.) must stay consistent. Catch any drift from earlier renames.
- **Speed cycle:** edit builder → rebuild target slide in place → PDF convert → pdftoppm render → view → deliver. Don't run the full insert pipeline if you're just rebuilding a slide.

---

## Things that have been wrong before — watch out

1. **Don't confuse "Pod Anatomy" with "Squad Contains Pods."** Pod Anatomy = zoom-in on one pod showing 4 roles inside it (slide 15). Squad Contains Pods = a squad with 3 technology pods inside it (slide 14, 16-19). They look superficially similar but answer different questions.
2. **Don't put Compliance in the handoff chain.** Compliance is a peer cross-cutting squad, not a downstream stage. Handoffs are 4 teams / 3 seams.
3. **Don't add "AI/ML" anywhere.** Always "Intelligent Ops" (long) or "IntelOps" (short).
4. **Don't introduce industry stats into the AI SDLC slide.** Dhwanil rejected that framing explicitly. The slide is about how agents accelerate pods at each stage, not about industry trends.
5. **Title length.** Slide titles must fit one line. If they wrap, shorten.
6. **Loop arrows in pipelines** need clearance below the pipeline to draw cleanly — reserve at least 0.3" vertical space for the horizontal segment of the loop.

---

## How to ask the user good questions

When starting a new session, ask:

1. "What part of the deck do you want to work on?" — get target slide(s) or section.
2. If it's a modification: "What change do you want?" — get clear scope.
3. If it's a new slide: "Where should it sit, and what should it argue?" — get position + thesis.
4. For novel content: discuss approach, get approval, then build.
5. For tactical edits: just do it.

---

## Final reminders

- Deck is currently 62 slides (after AI SDLC arc + agentic dev emphasis + section divider).
- Living file: `/mnt/user-data/outputs/Operating_Model_Transformation_RBC.pptx`
- Builder: `/home/claude/work/update_deck.py` (only present in same session; if new session, you'll need Dhwanil to share or you'll need to rebuild key builders from this document).
- If you don't have the builder file in a new session, the cleanest path is: have Dhwanil upload the current .pptx, and you can do small modifications via python-pptx directly on the existing slides without needing the full builder library. For major new slides, you'd need to either reconstruct the builders or ask him to upload the builder file too.
- Render check after EVERY change. Don't trust code to be correct without visual verification.
- Deliver the .pptx via `present_files` after every meaningful change.

---

**End of context document.** This should be sufficient to bootstrap a new Claude session and resume deck work without repeating every design decision from scratch.
