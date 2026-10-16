---
name: "gmail-inbox-zero"
description: "Safely clear and organize a Gmail inbox to zero. Audits the inbox, creates Action/Client labels, adds newsletter filters, bulk-archives the backlog without deleting, protects starred, Action and receipt mail, optionally clears Promotions/Social/Forums/Updates, and ends with an Executive Action Digest. Use when the user says \"inbox zero\", \"clear my inbox\", \"clean up Gmail\", \"archive my backlog\", or has a huge Gmail inbox they want cleaned without losing important email."
---
 
# Gmail Inbox Zero
 
Safely reduce any Gmail inbox backlog to zero. Archiving removes the `INBOX` label; messages stay indexed and searchable in All Mail. Requires the Google Workspace (Gmail) connector.
 
## Operational Rules
- **Safety first:** Archive by default. Never delete or trash without explicit permission (the optional step 6 is the only exception, and only when the user asks). Never touch Spam or Drafts. Never send mail.
- **Protected mail:** Never remove INBOX from anything matching `label:Action` or `is:starred`.
- **Receipt protection:** Exclude `(receipt OR invoice OR statement OR order OR ticket)` from category cleanups (step 6).
- **Label before archiving** so important mail stays findable.
- **Reporting cadence:** Present a Pre-Flight Audit before large batch runs, emit milestone metrics during execution, and finish with the Executive Action Digest.
- **Autonomy:** Confirm the plan once (step 1). If the user has already said "go ahead / stop asking", skip the confirmation and run autonomously, coming back only when genuinely unsure.
 
## Workflow
 
### Step 1: Pre-Flight Audit and Strategy Check
Run `search_gmail_messages` with `in:inbox` (page_size 5) to confirm access. Sample recent headers with `get_gmail_messages_content_batch` (format=metadata, batches of 20-25 to avoid 429s). Estimate size by paging with page_size 500. Show:
 
| Metric | Details |
| :--- | :--- |
| Total inbox backlog | ~[count] messages |
| Backlog date span | [earliest] to [latest] |
| Top noise senders | [sender] ([count]), ... |
| Detected clients/entities | [domain or sender], ... |
| Proposed labels | `Action`, `Clients/[Name]`, `Clients/Other` |
 
Ask the user once to approve the labels and plan before changing anything (unless they already said to proceed).
 
### Step 2: Provision Labels
`manage_gmail_label` (action=create); record the returned IDs:
- `Action`: pending user tasks or replies
- `Clients/<Client_Name>` for each primary client, plus `Clients/Other`
 
### Step 3: Triage and Protect Important Threads
1. **Client mail:** search `from:(@domain.com) in:inbox`, then `batch_modify_gmail_message_labels` with add_label_ids=[client label].
2. **Action items:** find recent person-sent mail containing a question or request and add `Action`. When unsure, add `Action` (false positives are cheap). Record Subject, Sender and Date of the top 10 for the final digest, with a one-line context note each.
 
### Step 4: Recurring Filter Suppression
Create one filter with `manage_gmail_filter` (action=create) for recurring newsletters and automated senders:
- criteria `{"from": "(a.com OR b.com OR c.com)"}`
- filter_action `{"removeLabelIds": ["INBOX"]}`
Tell the user which senders are included.
 
### Step 5: Safe Sweep Loop
Repeat until the protected search returns nothing:
- Query: `in:inbox -label:Action -is:starred` with page_size 500 (drop to 250 if responses fail).
- Large results are saved to a host file. Extract IDs with the Grep tool: pattern `Message ID: [0-9a-f]+`, output_mode content, `-o` true, head_limit 0. (bash and python can't read that path.) If the output is small enough to arrive inline, read the IDs directly.
- `batch_modify_gmail_message_labels` with remove_label_ids=["INBOX"], up to 500 IDs per call (use ~100 if errors occur). Skip anything that arrived in the last few minutes.
- **Milestone output every 3 batches:** `[Sweep Progress] Archived ~X messages | Inbox remaining: ~Y`
- Keep a running total for the digest.
 
### Step 6: Optional Category Cleanup (explicit user request only)
For each of `social`, `promotions`, `forums`, `updates`:
- Query (do not add `in:inbox`, since the sweep has usually already archived these): `category:X -is:starred -label:Action -label:Clients -in:trash -in:spam -subject:(receipt OR invoice OR order OR statement OR ticket)`
- Extract IDs as in step 5, then `batch_modify_gmail_message_labels` with add_label_ids=["TRASH"] and remove_label_ids=["INBOX"]. Repeat until the query is empty.
- Tell the user it is recoverable from Trash for 30 days, and warn up front that Forums and Updates can be thousands of messages.
- If adding `TRASH` errors, fall back to archive only (remove INBOX) rather than failing the run.
 
### Step 7: Verification
Run `in:inbox` twice (the first can race the last modify). The remaining inbox count should equal the `is:starred` plus `label:Action` items still in the inbox. Also confirm `label:Action` and `is:starred` still return results.
 
## Error Handling
- **Token Expired / disconnected:** stop immediately and tell the user: "Please re-authenticate your Google Workspace connector in Settings → Connectors, then reply 'continue'." Re-test with a 1-message search, then resume with a fresh `in:inbox` search. Don't reuse stale ID lists unless that batch was never applied.
- **Rate limit (HTTP 429, "Too many concurrent requests"):** reduce metadata batches to 15-20 and label batches to 100, then retry the failed IDs.
- **Parallel calls can race:** never trust a search that ran alongside a modify; always do a final standalone check.
 
## Deliverable: Executive Action Digest
 
### Inbox Zero Operations Report
 
| Operation Metric | Result |
| :--- | :--- |
| Total messages archived | [count] |
| Messages trashed (step 6, if run) | [count] |
| Action items identified | [count] (`label:Action`) |
| Client threads labeled | [count] across [X] client labels |
| Automated filters added | [count] senders routed |
| Current inbox state | **0 un-triaged items** ([X] Action/starred items retained) |
 
#### Immediate Action Required
Top preserved threads labeled `Action`:
1. **[Sender / Organization]**: `[Subject]` ([date]). Context: [one-sentence summary of the request].
2. ...
 
#### Quick Navigation and Undo
- Items needing a reply: `label:Action`
- Client communications: `label:Clients`
- Undo a batch: search `label:[LabelName]` (or the date range), select all, choose **Move to Inbox**. Trashed items: open Trash and restore within 30 days.
 
#### Caveats to state
- Any scheduled triage that only searches the inbox won't see archived client mail unless updated to include `label:Clients` and `label:Action`.
- Offer (don't force) a daily triage task that drafts replies and posts a summary, with no sending.