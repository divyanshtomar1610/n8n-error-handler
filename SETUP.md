# Error handler with alerting: setup guide

**What this gives you.** One n8n workflow that every other workflow points at as its Error Workflow. When any of them fails, this handler decides how serious it is, writes a row to a Notion log, posts to Slack, emails you for the worst cases, and never sends the same failure twice inside a short window. A companion "fault injector" lets you prove the whole chain still works after every change, without waiting for something real to break.

**Who it is for.** Anyone running unattended automations (scheduled jobs, webhooks, queue workers) who wants to know moments after one fails (delivery time depends on Slack and your mail provider), without an on-call rota and without an alert channel that everyone mutes in a week.

**Files.** `workflow.json` (import into n8n) and this guide.

**Works on.** n8n cloud or self-hosted, version 2.22.0 or later (the Slack node in the file is typeVersion 2.5, first shipped in 2.22.0). Older instance: change `"typeVersion": 2.5` on the Slack Alert node to `2.3` before importing, then re-run the fault injector, because the workflow was tested on 2.5. Nothing here depends on the operating system or device you edit from.

---

## 1. Before you import: three things to prepare

### 1a. The Notion error log database

Create a Notion database with exactly these properties (names and types matter, the workflow writes to them by name):

| Property | Type | Options |
|---|---|---|
| Error | Title | |
| Workflow | Text | |
| Node | Text | |
| Message | Text | |
| Execution ID | Text | |
| Status | Select | New, Investigating, Resolved |
| When | Date | |
| Severity | Select | P1, P2, P3 |
| Alerted | Select | Both, Slack, Email, Suppressed, Muted, None |

Share the database with your Notion integration (the one your n8n Notion credential uses). Copy the database ID from its URL: the 32-character string after the workspace name and before the `?`.

The workflow calls Notion API version 2022-06-28 by header. Notion has since split databases into data sources; keep the header as shipped, and do not raise it without changing the two Notion URLs.

### 1b. The Slack channel

Pick or create the channel alerts should land in. If it is a **private** channel, invite the n8n Slack app to it: click the channel name at the top, open the **Agents & apps** tab (older Slack versions: Integrations, Add apps), add the n8n app. Without that invite the Slack node fails with `channel_not_found` and every alert falls through to email. Copy the channel ID: open the channel's About tab and scroll to the bottom, or take it from the channel URL (`C` followed by letters and digits).

Choose a channel that also receives alerts from **outside** n8n if you can (for example an uptime or heartbeat service). Then a dead n8n still leaves a message there.

### 1c. Credentials in n8n

You need three credentials that already exist or that you create in the n8n UI: a Notion integration token, a Slack OAuth2 credential, and a Gmail OAuth2 credential. Create them in n8n's Credentials page, not in the workflow, so the same credential can be reused.

