# Running log — records assembly prototype

## 1. Not in v0 / out of scope
- Nurse-facing summary generation and the nurse's review screen. Different user, higher clinical risk; v1.
- Physician package view. End consumer, not the bottleneck.
- Automatically sending records requests / follow-ups to facilities. We draft the follow-up; sending and tracking still happen through Cora.
- Editing the drafted follow-up text (button is a placeholder).
- OCR, real document viewer, search within documents. Page "scans" are stylized placeholders.
- Cora and SharePoint integration UI. Assumed: checklist read from Cora, documents from SharePoint Graph API, package written back to Cora at "Send to nurse review".
- Login, settings, notifications, case history, multi-user assignment.
- Ops dashboard for Derek. The only gesture is the "agent decisions changed: N of M" counter and the pages-reviewed line in the send modal.
- Other queue rows are not clickable; only the Alvarez case is built.

## 1b. Other judgement calls
- Primary user is the records team member, not the nurse. They are the volume bottleneck and the cheapest place to start incrementally.
- Anchor the agent on the nurse's existing checklist in Cora rather than free-form page categorization. It is the existing clinical spec, it keeps the non-clinical user out of clinical calls, and it makes Poppy's intervention rate measurable per checklist item.
- Stand-alone web app alongside Cora rather than a Cora change, because Cora changes take weeks.
- Three checklist statuses only (Found / Partial / Missing) plus Accepted. No confidence thresholds exposed as settings.
- "Not this page" and "Dismiss suggestion" count as interventions; "Accept" and "Add" do not. That is the measurement Poppy cares about, surfaced in the case bar.
- Found items also get a "Generate follow-up" button that reveals a drafted request for anything beyond what arrived (addenda, later visits, a typed copy of a handwritten note). Nothing is drafted until asked, so the default view for a complete item stays quiet. Generated drafts can be discarded until sent.
- Pages the agent excluded are shown, not hidden, with "Include" / "Confirm exclude". Including counts as an intervention (the agent was overruled); confirming does not, but it is still logged as a label. Exclusions count toward the agent-decision denominator.
- Every action on an agent decision is reversible: set-aside pages move to a "Set aside by you" strip with "Put back", Accept has Undo, Add/Dismiss on a suggestion has Undo. The agent's original picks are never destroyed, and the intervention count reflects current state, so putting a page back un-counts it. Sending a follow-up is the one non-reversible action, because a fax cannot be un-sent.
- Show the agent's rationale as short plain-English bullets next to the pages, not a score alone. The bet is that rationale plus source page is what earns trust.
- Show one excluded page (bronchitis) on the ortho consult item so the demo makes the "relevance filtering" point visible.
- The "also relevant" suggestion is a separate section, visually distinct (dashed purple), and strictly opt-in. Nothing is added to the package unless the user adds it.
- Handwritten / low-OCR page is flagged on the page itself and in the rationale, and the item's confidence is lower, rather than the agent silently dropping it.
- Stack: single HTML file, vanilla JS, no build step, so iterations in the remaining time are fast.
- Queue sorting lives on the column headers (Requested, Oldest open request, Status), click again to flip direction. Default sort is Status by actionability (Ready to review > New documents > Waiting on facility > Sent), ties broken by oldest request. Status filter is a dropdown. No free-text search, no saved views. Header copy trimmed to the minimum.
- "Requested" column shows days since the initial records request, distinct from "oldest open request", because the initial date is what the two-week SLA clock runs against.

## 2. Assumptions
- Nurses' checklist items in Cora are specific enough (facility + document type + date range) to anchor matching. If they are often vague, the agent needs a "clarify checklist" step first.
- Records team members will accept agent-picked pages if shown the rationale and the source page, and will not re-read the whole fax.
- Faxes land in SharePoint reliably per case, so processing can be triggered on arrival.
- Case-level historical data (request + raw docs + final package) is enough to bootstrap and evaluate matching even without page-level labels.
- A ~10% intervention rate at the checklist-item level is the right unit for Poppy's bar (not per page).
- Facilities respond to a targeted follow-up ("the 03/2024 radiograph your MRI report references") faster than to a generic re-request. Unvalidated.

## 3. What got cut
- (nothing cut yet beyond the out-of-scope list above; update as we iterate)

## 4. What we'd build with more time
- Nurse review screen that shows the same checklist with the records team's decisions and the agent's rationale, so the nurse's approval is a second measured signal.
- Editable follow-up drafts, and sending them via the existing fax line with the request logged in Cora automatically.
- Page-level feedback loop: every "Not this page" and "Add" becomes a labelled example, which is the training and eval data the brief says does not exist today.
- Queue prioritisation by "ready to send" vs "waiting on facility" so the team works cases that can actually move.
- Cross-facility gap detection at intake time (the agent reads the nurse's checklist and the intake packet and predicts which requests are likely to come back incomplete).
- Confidence calibration and an eval dashboard for Poppy: intervention rate per item type, per facility, per document quality.

## 5. Who it's not for
- Nurses: they receive the output but do not work in this tool in v0.
- Physicians: they get the package from Cora as today.
- Derek / ops leadership: no reporting surface; they get numbers from Cora exports as today.
- Facilities: nothing changes on their side.
- Patients / insurers: no exposure.
