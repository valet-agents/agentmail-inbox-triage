# Inbox Sweep (Heartbeat)

The heartbeat channel fires every 2 minutes. There is no payload
to parse — your job is to find threads that arrived in the
configured AgentMail inbox since the last sweep, research each
sender, draft a reply, and post a triage card to Slack.

## What it does

1. Resolve the inbox per SOUL **Phase 1**: prefer the `INBOX_ID`
   env var; otherwise auto-discover via `agentmail inboxes list`
   and pick the first inbox returned. Cache the resolved id into
   MEMORY.md.
2. Read MEMORY.md for `last_seen_thread_id` and
   `last_seen_received_at`. List threads via
   `agentmail threads list` sorted by `received_at` descending
   and take any with `received_at` newer than the watermark,
   capped at 5 per fire.
3. De-dup against MEMORY.md `drafts` keyed by message id. Skip
   any message id that already has a draft posted.
4. For each new thread (oldest first), read the latest inbound
   message via `agentmail messages list/get`, then run a
   Parallel deep-research pass per SOUL **Phase 3** — company,
   role, last public mention, likely intent.
5. Draft a reply per SOUL **Phase 4** — agent voice, six
   sentences max, register-matched. If intent is clearly
   low-value, skip the draft and mark the card `can ignore — no
   reply needed`.
6. Compose the triage card per SOUL **Phase 5** — Slack
   `mrkdwn`, under 2,000 chars, sender summary capped at 3
   lines, every claim sourced, sender email redacted in public
   channels.
7. Resolve target channels per the SOUL **Where to post** rules
   and post one card per channel per thread. If a post fails
   for a particular channel, log and continue with the others —
   do not retry.
8. Update MEMORY.md (see state shape below). The next fire
   reads from there.

## MEMORY.md state shape

The agent persists a small block in MEMORY.md to track what's
been processed and to back the approval flow. Shape:

```
## agentmail-inbox-triage

inbox_id: inb_XXXXXXXXXXXXXX
last_seen_thread_id: thr_XXXXXXXXXXXXXX
last_seen_received_at: 2026-05-05T17:42:00.000Z

drafts:
  msg_XXXXXXXXXXXXXX:
    thread_id: thr_XXXXXXXXXXXXXX
    body: |
      Thanks for reaching out — happy to chat next week. I'm
      around Tuesday or Thursday afternoon Pacific…
    posted_to:
      - channel: C0123ABCD
        ts: "1714940000.000100"
      - channel: C0456EFGH
        ts: "1714940000.000200"
    intent: vendor pitch
    confidence: med
```

Update this block in place each fire. Do not append a new block
per fire. Drafts older than 7 days with no approval action can
be garbage-collected on the next sweep.

## Where to post

Per SOUL: every Slack channel the bot is a member of, one card
per thread per channel. If the bot is in zero channels, DM the
workspace install user with the card and a one-line invite hint.

## Skip conditions

Skip posting (and stop silently) if any of these are true:

- The inbox can't be resolved (no `INBOX_ID` set and
  `agentmail inboxes list` returns nothing). Write
  `inbox: not_found` to MEMORY.md and stop. Do not post.
- This is the first run after deploy. Seed the watermark to the
  most-recent existing thread and stop — the backlog is not
  triaged.
- Zero threads have `received_at` newer than the watermark.
  Stay silent — no "no new mail" post.
- The thread's latest message is from us, not inbound (we
  already replied). Skip and advance the watermark past it.
- The thread's latest inbound message id already has a draft in
  MEMORY.md `drafts`. Skip — the previous fire posted the card,
  it just hasn't been actioned yet.
