# AI SDR Agent — Autonomy Extension

This extension adds autonomous pipeline execution to the AI SDR Agent skill. It requires the core skill (`gtm-agent.md`) to be installed.

## User-Level State

User-level state is stored in `~/.mails/gtm/` (shared across all campaigns for the same mailbox). Create this directory automatically on first `/gtm auto` run.

Files:
- `~/.mails/gtm/daily-cap-{mailbox-hash}.json` — daily send tracking per mailbox
- `~/.mails/gtm/sent-keys-{mailbox-hash}.json` — idempotency keys per mailbox

Use the first 8 hex chars of SHA-256 of the mailbox email address as `{mailbox-hash}`.

## Campaign-Level State

Additional files in `.gtm/`:
- `autonomy.json` — autonomy configuration and session state
- `pending_review.json` — replies needing manual classification
- `draft_replies.json` — auto-generated reply drafts awaiting human approval
- `digests/` — historical digest files

### autonomy.json
```json
{
  "batch_size": 20,
  "cooldown_days": 3,
  "send_window": null,
  "last_auto_run": null,
  "total_auto_runs": 0
}
```

### daily-cap-{hash}.json (user-level)
```json
{
  "daily_cap": 20,
  "today_sends": 0,
  "last_reset_date": "2026-04-06",
  "consecutive_failures": 0,
  "session_send_count": 0,
  "session_error_count": 0
}
```

### sent-keys-{hash}.json (user-level)
```json
["alex@co.com_2026-04-06_verification-codes", "bob@startup.com_2026-04-06_mailbox-isolation"]
```

Each key is `{contact_email}_{date}_{angle}`. Check before sending. Append after successful send.

## Slash Commands

### /gtm auto

Run the autonomous outreach pipeline. Semi-automatic by default: first-touch emails require human approval, follow-ups and reply processing run automatically.

**Pipeline:**

1. Load `.gtm/campaign.json` and `.gtm/contacts.json`. If either missing/invalid, abort: "Run `/gtm campaign create` first."
2. Load `.gtm/autonomy.json` (create with defaults if missing).
3. Load user-level daily cap from `~/.mails/gtm/daily-cap-{hash}.json` (create with defaults if missing).
4. **Reset daily cap** if `last_reset_date` differs from today's date (local timezone). Reset `today_sends`, `consecutive_failures`, `session_send_count`, `session_error_count`.
5. **Check circuit breaker:** IF `consecutive_failures >= 3` OR (`session_send_count > 0` AND `session_error_count / session_send_count > 0.1`) THEN pause: "Deliverability issue detected. {consecutive_failures} consecutive failures, {error_rate}% error rate. Sending paused. Check your mails config and retry."
6. **Check daily cap:** IF `today_sends >= daily_cap` THEN skip to step 10 (reply processing only). Report: "Daily cap reached: {today_sends}/{daily_cap} sent today."
7. **Check send window:** IF `autonomy.send_window` is set (e.g., `"09:00-17:00"`) AND current local time is outside the window, THEN skip to step 10. Report: "Outside send window ({send_window}). Reply processing only."
8. **Generate & Send phase.** Number of contacts to process = `min(batch_size, daily_cap - today_sends)`.
   a. First: find contacts with status `approved` (from prior manual `/gtm preview`). Send these.
   b. Then: find contacts with status `pending`:
      - **Semi-auto mode (default when `campaign.mode` is `manual`):** Generate email, show preview to user. User approves (Send/Edit/Skip/Stop) per email. Only approved emails are sent. This is the existing `/gtm preview` + `/gtm send` flow.
      - **Full-auto mode (when `campaign.mode` is `autonomous`):** Generate and send without per-email approval.
   c. For each email to send:
      - Generate idempotency key: `{email}_{today_date}_{angle}`
      - Check `~/.mails/gtm/sent-keys-{hash}.json`. IF key exists, SKIP (already sent).
      - Send via: `mails send --to "{email}" --subject "{subject}" --body "{body}"`
      - **On success:** Update contact status to `sent`, increment `today_sends`, reset `consecutive_failures` to 0, increment `session_send_count`. Write daily-cap file immediately. Append key to sent-keys file. Append to `decisions.json` and `conversations/{email-hash}.json`.
      - **On failure (non-zero exit):** Wait 5 seconds, retry once. If still fails: mark `send_error`, increment `consecutive_failures` and `session_error_count`, write daily-cap file. Continue to next contact.
