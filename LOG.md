# Running log — records assembly prototype

## 1. Not in v0 / out of scope
- Nurse-facing summary generation and the nurse's review screen. Different user, higher clinical risk; v1.
- Physician package view. End consumer, not the bottleneck.
- Automatically sending records requests / follow-ups to facilities. We draft; the user sends; tracking still happens through Cora.
- Editing the drafted follow-up text (button is a placeholder).
- OCR, real document viewer, search within documents. Page "scans" are stylized placeholders.
- Cora and SharePoint write-back. Described in SPEC.md ("What Send to nurse review does"), not built. The send modal returns to the queue with the status flipped and does not show what is written where.
- Login, settings, notifications, case history, multi-user assignment, saved queue views, free-text search.
- Ops dashboard for Derek. The only gestures are the "Agent decisions changed: N of M" counter and the pages-sent line in the send modal.
- Other queue rows are not clickable; only the Alvarez case is built.
- Labelling checklist items as "pinned" vs "any matching in range". The prototype models both kinds but does not call them out.

## 1b. Other judgement calls
- Primary user is the records team member, not the nurse. They are the volume bottleneck and the cheapest place to start incrementally.
- Anchor the agent on the nurse's existing checklist in Cora rather than free-form page categorization. It is the existing clinical spec, it keeps the non-clinical user out of clinical calls, and it makes Poppy's intervention rate measurable per checklist item.
- The checklist is read as a spectrum: pinned items ("Mt Sinai MRI 12/10/25"), category-plus-range items ("PT records, last 12 months"), and relevant records not on the checklist at all. The brief only gives one pinned example; the rework story (a two-year-old radiology report nobody asked for) implies the other two exist. Items 1–2 are pinned, 3–5 are category-plus-range, the suggestion is the third kind.
- Stand-alone web app alongside Cora rather than a Cora change, because Cora changes take weeks.
- Three checklist statuses only (Found / Partial / Missing) plus Accepted. No confidence thresholds exposed as settings.
- What counts as an intervention: set-aside, dismiss a suggestion, include an agent-excluded page. Accept, add, confirm-exclude, and put-back do not. The counter reflects current state, so an undo un-counts. Denominator is every agent decision on the case: page picks, exclusions, suggestions.
- Every action on an agent decision is reversible: set-aside pages move to a "Set aside by you" strip with "Put back", Accept has Undo, Add/Dismiss on a suggestion has Undo, Include/Confirm on an excluded page has Undo. The agent's original picks are never destroyed. Sending a follow-up is the one non-reversible action, because a fax cannot be un-sent.
- Pages the agent excluded are shown, not hidden, with "Include" / "Confirm exclude". Including overrules the agent and counts; confirming does not count but is still logged as a label.
- Found items also get a "Generate follow-up" button that reveals a drafted request for anything beyond what arrived (addenda, later visits, a typed copy of a handwritten note). Nothing is drafted until asked, so the default view for a complete item stays quiet. Generated drafts can be discarded until sent.
- Show the agent's rationale as short plain-English bullets next to the pages, not a score alone. The bet is that rationale plus source page is what earns trust.
- Handwritten / low-OCR page is flagged on the page itself and in the rationale, and the item's confidence is lower, rather than the agent silently dropping it.
- The "also relevant" suggestion is a separate section, visually distinct (dashed purple), and strictly opt-in. Nothing is added to the package unless the user adds it. First thing to cut if Poppy pushes back.
- Queue sorting lives on the column headers (Requested, Oldest open request, Status), click again to flip direction. Default sort is Status by actionability (Ready to review > New documents > Waiting on facility > Sent), ties broken by oldest request. Status filter is a dropdown. Header copy trimmed to the minimum.
- "Ready to review" ranks above "New documents" on the logic that a case where everything has arrived can be finished and shipped. The reverse ordering (touch fresh faxes first so gaps get flagged sooner) is defensible and a one-line change.
- "Requested" column shows days since the initial records request, distinct from "oldest open request", because the initial date is what the two-week SLA clock runs against.
- Intelligence layer is a fixed pipeline with a human at the end, not an autonomous agent; the steps are predictable. Matching is tuned so a confident wrong "Found" is the error to avoid (user accepts it, nurse finds it later); suggestions are tuned for precision (false alarms train people to ignore the card). See `agent-design.png`.
- Handoff on "Send to nurse review" writes into the systems that already exist (one PDF per item into SharePoint, per-item status and open requests into Cora, actions into an eval log) rather than creating a new place for the package to live. The nurse's Cora workflow is unchanged.
- Stack: single HTML file, vanilla JS, no build step, so iterations in the remaining time are fast.

## 2. Assumptions
- Nurses' checklist items in Cora are specific enough (facility + document type + date range) to anchor matching. If they are often vague, the agent needs a "clarify checklist" step first.
- Records team members will accept agent-picked pages if shown the rationale and the source page, and will not re-read the whole fax.
- Faxes land in SharePoint reliably per case, so processing can be triggered on arrival.
- Case-level historical data (request + raw docs + final package) is enough to bootstrap and evaluate matching even without page-level labels.
- A ~10% intervention rate at the checklist-item level is the right unit for Poppy's bar (not per page).
- Facilities respond to a targeted follow-up ("the 03/2024 radiograph your MRI report references") faster than to a generic re-request. Unvalidated.
- The nurse checklist is written before documents arrive and is not routinely edited afterwards. If nurses do edit it mid-case, the agent needs to re-run matching on checklist change, which the pipeline supports but the prototype does not show.
- Cora can accept a per-item status write-back, either via a small Cora change or a pasted summary block. If neither is acceptable, the tool becomes SharePoint-only and the records team keys status into Cora by hand, which erodes the time saving.

## 3. What got cut
- The `/brief` adversarial spec review. Skipped to keep moving.
- A sort dropdown on the queue, replaced by clickable column headers.
- A "writes N PDFs to SharePoint, updates M items in Cora" line in the send modal. Considered and declined; the handoff is described in SPEC.md instead.
- A nurse-facing screen, a second case, and any real document rendering were never started.
- "Also relevant" is kept but flagged as the next cut if it has to go.

## 4. What we'd build with more time
- Nurse review screen that shows the same checklist with the records team's decisions and the agent's rationale, so the nurse's approval is a second measured signal.
- Editable follow-up drafts, and sending them via the existing fax line with the request logged in Cora automatically.
- Real Cora and SharePoint write-back, and a read of the checklist from Cora so nurses do not double-enter.
- Page-level feedback loop wired end to end: every set-aside, put-back, include, confirm, dismiss and add becomes a labelled example, which is the training and eval data the brief says does not exist today.
- Re-run matching when a fax lands or the checklist changes, with a "new since you last looked" marker on the queue and on items.
- Cross-facility gap detection at intake time: the agent reads the nurse's checklist and the intake packet and predicts which requests are likely to come back incomplete, so the first request is more specific.
- Confidence calibration and an eval dashboard for Poppy: intervention rate per item type, per facility, per document quality; nurse rework on accepted items as the harm metric.
- Queue: saved views, "new documents since last visit", and assignment across the team.

## 5. Who it's not for
- Nurses: they receive the output in Cora but do not work in this tool in v0.
- Physicians: they get the package from Cora as today.
- Derek / ops leadership: no reporting surface; they get numbers from Cora exports as today.
- Poppy: no eval dashboard yet; the intervention counter is the only visible measurement.
- Facilities: nothing changes on their side.
- Patients / insurers: no exposure.
