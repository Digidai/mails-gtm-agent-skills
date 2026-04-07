# AI SDR Agent — Autonomy Extension (OpenClaw)

This extension adds autonomous pipeline execution to the AI SDR Agent. It requires the core skill (SKILL.md) to be installed.

## State Management

### Campaign-level (`.gtm/`)
- `autonomy.json` — configuration (batch_size, cooldown_days, send_window)
- `pending_review.json` — replies needing manual classification
- `draft_replies.json` — auto-generated reply drafts awaiting human approval
- `digests/` — historical digest files

### User-level (`~/.mails/gtm/` or env-based)
If file system access is available:
- `~/.mails/gtm/daily-cap-{mailbox-hash}.json` — daily send tracking per mailbox
- `~/.mails/gtm/sent-keys-{mailbox-hash}.json` — idempotency keys per mailbox

If file system access is NOT available (e.g., sandboxed OpenClaw session):
- Track daily cap and sent keys in conversation memory. Note: these will not persist across sessions. Warn user on first run.

Use first 8 hex chars of SHA-256 of $MAILS_MAILBOX as `{mailbox-hash}`.

## Auto Pipeline

### Workflow: Run Autonomous Pipeline

1. Load campaign state and contacts from `.gtm/`. Abort if missing.
2. Load `autonomy.json` (create with defaults if missing).
3. Load user-level daily cap (create with defaults if missing).
4. Reset daily cap if date changed.
5. **Circuit breaker:** IF 3+ consecutive failures OR >10% error rate → pause, warn user.
6. **Daily cap:** IF `today_sends >= daily_cap` → skip to reply processing.
7. **Send window:** IF configured and outside window → skip to reply processing.
8. **Generate & Send:**
   - Send `approved` contacts first.
   - For `pending` contacts:
     - Semi-auto (default): show preview, require approval per email.
     - Full-auto: generate and send without approval.
   - Before each send, check idempotency key (`{email}_{date}_{angle}`).
   - Send via:
     ```
     POST $MAILS_API_URL/api/send
     Authorization: Bearer $MAILS_AUTH_TOKEN
     Content-Type: application/json
     {"from": "$MAILS_MAILBOX", "to": ["{email}"], "subject": "{subject}", "text": "{body}"}
     ```
   - On success: update status, increment counters, append idempotency key, log to decisions.json.
   - On failure (non-2xx): wait 5 seconds, retry once. If still fails: mark `send_error`, increment failure counters.
9. **Follow-ups:** Auto-send for contacts due (same rules as core skill). Check idempotency.
10. **Reply processing:**
    - Fetch with pagination:
      ```
      GET $MAILS_API_URL/api/inbox?direction=inbound&to=$MAILS_MAILBOX&limit=50
      ```
      Use `&before={oldest_id}` for pagination.
    - Per-intent automation rules:
      - `unsubscribe`/`do_not_contact`: AUTO-EXECUTE at ANY confidence.
      - `interested` (high): auto-update + draft reply (never auto-send replies).
      - `not_now` (high): auto-update, set follow_up_after.
      - `not_interested`/`wrong_person`: human review always.
      - `unclear`/medium/low confidence: human review.
11. **Digest:** Output summary with angle tracking. Save to `.gtm/digests/{date}.txt`.

### Workflow: Switch Mode

- Set campaign mode to `manual` or `autonomous`.
- First autonomous switch requires confirmation.

### Workflow: Show Digest

Display most recent digest from `.gtm/digests/`.

## Concurrency

One session per campaign. No parallel runs on same `.gtm/` directory.

## Error Handling

- Missing state files: create with defaults.
- Corrupt JSON: warn user, recreate as empty.
- No file system access: fall back to conversation memory with persistence warning.
