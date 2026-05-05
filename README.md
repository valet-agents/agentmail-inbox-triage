# AgentMail Inbox Triage

Email lands in your dedicated agent inbox — it pulls the company, the role, and the likely intent, then drafts a reply for your sign-off in Slack.

## Prerequisites
- An [AgentMail](https://agentmail.to) account with a dedicated inbox for this agent and an API key
- A Slack workspace where you can install the agent's bot and invite it to one or more channels

<table>
  <tr>
    <td><strong>CHANNELS</strong></td>
    <td><code>slack</code> · <code>heartbeat</code> — every 2m</td>
  </tr>
  <tr>
    <td><strong>CONNECTORS</strong></td>
    <td><code>agentmail</code> <code>parallel-search-mcp</code></td>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <br />
      <a href="https://valet.dev/deploy?from=github.com/valet-agents/agentmail-inbox-triage">
        <img src="https://raw.githubusercontent.com/valet-agents/agentmail-inbox-triage/main/.github/deploy-button.svg" alt="Deploy Agent →" height="40" />
      </a>
      <br /><br />
    </td>
  </tr>
</table>
