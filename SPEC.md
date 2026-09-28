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
**User (leading):** Time from "documents received" to "sent to nurse review" per case; % of checklist items the agent resolved without the user changing pages.
**Business (lagging):** Rework rate (cases bounced from nurse back to records); records-team hours per case; cases per team member per week.
**Product (process):** Intervention rate on agent outputs (Poppy's bar: ≤10% of outputs modified); % of cases where a gap was flagged before nurse review.

## Core flow
1. User opens a case from their queue → sees the nurse's checklist with each item's status (Found / Partial / Missing) computed by the agent from the faxes that have landed in SharePoint, plus a small "pages reviewed by agent vs. by you" counter.
2. User clicks a **Found** item → sees the specific pages the agent extracted from the 200-page fax, side-by-side with the agent's one-line rationale (facility, date, document type, confidence) → accepts (or removes a page / picks a different page).
3. User clicks a **Missing** item → sees what the agent found that is related but insufficient (e.g. the MRI order, not the report), and a pre-drafted follow-up request to the facility → sends it. Also sees an **"Also relevant"** card: pages not on the checklist the agent thinks the nurse will want (e.g. a 2-year-old X-ray report noting cartilage loss) → adds to package.
4. User clicks **Send to nurse review** → sees the package summary (items resolved, items still pending with follow-up sent, extras added) and a modest "you touched N of M agent decisions" line.

**The aha moment:** step 2/3 — opening a checklist item and seeing the right pages already pulled out of a messy fax, with the reason why, and the missing item already drafted as a follow-up.

## Pressure test (vs. alternatives considered)
- **A. Page splitter/classifier only** (labs / imaging / notes buckets). Solves the 2h problem but not the relevance gap that drives rework. Less differentiated; it's what a generic document AI does.
- **B. Nurse-facing completeness reviewer.** Puts the AI in front of the more expensive user and doesn't remove the Acrobat work. Good v1, wrong v0.
- **C. Facility follow-up automation.** Real value, but mostly workflow plumbing; hard to demo the intelligence layer.
- **Chosen: checklist fulfilment for the records team.** It uses the nurse's checklist as the clinical spec so the non-clinical user isn't asked to make clinical calls, it removes the manual extraction, and the intervention rate is directly measurable per checklist item (Poppy's metric). Riskiest assumption tested: can the agent match scanned pages to checklist items well enough that the records team trusts and accepts it.

## Screens
1. **Case queue** (thin): 4–5 cases, patient, procedure under review, checklist progress (e.g. 3/5), oldest outstanding request. Exists only to give context and let the demo start.
2. **Case workspace** (the product): left = nurse checklist with statuses; center = page viewer for the selected item; right = agent rationale + actions (accept / remove page / send follow-up / add "also relevant"). The "Send to nurse review" summary is a modal on this screen, not a third screen.

## Data (hardcoded)
- 1 fully built case: knee replacement second opinion, patient "R. Alvarez", 3 facilities, 5 checklist items (MRI report, ortho consult notes, PT records, prior X-ray, injection/procedure notes). 3 incoming fax documents with ~10 representative pages each (rendered as stylized page thumbnails with a header line, not real scans). 2 items Found, 1 Partial, 2 Missing; 1 "also relevant" suggestion.
- 3–4 other queue rows with names, procedure, progress, days waiting. Not clickable beyond a placeholder.
- All numbers in the UI (time saved, confidence) are illustrative; none are claimed as real.

## Out of scope (not now)
- Nurse summary generation — different user, different risk profile; v1.
- Physician package view — end consumer, not the bottleneck.
- Sending records requests to facilities automatically — plumbing; we only draft the follow-up.
- OCR / document viewer fidelity, search within documents — infra.
- Cora/SharePoint integration UI — assumed: reads checklist from Cora, documents from SharePoint Graph API, writes package back. Described, not built.
- Login, settings, queue filters, notifications, history.
- Ops dashboard for Derek — one line of "time saved" in the send modal is the only gesture.

## Assumptions
- Nurses' checklist items in Cora are specific enough (facility + doc type + date) to anchor matching. If they're often vague, the agent needs a "clarify checklist" step first.
- Records team members will accept agent-picked pages if shown the rationale and the source page; they won't re-read the whole fax anyway.
- Faxes land in SharePoint reliably enough to trigger processing per case.
- Case-level historical data (request + raw docs + final package) is enough to bootstrap and evaluate matching, even without page-level labels.

## Decisions / judgement calls
- Target the records team, not nurses: they're the volume bottleneck and the cheapest place to start incrementally.
- Anchor everything on the nurse's checklist rather than free-form categorization: it's the existing clinical spec and it makes the intervention rate measurable per item.
- Include the "also relevant" suggestion even though it's the riskiest piece: it's the only part that addresses the clinical-relevance gap the brief names as the primary rework driver, and it's opt-in (user adds it, it's never auto-included).
- Stand-alone web app alongside Cora rather than a Cora change: Cora changes take weeks; this reads from SharePoint and hands off to Cora at the end.

## Build constraints
- Front end only, hardcoded data, no backend or login.
- Single HTML file (vanilla JS + CSS), opens from disk; no build step.
- Build ONLY the core flow above; don't add unrequested features.
- Page "scans" are styled placeholders, not images.
