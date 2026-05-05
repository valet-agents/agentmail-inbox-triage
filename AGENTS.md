This folder contains the source for a Skilled Agent originally built for the Valet runtime. Changes should follow the Skilled Agent open standard.

## Setup

### Connectors

- **agentmail**: The AgentMail CLI, preconfigured with the API key from the connector slot. The connector name `agentmail` IS the CLI command on PATH — invoke it as `agentmail`, never as `npx agentmail-cli` or any npm package name. The agent uses it to list inboxes, list threads in `received_at` order to detect new arrivals, read messages, draft replies, and (only after an explicit Slack confirmation) send replies. See `skills/agentmail/SKILL.md` for invocation patterns.
- **parallel-search-mcp**: Parallel's deep-research search MCP. The agent uses it to research each new sender — company, role, last public mention, likely intent — with source URLs returned alongside every result. Keyless; add it from the catalog at the org level.

### Channels

- **slack** (slack): The agent's per-agent Slack bot. Listens for @mentions and replies in-thread, posts each triage card to whichever channels the bot has been invited to, and routes 👍 / ✏️ / ❌ reactions to the send/edit/skip flow. Slack writes use the auto-injected outbound Slack connector.
- **heartbeat** (heartbeat): Fires every 2 minutes to sweep the configured AgentMail inbox for new threads. Declared inline in `valet.yaml`, so it's created automatically by the dashboard setup flow.

### Secrets

- **AGENTMAIL_API_KEY** — required, sourced from the AgentMail dashboard at agentmail.to. The connector slot collects it during the dashboard setup flow.

The Parallel MCP is keyless and the Slack bot is provisioned via OAuth in the dashboard, so no other secrets are required.

### External Setup

1. Sign up for AgentMail at agentmail.to.
2. Create a dedicated inbox for this agent (e.g. `triage@<your-subdomain>.agentmail.to`). Forward or alias the addresses you want triaged into this inbox — the agent only watches the one it's pointed at.
3. Copy your AgentMail API key from the dashboard and paste it into the `AGENTMAIL_API_KEY` slot during deploy. Optional: set the `INBOX_ID` env var on the agent to pin a specific inbox; otherwise the agent picks the first inbox returned by `agentmail inboxes list`.
4. Install the agent's Slack bot from the dashboard. Invite it to whichever channel(s) you want triage cards in. The agent posts to every channel it's a member of — invite it to one focused channel, or several. If the bot has not been invited anywhere, the card is sent as a DM to the workspace install user with a one-line nudge to invite it somewhere.
5. Invite the bot to any additional channels where teammates should be able to @mention it for ad-hoc inbox questions (e.g. *"who is this person and how should I reply?"*).
6. The first heartbeat fire after deploy seeds the watermark to the most recent existing thread — it does not triage the backlog. Send a fresh email to the inbox to smoke-test the flow within 2 minutes.

## Customizing

- **Change the heartbeat cadence**: edit the `every` value on the `heartbeat` channel in `valet.yaml` (e.g. `1m` for the highest-volume inboxes that want near-real-time, `5m` for low-volume inboxes that don't need sub-2-minute SLAs), then redeploy. The default `2m` matches typical email response expectations.
- **Pin a specific inbox**: set `INBOX_ID` on the agent to skip discovery when your account has multiple inboxes. Useful if you don't want every inbox triaged — only one.
- **Tune what counts as low-value**: edit *Phase 4: Draft the reply* in `SOUL.md` to expand or shrink the "skip" categories (cold vendor pitches, generic recruiter blasts, transactional receipts). Skipped threads still get a card in Slack — they're just marked `can ignore — no reply needed` instead of carrying a draft.
- **Adjust the dossier sections**: edit the *Phase 5* template in `SOUL.md` to add or drop fields (e.g. drop the `Confidence` line if your team prefers a tighter card, or add a `Last contact` line if you want history surfaced).
