# PRODUCT SPEC — Checklist-driven records assembly (v0)

## Business layer
**Customer:** Expert medical opinion (EMO) / second-opinion provider running a records team (80 people) and a nursing team (60 people) on Cora + SharePoint.
**Buyer (who pays/decides):** Derek (ops: throughput, cost per case, SLAs) with Alicia (nursing: rework, nurse time on non-clinical work) as co-decider. Poppy (Cora PM) is the gatekeeper on the AI intervention-rate bar.
**Business problem:** ~2-week turnaround, compounding backlog, ~30% rework. Two of the three drivers are inside our control: the up-to-2h/case manual page extraction and organization, and rework caused by missing records or miscategorization that isn't caught until nurse review. (The third, facility wait time, is largely outside software's control; we only shorten how long a gap goes unnoticed.)
**Business outcome:** Fewer records-team hours per case, lower rework rate, and gaps surfaced days earlier so cases don't bounce back after the facility wait has already been paid.

## User layer
**User problem:** Records team members spend hours per case hand-extracting pages from image-based faxes in Acrobat, and can't tell which pages are clinically relevant, so cases go to nurses incomplete or miscategorized and bounce back.
**Target user:** A records team member working a queue of 15–25 cases per shift, no clinical training, judged on errors, working from the nurse's checklist in Cora (e.g. "Mt Sinai Knee Center MRI on 12/10/25").
**Core job:** Fulfil the nurse's checklist for a case: confirm which requested records have arrived, pull the right pages, and see what's still missing, without reading every page.

## The link
If a records team member can fulfil the nurse's checklist by reviewing agent-matched pages instead of manually extracting them, then rework and cost per case fall because (a) categorization is done against the nurse's own spec rather than the team member's guess, (b) gaps are flagged the day a fax lands rather than at nurse review, and (c) the ~2h of Acrobat work becomes a review-and-accept task.

**One-sentence brief:** For a records team member working an incoming fax against a nurse's checklist, this matches pages to checklist items and flags what's missing, instead of manual Acrobat extraction. Derek and Alicia care because it should move hours per case and the rework rate via checklist-anchored matching and early gap detection.

