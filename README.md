# n8n error handler

One n8n workflow that every other workflow points at as its Error Workflow: it classifies each failure, logs it to Notion, alerts by Slack and email, and does not send the same failure twice inside a short window.

## What it is for

Unattended n8n workflows (scheduled jobs, webhooks) fail silently unless something is watching. Wiring an alert into each workflow by hand is easy to forget, and an alert channel that repeats the same failure over and over gets muted.

This handler gives each failure one of three tiers, writes the log row before it sends anything, and suppresses repeats. It still alerts when the log or Slack is down.

| Tier | Meaning | What happens |
|---|---|---|
| P1 | A record was lost or unconfirmed, a critical workflow failed, or a credential is dead | Slack post and email |
| P2 | Any other production failure | Slack post |
| P3 | Not production (test, demo, the fault injector) | Notion row only |

## Run it in one minute

The import takes about a minute. The full setup takes longer, because you create one Notion database and pick three credentials. [SETUP.md](SETUP.md) has every step.

You need n8n 2.22.0 or later (cloud or self-hosted; older versions need one change, see Limits), a Notion integration, a Slack app that can post to your alert channel, and a Gmail account or another email node.

1. In n8n, create a workflow, open the three-dot menu, choose **Import from File**, and select `workflow.json`.
2. Create the Notion error log database described in [SETUP.md, section 1a](SETUP.md#1a-the-notion-error-log-database).
3. Replace every `{PLACEHOLDER}` in the nodes. The table in [SETUP.md, section 2](SETUP.md#2-import-and-fill-the-placeholders) lists each one and where it lives.
4. Pick credentials on the five credential pickers: Notion on three nodes, Slack on one, Gmail on one.
5. Save. Then open each unattended workflow, go to **Settings**, and set **Error workflow** to this one.

`workflow.json` contains no credentials, no IDs from a real workspace and no addresses. Every value you must supply is a `{PLACEHOLDER}`.

## How to check it worked

An error workflow cannot be tested from a manual run, so you build a second, tiny workflow that fails on purpose: the fault injector. [SETUP.md, section 4](SETUP.md#4-build-the-fault-injector-the-positive-control) has the build steps. In short:

1. Create a workflow named `Fault Injector (positive control)` with three Schedule Trigger nodes on a cron that never fires in practice (`0 4 29 2 *`).
2. Connect each trigger to a Stop and Error node with a fixed message: `FAULT-INJECTOR P1: deliberate failure (positive control)`, then the same for P2 and P3.
3. Set its Error workflow to the handler. Add no credentials and no node that touches a real system.
4. Fire each trigger once. SETUP.md gives two ways to do that.

You should see:

| You fire | Slack | Email | Notion row |
|---|---|---|---|
| P1 | One message | One email | Severity P1, Alerted = Both |
| P2 | One message | None | Severity P2, Alerted = Slack |
| P3 | None | None | Severity P3, Alerted = None |
| P2 again within 6 hours | None | None | Severity P2, Alerted = Suppressed |

If the P2 test arrives by email and not in Slack, the Slack app is not a member of the channel.

Fire the injector again after every change to the handler, so the alert path is proved again and not assumed.

## Do not send twice after a failure

An alert tells you a node failed. If that node sends something to a person outside your team, the failure does not tell you whether the message went out. A timeout or a server error can arrive after the message was delivered, and pressing **Retry** sends it again.

Before you re-run anything that sends: sort the error into "proven not sent" or "may have landed", then read the provider's own sent history. Send again only when you have proof the first attempt did not arrive. [SETUP.md, section 6](SETUP.md#6-when-an-alert-names-a-send-node) has the full rule and the build checks for send nodes.

## Limits

- It depends on Notion, Slack and Gmail as shipped. You can swap the email node (SETUP.md says what else to change), but the log and the repeat check are written for the Notion API.
- It runs inside n8n. If your n8n instance is down, nothing fires. Point an outside uptime or heartbeat check at the same Slack channel to cover that case.
- Repeat suppression reads the Notion log. If that query fails, the alert is sent anyway, so you can get a duplicate during a Notion outage.
- Repeats are matched on workflow, node, severity and the first 80 characters of the message. Two different failures that share those four values inside the window count as one.
- If Slack and email both fail, the only records are the Notion row (with Alerted left empty) and the handler's own failed execution in n8n.
- The Notion calls use API version `2022-06-28` by header. Raising it means changing the two Notion URLs as well.
- The Slack node is typeVersion 2.5. On an n8n version older than 2.22.0, change it to 2.3 before importing and run the fault injector again.
- The classifier rules are regular expressions and node names that you write. A production failure that matches none of them lands in P2, so nothing is silently downgraded. Without rules from you, only credential and authorisation errors are raised to P1.

## Licence

MIT. See [LICENSE](LICENSE).