Any email node can replace Gmail Alert (for example n8n's Send Email node over SMTP); if you swap it, also change the `g.id || g.threadId` check in Mark Alerted to whatever your node returns, or Alerted will never read Email.

---

## 2. Import and fill the placeholders

1. In n8n: Workflows, then Create, then the three-dot menu in the upper right, Import from File, choose `workflow.json`.
2. Open each node listed below and replace the `{PLACEHOLDER}` text.

Regular-expression placeholders (`{NON_PRODUCTION_NAME_PATTERN}`, `{P1_ALARM_PATTERN}`, `{P1_WORKFLOW_NAME_PATTERN}`) are pasted between existing `/ /` slashes: write the pattern only, no slashes, and escape any `/` as `\/`. To disable one, use `^$` (never leave it blank: an empty pattern turns the line into a comment and the whole Classify node stops compiling). `{P1_RECOVERY_LINE}` sits inside single quotes: no apostrophes.

| Placeholder | Where | What to put |
|---|---|---|
| `{WORKSPACE_PREFIX}` | workflow name | a short tag that groups your workflows, or delete it |
| `{NOTION_ERROR_LOG_DATABASE_ID}` | Repeat Check (URL), Format Error (code) | the database ID from 1a, in both places |
| `{NON_PRODUCTION_NAME_PATTERN}` | Classify | a regular expression matching workflow names that are NOT production, for example `^(TEMP|TEST|DEMO)`; use `^$` if none |
| `{P1_ALARM_PATTERN}` | Classify | words your own dead-letter alarms use, for example `Persist Failure Alarm|Delivery Failed`; use `^$` if you have none |
| `{P1_NODE_NAMES}` | Classify | single-quoted node names separated by commas, no outer brackets (the code already has them), for example `'Create CRM Contact', 'Save Lead'`; if none, delete the placeholder so the line reads `const p1Nodes = [];` |
| `{P1_WORKFLOW_NAME_PATTERN}` | Classify | a regular expression for workflows whose any failure is critical, for example `^Client Onboarding`; use `^$` to disable |
| `{P1_RECOVERY_LINE}` | Build Alert | the one line you want on every P1, for example `Recovery: open the workflow, read its notes, if a send failed check sent history before sending by hand, never re-execute` (no apostrophes) |
| `{SLACK_CHANNEL_ID}` | Slack Alert | the channel ID from 1b |
| `{ALERT_EMAIL}` | Gmail Alert | the inbox a human reads |
| `{ALERT_SENDER_NAME}` | Gmail Alert | the name the alert email shows as sender |

3. Pick credentials on the five credential pickers: Notion on Repeat Check, Write to Error Log and Patch Alerted; Slack on Slack Alert; Gmail on Gmail Alert.
4. Save. Error workflows do not need to be published; n8n runs them when a linked workflow fails.
5. Wire it in: open every unattended workflow, Options (three dots), Settings, Error workflow, select this one. If you have many workflows, the public REST API accepts `settings.errorWorkflow` (this handler's workflow ID) on the workflow update call.

---

## 3. How the tiers work, and how to change them

The `Classify` node holds one ordered rule table; the first rule that matches wins and unmatched production failures land in the middle tier so nothing is silently downgraded.

| Tier | Meaning | What happens |
|---|---|---|
| P1 | a customer record was lost or unconfirmed, a critical workflow failed, or a credential is dead | Slack post and email |
| P2 | any other production failure | Slack post |
| P3 | not production (test, demo, the fault injector) | Notion row only |

**Repeat suppression.** The same workflow, node, severity and message (first 80 characters, up to the first timestamp) inside 1 hour (P1) or 6 hours (P2) is written to the log with Alerted = Suppressed and not re-sent. The check queries the Notion log; if that query fails, the alert goes out anyway.

**Known-condition mutes.** For a failure you already know about and cannot fix today, add a dated entry to `knownConditions` in Classify. It is logged with Alerted = Muted, sends nothing, and expires on the date you set. Never add one without a date.

**Order of operations.** The log row is written before any message is sent, so a Slack or email outage never loses the record. If Slack fails, the message goes to email whatever the tier. If the Notion write fails, the alert still goes out with "Log row: WRITE FAILED" in the text.

---

## 4. Build the fault injector (the positive control)

Error workflows cannot be tested from a manual run, and you do not want to break a real workflow to test them. Build a second, tiny workflow:

1. Create a workflow named `{WORKSPACE_PREFIX} Fault Injector (positive control)`.
2. Add three Schedule Trigger nodes named `Fire P1`, `Fire P2`, `Fire P3`, each set to Trigger Interval: Custom (Cron), with an expression that never fires in practice, for example `0 4 29 2 *` (minute 0, hour 4, day 29, month 2, any weekday). It fires once on 29 February 2028 at 04:00 in your instance's timezone; expect one deliberate P1, P2 and P3 that morning, or deactivate the injector beforehand.
3. Connect each to a Stop and Error node whose message is exactly `FAULT-INJECTOR P1: deliberate failure (positive control)` (and P2, P3 for the others). Keep the message fixed, with no timestamp, so a second firing counts as a repeat.
4. Optional backoff proof: add two more triggers feeding two HTTP Request nodes to a public URL that answers 429, such as `https://httpbin.org/status/429` (a free public test service; confirm it still answers 429 before relying on it), one with Retry On Fail (3 tries, 5000 ms) and one without.
5. Settings: Error Workflow = the handler; save; publish (the cron cannot fire in a non-leap year, so publishing is inert until 2028).
6. Never add a credential or any node that touches a real system to this workflow.

**To fire it:** you need something that can start a production execution from a chosen trigger. n8n's own MCP server tools can do this (the workflow must be marked available to MCP in its settings); the public REST API cannot, as of the current specification. Without such a tool, temporarily change one trigger's cron to two minutes ahead, wait for it, then restore the never-fires expression. Then check: P1 produced a Slack message and an email; P2 a Slack message; P3 only a log row; a second P2 inside 6 hours a row marked Suppressed and no message.

---

## 5. Checks before you rely on it

- [ ] Every unattended workflow shows this handler under Options, Settings, Error workflow.
- [ ] The Slack app is a member of the channel (a P2 test landed in Slack, not in email).
- [ ] A P1 test email arrived in the inbox a human reads.
- [ ] A repeated P2 test was suppressed.
- [ ] The Notion log shows Severity and Alerted filled on each test row.
- [ ] Nobody but you can trigger the fault injector (no webhook, no credential).

---

## 6. When an alert names a send node

An alert tells you a node failed. If that node sends something to a customer or anyone outside your team (an email, a text, a chat message, a message from your customer relationship management (CRM) system), the failure does not tell you whether the message went out. Sending again can send it twice, and a sent message cannot be taken back.

**Sort the error first.**

- **Proven not sent.** The request never left (the address could not be looked up, the connection was refused, a credential was missing), or the provider refused it outright: "invalid request" (400), "unauthorized" (401) or "forbidden" (403). Fix the cause, then send it once by hand and record it. A "too many requests" (429, rate limit) error usually means nothing was sent; check your provider's documentation before relying on that.
- **May have landed.** A timeout, a server error from the provider (status code 500 to 599), a connection dropped after the request went out, an execution that crashed or was stopped mid-run, or any error you cannot place. Treat these as sent until you prove otherwise.

**Read before you act again.** For "may have landed", look in two places: the provider's own sent history (Sent folder, sent-email log, conversation history, channel history) and your own record of the send, if you keep one. First confirm the search can find a message you know was sent earlier; an empty result from a search that would find nothing anyway proves nothing. Found (that message, to that recipient, sent after the failed execution started): do not send again, and record it as sent. Not found anywhere: send once, by hand, and record it. Cannot tell: ask whoever owns the customer relationship. Do not guess.

**Retrying is retrying.** Pressing Retry on a failed execution, or running the workflow again, re-attempts the send. A full re-run also repeats every other step that already succeeded, such as tasks created and chat messages posted.

**Build checks for every node that sends to a customer or anyone outside your team:**

- [ ] Retry On Fail is off. It is fine on reads and on writes that update one record by a fixed identifier. Alerts to your own team (like this handler's Slack and email alerts) may keep it on: a duplicate alert is cheaper than a lost one. The backoff test in section 4 calls a test address and sends nothing to anyone, which is why it may use it.
- [ ] On Error is Stop Workflow, or Continue (using error output) with that output wired to an alert; never plain Continue. A failure passed on as success lets later steps record a message that never went out.
- [ ] Every send writes a "pending" record before the send and marks it "sent" after. A record still at "pending" is a "may have landed" case.
- [ ] Running the workflow again for the same input sends nothing, because it reads that record first and stops.

---

## 7. Maintenance

- Re-run the fault injector after any change to the handler, and once a quarter regardless.
- Review `knownConditions` when a mute expires: fix the cause or extend the date consciously.
- If the Slack channel is renamed or replaced, update `{SLACK_CHANNEL_ID}` and re-fire P2.
- If your log database gains or loses properties, update Format Error to match; the write fails otherwise (and the alert still goes out, saying so).