## Success metrics
**User (leading):** Time from "documents received" to "sent to nurse review" per case; days from fax arrival to follow-up sent (gap lead time).
**Business (lagging):** Rework rate (cases bounced from nurse back to records); records-team hours per case; cases per team member per week.
**Product (process):** Intervention rate on agent decisions (Poppy's bar: ≤10% changed), split by set-aside / put-back / include / dismiss. Harm metric: nurse rework on items the records team accepted. Calibration: accept rate should rise with stated confidence.

## Core flow
1. User opens the queue → sees cases sorted by actionability (Ready to review, New documents, Waiting on facility, Sent), with checklist progress, days since initial request, and oldest open facility request. Can re-sort by column header or filter by status. Clicks a case.
2. User sees the nurse's checklist with each item's status (Found / Partial / Missing) computed by the agent from the faxes that have landed in SharePoint, and a case-bar counter "Agent decisions changed: N of M".
3. User clicks a **Found** item → sees the pages the agent extracted from the fax, with matching lines highlighted, next to the agent's confidence and plain-English rationale → accepts, or sets a page aside ("Not this page", reversible with "Put back"). Pages the agent excluded are shown below with "Include" / "Confirm exclude". A "Generate follow-up" button drafts a request for anything beyond what arrived.
4. User clicks a **Partial** or **Missing** item → sees what was found (if anything), the gap the agent identified (e.g. the MRI report cites a 2024 comparison X-ray that isn't in the fax), and a pre-drafted follow-up to the right facility → sends it.
5. User clicks the **"Also relevant?"** suggestion → sees pages not on the checklist that the agent thinks the nurse will want (a 2023 arthroscopy report on the same knee) → adds to package or dismisses. Opt-in only.
6. User clicks **Send to nurse review** → sees the package summary (items complete, gaps with follow-up sent, extras added, pages sent vs received, decisions changed) → confirms. Case flips to "Sent to nurse review" in the queue.

**The aha moment:** step 3/4 — opening a checklist item and seeing the right pages already pulled out of a messy fax, with the reason why, and the gap already drafted as a follow-up.

## What "Send to nurse review" does (described, not built)
- **SharePoint:** writes one organized PDF per checklist item (accepted pages only) into the case folder alongside the untouched raw faxes.
- **Cora:** updates the case record per checklist item (status, link to the PDF, agent rationale), logs open follow-ups as outstanding requests, attaches "also relevant" extras flagged as added-by-records-team, and moves the case to the nurse queue. Cora has no API, so v0 is either a small Cora change or a pasted summary block; SharePoint via Graph API is straightforward.
- **Eval log:** every accept / set-aside / put-back / include / confirm / dismiss / add is recorded against the agent's original decision. This is the page-level label set the brief says doesn't exist.
- The nurse's workflow is unchanged: they still write the clinical summary and mark "Ready for Consult" in Cora.

## Intelligence layer
Fixed pipeline with a human at the end, not an autonomous agent. See `agent-design.png`. Read & split (OCR, doc boundaries) → label pages (small model; hard rule: patient mismatch never files) → match to checklist (deterministic filter, then model ranks and writes rationale) → draft follow-ups → "also relevant?" suggestions (large model, precision-tuned) → records team review. Tuning asymmetry: a confident wrong "Found" is the costly error for matching; a false alarm is the costly error for suggestions.

## Pressure test (vs. alternatives considered)
- **A. Page splitter/classifier only** (labs / imaging / notes buckets). Solves the 2h problem but not the relevance gap that drives rework. Less differentiated; it's what a generic document AI does.
- **B. Nurse-facing completeness reviewer.** Puts the AI in front of the more expensive user and doesn't remove the Acrobat work. Good v1, wrong v0.
- **C. Facility follow-up automation.** Real value, but mostly workflow plumbing; hard to demo the intelligence layer.
- **Chosen: checklist fulfilment for the records team.** It uses the nurse's checklist as the clinical spec so the non-clinical user isn't asked to make clinical calls, it removes the manual extraction, and the intervention rate is directly measurable per checklist item (Poppy's metric). Riskiest assumption tested: can the agent match scanned pages to checklist items well enough that the records team trusts and accepts it.

## Screens
1. **Case queue:** 5 cases; columns for patient, procedure, days since initial request, checklist progress, documents received, oldest open request, status. Sortable by Requested / Oldest open request / Status headers; status filter dropdown. Only the Alvarez case opens.
2. **Case workspace:** left = nurse checklist with statuses plus an "Agent suggestions" section; center = pages for the selected item, with "Set aside by you" and "Excluded by agent" strips; right = agent confidence, rationale, actions (accept / undo, generate or send follow-up, add / dismiss), and the follow-up draft. "Send to nurse review" is a modal on this screen, not a third screen.

## Data (hardcoded)
- 1 fully built case: right total knee replacement second opinion, patient "R. Alvarez", 3 facilities, 5 checklist items. Two faxes received (Mt Sinai Knee Center 42 pp., Riverside Orthopedics 118 pp.), one facility (Lakeside PT) still pending. 13 representative pages rendered as stylized page cards, not real scans, including one handwritten low-OCR page and one agent-excluded page.
- Statuses: 3 Found (MRI report, ortho consult notes, injection notes), 1 Partial (prior X-rays: found one, MRI cites a 2024 comparison study not received), 1 Missing (PT records: nothing received). 1 "also relevant" suggestion (2023 arthroscopy report).
- 4 other queue rows with name, procedure, progress, days, status. Not clickable.
- All numbers in the UI (confidence, pages, days) are illustrative; none are claimed as real.

## Out of scope (not now)
- Nurse summary generation — different user, different risk profile; v1.
- Physician package view — end consumer, not the bottleneck.
- Sending records requests to facilities automatically — plumbing; we draft, the user sends.
- Editing follow-up drafts — button is a placeholder.
- OCR / document viewer fidelity, search within documents — infra.
- Cora/SharePoint integration UI — described above, not built.
- Login, settings, notifications, history, saved queue views, free-text search.
- Ops dashboard for Derek — the "Agent decisions changed" counter and the pages-sent line in the send modal are the only gestures.

## Assumptions
- Nurses' checklist items in Cora are specific enough (facility + doc type + date range) to anchor matching. The brief gives one pinned example; we assume a spectrum from pinned items to category-plus-range items, and model both.
- Records team members will accept agent-picked pages if shown the rationale and the source page; they won't re-read the whole fax anyway.
- Faxes land in SharePoint reliably enough to trigger processing per case.
- Case-level historical data (request + raw docs + final package) is enough to bootstrap and evaluate matching, even without page-level labels.
- A ~10% intervention rate at the checklist-item level is the right unit for Poppy's bar (not per page).
- Facilities respond to a targeted follow-up faster than to a generic re-request. Unvalidated.

## Decisions / judgement calls
- Target the records team, not nurses: they're the volume bottleneck and the cheapest place to start incrementally.
- Anchor everything on the nurse's checklist rather than free-form categorization: it's the existing clinical spec and it makes the intervention rate measurable per item.
- Include the "also relevant" suggestion even though it's the riskiest piece: it's the only part that addresses the clinical-relevance gap the brief names as the primary rework driver. Opt-in, visually separate, and the first thing to cut if Poppy pushes back.
- Every action on an agent decision is reversible and the agent's original picks are never destroyed. Sending a follow-up is the one exception.
- Show excluded pages rather than hide them, so the user can overrule the agent and so exclusions are measured too.
- Stand-alone web app alongside Cora rather than a Cora change: Cora changes take weeks; this reads from SharePoint and hands off to Cora at the end.
- Full list, with rationale, in `LOG.md`.

## Build constraints
- Front end only, hardcoded data, no backend or login.
- Single HTML file (vanilla JS + CSS), opens from disk; no build step.
- Build ONLY the core flow above; don't add unrequested features.
- Page "scans" are styled placeholders, not images.