9. **Follow-up phase:**
   a. Find contacts: status `sent`, `last_sent_at` > `cooldown_days` days ago, `emails_sent` < 3.
   b. Generate idempotency key and check before sending.
   c. Generate follow-up with different angle from `knowledge_base.features` (not in `angles_used`).
   d. Send without per-email approval (follow-ups are always automatic in both modes).
   e. Contacts with `emails_sent` >= 3 and no reply: mark `stopped`.
10. **Reply processing phase:**
    a. Fetch inbox with pagination:
       ```bash
       mails inbox --direction inbound --limit 50 --plain
       ```
       If 50 results returned, keep fetching with `--before {oldest_id}` until fewer than 50 returned.
    b. Match each reply's sender to contacts.json using case-insensitive email comparison.
    c. Skip already-processed email IDs (check contact's `processed_email_ids`).
    d. Fetch full body: `mails inbox {email-id}`
    e. Classify intent with confidence (high/medium/low).
    f. **Per-intent automation rules:**
       - `unsubscribe` / `do_not_contact`: AUTO-EXECUTE at ANY confidence. Update status immediately. No further emails ever. Safety: false positive (stopping a contact who didn't ask) is far less costly than false negative (emailing someone who asked to stop).
       - `interested` (high confidence): Auto-update status. Generate draft reply, save to `.gtm/draft_replies.json` for human review. Never auto-send replies.
       - `not_now` (high confidence): Auto-update status. Set `follow_up_after` (default: 30 days).
       - `not_interested` / `wrong_person`: ALL go to `.gtm/pending_review.json` for human review. Closing a contact is irreversible.
       - `out_of_office` / `auto_reply`: No status change. Note return date if OOO.
       - `unclear` / any medium or low confidence: Save to `.gtm/pending_review.json`.
    g. Update `conversations/{email-hash}.json` with inbound message.
    h. Append classification record to `decisions.json`.
11. **Digest output:** Print to terminal AND save to `.gtm/digests/{date}.txt`.

```
═══ GTM Auto Digest ═══
Campaign: {product_name}
Run: {timestamp} | Mode: {manual/autonomous}

Sent:     {N} emails ({N} new, {N} follow-ups)
Cap:      {today_sends}/{daily_cap}
Replies:  {N} processed ({N} interested, {N} not_now, {N} unsubscribe)
Pending:  {N} replies need manual review (/gtm replies)
Drafts:   {N} reply drafts ready for review
Remaining: {N} contacts to process (run /gtm auto again)
Top angle: {angle} ({N}% reply rate)

Pipeline: {N} pending | {N} sent | {N} interested | {N} stopped
═══════════════════════
```

**Angle tracking:** Calculate reply rate per angle by counting contacts where `reply_intent` is not null, grouped by `angles_used[0]` (first angle = initial outreach angle). Only include angles with >= 3 sends for statistical relevance.

12. **Persist state:** Write updated `contacts.json`. User-level files (daily-cap, sent-keys) are already written incrementally.
13. Update `autonomy.last_auto_run` and increment `autonomy.total_auto_runs`.

**Batching:** Processes up to `batch_size` contacts per invocation. If more contacts remain: "X contacts remaining, run `/gtm auto` again."

**Cadence reminder:** If `last_auto_run` is > 24h old and user runs any `/gtm` command, remind: "It's been {N} days since your last autonomous run. Run `/gtm auto` to process pending outreach."

### /gtm mode {manual|autonomous}

Switch campaign mode.

- `/gtm mode manual`: Set `campaign.mode` to `"manual"`. First-touch emails require approval.
- `/gtm mode autonomous`: Set `campaign.mode` to `"autonomous"`. All emails sent without individual approval (subject to safety rules).
- `/gtm mode`: Show current mode.

**First-time autonomous confirmation:** When switching to autonomous for the first time, prompt: "This enables full autonomous mode. ALL emails (including first touch) will be sent without individual approval, subject to daily cap of {daily_cap}. Type 'confirm' to proceed." After confirmation, set mode. Subsequent switches skip confirmation.

### /gtm digest

Show the most recent digest from `.gtm/digests/`. If no digests exist, run a lightweight version that just counts pipeline status.

## Concurrency

Same rule as core: do NOT run multiple `/gtm` commands in parallel. One session per campaign at a time.

## Error Handling

- **autonomy.json missing:** Create with defaults automatically.
- **User-level directory missing (~/.mails/gtm/):** Create automatically with mode 700.
- **Daily cap file missing:** Create with defaults.
- **Sent-keys file missing:** Create as empty array.
- **pending_review.json / draft_replies.json corrupt:** Warn user, recreate as empty array.
