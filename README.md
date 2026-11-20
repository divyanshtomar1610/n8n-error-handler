# n8n error handler

One n8n workflow that every other workflow points at as its Error Workflow: it classifies each failure, alerts you, and does not send the same failure twice inside a short window. Two versions: **lite** (Slack only, one credential, five minutes) and **full** (adds a Notion log, email for the worst failures, and an email fallback when Slack is down).

**Why I built this.** A scheduled workflow of mine failed and nothing said so until someone looked. This handler is the answer: one place that watches every workflow, logs first, alerts once.

## What it is for

Unattended n8n workflows (scheduled jobs, webhooks) fail silently unless something is watching. Wiring an alert into each workflow by hand is easy to forget, and an alert channel that repeats the same failure over and over gets muted.

This handler gives each failure one of three tiers and suppresses repeats. The full version also writes a log row before it sends anything, and still alerts when the log or Slack is down.

| Tier (full version) | Meaning | What happens |
|---|---|---|
| P1 | A record was lost or unconfirmed, a critical workflow failed, or a credential is dead | Slack post and email |
| P2 | Any other production failure | Slack post |
| P3 | Not production (test, demo, the fault injector) | Notion row only |

## Run it in one minute

Start with the lite version. You need n8n 2.22.0 or later and a Slack app that can post to your alert channel.

