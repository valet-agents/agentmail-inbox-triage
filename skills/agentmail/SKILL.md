The connector name `agentmail` IS the CLI command on PATH. Never invoke `npx agentmail-cli`, `agentmail-cli`, or any npm package name — always just `agentmail`. The CLI is preconfigured with `AGENTMAIL_API_KEY` from the connector slot, so you do not pass `--api-key` at the command line.

─── Running commands non-interactively ──────────────────────────────────────────────────────────────

Always invoke the CLI with this shape:

  PAGER=cat agentmail <root-flags> <resource> <subcommand> <subcommand-flags>

Three rules that bite if you get them wrong:

1. Disable the pager. When stdout is a TTY, the CLI may pipe output through
   `$PAGER` (default `less`) and hang waiting for keypresses. Prefix every
   invocation with `PAGER=cat` (or pipe to `| cat`).

2. Root flags go BEFORE the resource, subcommand flags go AFTER.
   `--format`, `--debug`, `--yes` / `-y` are root flags on `agentmail`
   itself — placing them after the subcommand makes them unknown flags.

3. Always request structured output. Pass `--format json` (single envelope)
   or `--format jsonl` (one record per line, easier to pipe). Never parse
   the human-readable default — it is for humans, not agents.

Canonical example — list the 10 most recent threads in an inbox, newest first:

  PAGER=cat agentmail --format json threads list --inbox $INBOX_ID --limit 10 --order desc

─── Resources you actually use ──────────────────────────────────────────────────────────────────────

The agent uses a small subset. Confirm exact subcommand names with
`PAGER=cat agentmail --help` and `PAGER=cat agentmail <resource> --help`
on first use, then cache the resolved shape.

  inboxes        List the inboxes attached to your AgentMail account.
  threads        List threads in an inbox; read a single thread.
  messages       List messages in a thread; read a single message; reply.
  drafts         Create a draft reply on a thread (do not send).

─── Common flows ────────────────────────────────────────────────────────────────────────────────────

1. Discover the inbox the agent should triage.

     PAGER=cat agentmail --format json inboxes list

   Pick the inbox by `INBOX_ID` env var if set, else the first inbox
   returned. Cache the resolved id into MEMORY.md.

2. List recent threads in that inbox, newest first, capped.

     PAGER=cat agentmail --format json threads list \
       --inbox $INBOX_ID --limit 25 --order desc

   Take threads with `received_at >` the MEMORY.md watermark.

3. Read the latest message on a thread (the inbound one to triage).

     PAGER=cat agentmail --format json messages list \
       --thread $THREAD_ID --order desc --limit 1

   Or, if you have the message id directly:

     PAGER=cat agentmail --format json messages get --id $MESSAGE_ID

4. Draft a reply, do NOT send. The draft is what the agent posts to
   Slack for human approval.

     PAGER=cat agentmail --format json drafts create \
       --thread $THREAD_ID \
       --body "$REPLY_BODY"

   // TODO: confirm exact draft subcommand surface — fall back to building
   // the draft text in MEMORY.md and only calling `messages reply` once
   // the user confirms, if `drafts create` is not available.

5. Send the reply ONLY after explicit user confirmation in Slack.

     PAGER=cat agentmail --format json messages reply \
       --thread $THREAD_ID \
       --body "$APPROVED_BODY" \
       --yes

─── Hygiene ─────────────────────────────────────────────────────────────────────────────────────────

- Never call any send / reply / forward subcommand without an explicit
  in-thread 👍 or "send" from the user. Drafting is free; sending is not.
- Always pass `--format json` or `--format jsonl` for parseable output.
- Cache the resolved inbox id, last-seen thread id, and last-seen
  `received_at` in MEMORY.md so subsequent heartbeats skip discovery and
  de-duplicate cleanly.
- Redact obvious PII when re-posting email contents to public Slack
  channels (mask the address as `m***@example.com`).