1. In n8n, create a workflow, open the three-dot menu, choose **Import from File**, and select `workflow-lite.json`.
2. On **Slack Alert**, set **Authentication** to match your Slack credential (OAuth2, or Access Token for a bot token), pick the credential, and replace `{SLACK_CHANNEL_ID}` with the channel ID (the channel's About tab, at the bottom). Keep the handler in the same project as the workflows it watches.
3. Save, then **Publish**. n8n runs only the published version of an error workflow; publish again after every edit.
4. Open each unattended workflow, go to **Settings**, and set **Error workflow** to this one.

| Tier | Lite version |
|---|---|
| P1 | A credential or authorisation failure (including any 401 or 403): posted |
| P2 | Any other failure: posted |
| P3 | Workflows whose name starts with TEST, TEMP or DEMO: not posted |

The same failure (same workflow, node, tier and message) is posted once, then not again for 1 hour (P1) or 6 hours (P2). There is no database: the repeat check is kept in the handler's own workflow data, which n8n saves for published runs. Only a post Slack accepted is remembered, so a failed post never silences the next one. To change the TEST/TEMP/DEMO rule or the windows, edit the two lines at the top of the **Classify** node.

## Upgrade to the full version

The full version (`workflow.json`) adds a Notion log of every failure, email for P1, an email fallback when Slack is down, dated mutes for known conditions, and your own P1 rules. The import takes a minute; the setup takes longer, because you create one Notion database and pick three credentials. [SETUP.md](SETUP.md) has every step.

You need n8n 2.22.0 or later (cloud or self-hosted; older versions need one change, see Limits), a Notion integration, a Slack app that can post to your alert channel, and a Gmail account or another email node.

1. In n8n, create a workflow, open the three-dot menu, choose **Import from File**, and select `workflow.json`.
2. Create the Notion error log database described in [SETUP.md, section 1a](SETUP.md#1a-the-notion-error-log-database).
3. Replace every `{PLACEHOLDER}` in the nodes. The table in [SETUP.md, section 2](SETUP.md#2-import-and-fill-the-placeholders) lists each one and where it lives.
4. Pick credentials on the five credential pickers: Notion on three nodes, Slack on one, Gmail on one.
5. Save, then **Publish**: n8n runs only the published version of an error workflow, and that applies to every later edit too. Then open each unattended workflow, go to **Settings**, and set **Error workflow** to this one.

Neither file contains credentials, IDs from a real workspace or addresses. Every value you must supply is a `{PLACEHOLDER}`.

## How to check it worked

An error workflow cannot be tested from a manual run, so you build a second, tiny workflow that fails on purpose: the fault injector. [SETUP.md, section 4](SETUP.md#4-build-the-fault-injector-the-positive-control) has the build steps. In short:

1. Create a workflow named `Fault Injector (positive control)` with three Schedule Trigger nodes on a cron that never fires in practice (`0 4 29 2 *`). Keep the words "Fault Injector" in the name: the test messages only set the severity when the failing workflow carries that name, so an ordinary error cannot lower its own severity by quoting one.
2. Connect each trigger to a Stop and Error node with a fixed message: `FAULT-INJECTOR P1: deliberate failure (positive control)`, then the same for P2 and P3.
3. Set its Error workflow to the handler. Add no credentials and no node that touches a real system.
4. Fire each trigger once. SETUP.md gives two ways to do that.

With the lite version, P1 and P2 each post one Slack message, P3 posts nothing, and P2 fired again within 6 hours posts nothing. With the full version you should see:

| You fire | Slack | Email | Notion row |
|---|---|---|---|
| P1 | One message | One email | Severity P1, Alerted = Both |
| P2 | One message | None | Severity P2, Alerted = Slack |
| P3 | None | None | Severity P3, Alerted = None |
| P2 again within 6 hours | None | None | Severity P2, Alerted = Suppressed |

To re-test the lite version inside the window, change the injector's message by one word (a different message is a different failure). To re-test the full version, archive the earlier injector rows in the Notion log first, or wait out the window (1 hour for P1, 6 hours for P2 and P3): a repeat inside the window is suppressed on purpose, whatever the tier.

If the P2 test arrives by email and not in Slack, the Slack app is not a member of the channel.

Fire the injector again after every change to the handler, so the alert path is proved again and not assumed.

## Do not send twice after a failure

An alert tells you a node failed. If that node sends something to a person outside your team, the failure does not tell you whether the message went out. A timeout or a server error can arrive after the message was delivered, and pressing **Retry** sends it again.

Before you re-run anything that sends: sort the error into "proven not sent" or "may have landed", then read the provider's own sent history. Send again only when you have proof the first attempt did not arrive. [SETUP.md, section 6](SETUP.md#6-when-an-alert-names-a-send-node) has the full rule and the build checks for send nodes.

## Limits

- **Lite:** if Slack is down or the app was removed from the channel, the failure is not posted anywhere (the full version falls back to email). The failure still shows in n8n's own execution list.
- **Lite:** two failures within a second of each other can both be posted, because each reads the repeat store before the other writes it. The worst case is an extra post, never a lost one.
- **Both:** if Slack accepts a post but its reply times out, the retry posts it a second time.
- **Both:** the repeat check ignores timestamps in a message but not other numbers, so a failure whose message carries an item ID or a count is posted each time the number changes.
- **Lite:** proved on 2026-11-20 with the fault injector on n8n cloud: a P2 posted, the same P2 again was not posted, a P1 posted, a P3 was not posted.
- It depends on Notion, Slack and Gmail as shipped. You can swap the email node (SETUP.md says what else to change), but the log and the repeat check are written for the Notion API.
- It runs inside n8n. If your n8n instance is down, nothing fires. Point an outside uptime or heartbeat check at the same Slack channel to cover that case.
- Only a row that was actually delivered (Alerted = Both, Slack or Email) counts as an earlier alert, so a failure that keeps recurring is alerted again once each window, and a row whose sends both failed never silences the next one.
- Failures that arrive within seconds of each other all alert: the repeat check reads the log, and the first row is still being written while the second failure is classified.
- Repeat suppression reads the Notion log. If that query fails, the alert is sent anyway, so you can get a duplicate during a Notion outage.
- Repeats are matched on workflow, node, severity and the first 80 characters of the message. Two different failures that share those four values inside the window count as one.
- If Slack and email both fail, the only records are the Notion row (with Alerted left empty) and the handler's own failed execution in n8n.
- The Notion calls use API version `2022-06-28` by header. Raising it means changing the two Notion URLs as well.
- The Slack node is typeVersion 2.5. On an n8n version older than 2.22.0, change it to 2.3 before importing and run the fault injector again.
- The classifier rules are regular expressions and node names that you write. A production failure that matches none of them lands in P2, so nothing is silently downgraded. Without rules from you, only credential and authorisation errors are raised to P1.
- Error text is masked before it is logged or sent: a user name and password inside a URL, the query string of any URL, Slack and Telegram webhook paths, bearer and basic tokens, `key=value` pairs whose key ends in words like key, token, secret, password or authorization, and any unbroken run of 40 or more letters and digits outside a link or a file path. The masking is pattern-based, so treat the log and the alert channel as places a secret could still reach, and limit who can read them.
- Proved on 2026-11-20 with the fault injector described above, on this exact version: a P1 firing posted to Slack, sent the email and marked the row Both; a P2 fired again inside its window was suppressed; a P3 wrote the log row and sent nothing.

## Licence

MIT. See [LICENSE](LICENSE).
