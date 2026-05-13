# aha-bot-charts

Shared infrastructure for AhaSlides analytics routines that post weekly digests to Slack. A scheduled Claude Code remote agent (CCR) does the analysis, then hands off the final Slack post and the Metabase data fetches to GitHub Actions workflows in this repo.

If you're spinning up a new routine, you can probably reuse 90% of what's already here. Read the playbook below.

---

## Architecture

```
                       ┌───────────────────────┐
                       │   CCR scheduled       │
                       │   (claude.ai)         │
                       │   prompt + cron       │
                       └──────────┬────────────┘
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        │ Mixpanel MCP            │ Reads (via raw.gh…)      │ Writes (PUT /contents)
        │ (allowlisted)           │ data/metabase/*.json     │ charts/*.png + payloads/*.json
        ▼                         │                          ▼
  ┌────────────┐                  │                  ┌──────────────────┐
  │  Mixpanel  │                  │                  │  this GitHub     │
  │  EU        │                  │                  │  repo            │
  └────────────┘                  │                  └─────┬────────────┘
                                  │                        │
                                  │     ┌──────────────────┴──────────────────┐
                                  │     │                                     │
                                  │     ▼                                     ▼
                                  │  ┌──────────────────────┐    ┌──────────────────────┐
                                  │  │ fetch-metabase.yml   │    │ post-to-slack.yml    │
                                  │  │ cron 01:00 UTC daily │    │ trigger: push to     │
                                  │  │ hits bi.ahaslide.com │    │ payloads/**/*.json   │
                                  └──┤ writes data/...json  │    │ POSTs to Slack       │
                                     └──────┬───────────────┘    │ chat.postMessage     │
                                            │                    └──────┬───────────────┘
                                            ▼                           ▼
                                     ┌────────────┐               ┌────────────┐
                                     │  Metabase  │               │   Slack    │
                                     │ (bi.aha…)  │               │ (#channel) │
                                     └────────────┘               └────────────┘
```

### Why the convoluted setup

The CCR sandbox has an egress allowlist. From inside a routine:

- **Reachable**: `api.github.com`, `raw.githubusercontent.com`, `s3.amazonaws.com`, `storage.googleapis.com`, plus the MCP connector hosts.
- **Blocked**: `hooks.slack.com`, `slack.com/api/*`, `files.slack.com`, `bi.ahaslide.com`, all of Cloudflare (`*.workers.dev`, `*.r2.dev`, `*.cloudflare.com`), Cloudinary, Imgur, Vercel Blob, public dump hosts (`0x0.st`, `catbox.moe`, `tmpfiles.org`).
- **Important**: do NOT pass `dangerouslyDisableSandbox: true` in bash calls. Disabling the sandbox cuts off the allowlisted hosts too. Default sandbox = working network.

So Slack and Metabase have to be proxied through this repo. The agent reads from / writes to GitHub via the Contents API (which is allowlisted), and GitHub Actions running on GitHub-hosted runners reach the otherwise-blocked services.

---

## What's already set up in this repo

### GitHub secrets

| Secret | Used by | Purpose |
|---|---|---|
| `SLACK_BOT_TOKEN` | `post-to-slack.yml` | Bot user OAuth token for the `Dave's Assistant` Slack app. Scope: `chat:write`. |
| `METABASE_API_KEY` | `fetch-metabase.yml` | Header `x-api-key` for `bi.ahaslide.com/api/*`. |

### Workflows

- **`.github/workflows/post-to-slack.yml`** — triggers on push to `payloads/**/*.json`. For each new JSON file, POSTs the body to `slack.com/api/chat.postMessage`. The JSON file IS the full Block Kit payload (must include `channel`, optionally `blocks`).
- **`.github/workflows/fetch-metabase.yml`** — runs on cron (`0 1 * * *` UTC) and on `workflow_dispatch`. Queries one Metabase question (default 5600), transforms to clean JSON, commits to `data/metabase/<question_id>.json`. To add another question, either pass `question_id` to dispatch or extend the workflow to fetch a list.

### Slack app

- **Name**: `Dave's Assistant` (bot user: `interest_scout`)
- **Workspace**: AhaSlides (team ID `TF9179056`)
- **Scopes**: `chat:write` (only — `users:read`, `users:read.email`, `channels:read` are NOT granted, so user/channel IDs need to be obtained manually)
- The bot must be `/invite`'d to every channel it posts to.

### claude.ai connectors

- **Mixpanel EU MCP** — `https://mcp-eu.mixpanel.com/mcp` (project 2668743). Used directly from routines for `Get-Report`, `Run-Query`, etc.
- **Slack MCP** — `https://mcp.slack.com/mcp`. Limited (no Block Kit, no `unfurl_*` flags) — only used as a fallback for plain-text failure messages.

### Repo layout

```
charts/<ISO-WEEK>-<routine>-<rand>.png   # chart images, public via raw.gh…
payloads/<ISO-WEEK>-<routine>-<rand>.json # Block Kit payloads (the post-to-slack workflow eats these)
data/metabase/<question-id>.json         # daily Metabase snapshots
.github/workflows/                       # the two proxy workflows
```

---

## Playbook: spinning up a new analytics routine

### Quick checklist

- [ ] Decide channel; `/invite @interest_scout` in Slack; copy channel ID.
- [ ] Decide schedule (write the cron in UTC; Hanoi = UTC+7).
- [ ] Identify Mixpanel event(s) and breakdown property.
- [ ] If a new Metabase question is needed: add it to `fetch-metabase.yml` and run the workflow once via `gh workflow run` so the data file exists.
- [ ] Collect any user IDs you want to `@mention` (Slack profile → ⋯ → Copy member ID).
- [ ] Fill the prompt template (below). Save a copy in a scratch file.
- [ ] Create the routine via `/schedule` skill or `RemoteTrigger` API.
- [ ] Smoke test: create a `run_once_at` clone of the routine that fires in 2–3 min. Watch the session log AND the Actions tab.
- [ ] When the Slack post looks right, leave the production routine running.

### Slack channel prep

In Slack, in the target channel:

```
/invite @interest_scout
```

Click the channel name at the top → scroll to the bottom of the modal → **Channel ID** → copy. You'll need it for the prompt.

### Adding a new Metabase question to the daily fetch

The current workflow fetches one question per dispatch (defaults to 5600). For a recurring fetch of additional questions, the simplest extension is to make the cron run multiple questions in a `for` loop. For a one-off, just trigger via `gh`:

```
gh workflow run fetch-metabase.yml -f question_id=<id>
```

After the run completes, the file lives at `data/metabase/<id>.json` and the routine reads it from `https://raw.githubusercontent.com/mrdungx/aha-bot-charts/main/data/metabase/<id>.json`.

### Finding user IDs for mentions

Click profile picture → click **⋯ More** → **Copy member ID**. Format: `U0XXXXXX`. In the Slack message text, use `<@U0XXXXXX>` — bare `@name` won't tag.

### Creating the routine

Use the `/schedule` skill in Claude Code. Provide:

- **Name**: descriptive (`Slide-type signals — weekly`, `Onboarding funnel digest — weekly`, etc.)
- **Cron**: UTC. Examples: `0 7 * * 1` = Mon 07:00 UTC = Mon 14:00 Hanoi.
- **Model**: `claude-opus-4-7` (or `claude-sonnet-4-6` if the analysis is simpler and you want it cheaper).
- **Environment**: `env_01DRwxy8Jsm4J25bgFTiz5g2` (default Anthropic cloud).
- **MCP connections**: `mixpanel-eu` (analysis) and `slack` (failure fallback). If your routine only writes payloads to the repo and never needs to query Slack for failure messages, you can drop the Slack MCP.
- **Allowed tools**: `Bash, Read, Write, Edit`.
- **Prompt**: see template below.

---

## Prompt template

This is the boilerplate. Replace anything in `<<...>>` with your specifics.

````markdown
<<One-line description of the routine, e.g. "Weekly onboarding-funnel review for AhaSlides, posted to #growth-signals every Tuesday 10:00 Hanoi.">>

<<2–3 sentence framing: what's the audience for the post, what should they take away, what's the comparison if any?>>

British English. Direct, no flattery. If nothing is notable, say so plainly. A light touch of wry humour is welcome but never forced — drop it if the week is flat or numbers are bad.

The CCR sandbox cannot reach Slack endpoints. Posts are proxied via `mrdungx/aha-bot-charts`: push a Block Kit payload JSON, a workflow in that repo posts it to Slack.

## Constants

```bash
GITHUB_PAT='<<paste fine-grained PAT with Contents:write on aha-bot-charts>>'
REPO='mrdungx/aha-bot-charts'
SLACK_CHANNEL_ID='<<C0XXXXXXX — your target channel>>'
<<Optional: METABASE_DATA_URL='https://raw.githubusercontent.com/mrdungx/aha-bot-charts/main/data/metabase/<id>.json'>>
<<Optional: MENTION_FOO='<@U0XXXXXX>'>>
```

## Step 1 — Mixpanel queries

Use Mixpanel EU MCP. Project ID 2668743.

<<Specify the queries here. Typically:
- Which event(s)?
- Math (`total` / `unique` / `dau` etc.)
- Breakdown property (if any)
- Time range (`last 10 weeks`, `last 30 days`, etc.)
- Filters (or no filter if you need everything)>>

<<Optionally, point at a Mixpanel saved-report bookmark for ground truth on property names: "Inspect bookmark_id NNNNN via Get-Report (skip_results=false) to confirm the property name.">>

## Step 2 — Fetch Metabase data (optional, if using)

```bash
curl -sSL "$METABASE_DATA_URL" -o /tmp/metabase.json
```

Schema: `{"fetched_at": ..., "rows": [{... columns from the question ...}, ...]}`. The file is refreshed daily by `.github/workflows/fetch-metabase.yml`.

If the curl fails or the JSON is malformed, treat Metabase as UNAVAILABLE — continue Mixpanel-only. Don't fail the whole routine.

<<IMPORTANT: describe exactly what each row/column means and what "high" or "low" actually represents. This is where misinterpretations creep in. List banned and preferred phrasings if the metric is easily misread.>>

## Step 3 — Analysis

<<List the calculations:
- A. Trajectory of the key metric — current value, WoW change, 4-week trailing avg, direction (📈 growing / ➖ flat / 📉 fading)
- B. Comparisons against a peer cohort
- C. Rankings / movements
- D. Divergences between metrics
Be specific about which group / cohort each comparison runs against. Define the cohort explicitly.>>

<<Small-sample rule: if anything is below a meaningful volume threshold, flag it and avoid WoW conclusions.>>

## Step 4 — Chart (optional)

Python + matplotlib. PNG at `/tmp/<routine>-week.png`, ~1000×600px.

<<Spec the chart:
- Type (line / bar / etc.)
- X-axis, Y-axis
- Reference band or comparison series
- Title and subtitle
- Legend position, font, etc.>>

`pip install matplotlib` if needed.

### Upload the chart to GitHub

```bash
ISO_WEEK=$(date -u +'%G-W%V')
SUFFIX=$(openssl rand -hex 3)
CHART_PATH="charts/${ISO_WEEK}-<<routine>>-${SUFFIX}.png"
CONTENT_B64=$(base64 -w0 < /tmp/<<routine>>-week.png 2>/dev/null || base64 < /tmp/<<routine>>-week.png | tr -d '\n')
export CHART_PATH CONTENT_B64

RESPONSE=$(curl -sS -X PUT \
  -H "Authorization: Bearer ${GITHUB_PAT}" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/${REPO}/contents/${CHART_PATH}" \
  -d "$(python3 -c 'import json,os; print(json.dumps({"message":"chart: "+os.environ["CHART_PATH"],"content":os.environ["CONTENT_B64"]}))' )")

CHART_URL=$(echo "$RESPONSE" | python3 -c 'import json,sys; print(json.load(sys.stdin)["content"]["download_url"])' 2>/dev/null || echo "")
echo "CHART_URL=$CHART_URL"
```

If `CHART_URL` is empty, omit the `image` block from the payload and note the failure in the post.

## Step 5 — Build the Block Kit payload

Slack mrkdwn inside section blocks:
  • Bold (`*text*`): only the title and named subheadings.
  • Bullets: `— ` (em-dash + space). Slack's mrkdwn does NOT render `- ` as a list reliably; use the em-dash.
  • Links: `<https://url|label>`.
  • Direction markers: `📈 growing`, `➖ flat`, `📉 fading`.
  • Mentions: `<@U...>` user-id syntax.

### Section text format

<<Provide an explicit template here with all subheadings, bullets, and the source/mention footer. Show conditional inclusions (e.g. only include line X if condition Y).>>

Example layout:

```
*<<ROUTINE TITLE — DATE RANGE>>*

*🔸 <<Subhead 1>>:*
— bullet
— bullet

*🔸 <<Subhead 2>>:*
— bullet
— bullet

*🔸 Notable:*
— 2–3 bullets max, only genuinely interesting

<https://...|Source A> · <https://...|Source B>

<@U0XXXXXX> <@U0YYYYYY>
```

Wrap the section text into a payload:

```json
{
  "channel": "<<SLACK_CHANNEL_ID>>",
  "text": "<<short summary for notifications>>",
  "blocks": [
    {"type": "section", "text": {"type": "mrkdwn", "text": "<<section text>>"}},
    {"type": "image", "image_url": "<<CHART_URL>>", "alt_text": "<<alt>>"}
  ]
}
```

Save to `/tmp/payload.json`. Validate: `python3 -c 'import json; json.load(open("/tmp/payload.json"))'`. If the JSON is broken, fall to the failure path.

## Step 6 — Push the payload (triggers the Slack post)

```bash
PAYLOAD_PATH="payloads/${ISO_WEEK}-<<routine>>-${SUFFIX}.json"
PAYLOAD_B64=$(base64 -w0 < /tmp/payload.json 2>/dev/null || base64 < /tmp/payload.json | tr -d '\n')
export PAYLOAD_PATH PAYLOAD_B64

RESP=$(curl -sS -X PUT \
  -H "Authorization: Bearer ${GITHUB_PAT}" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/${REPO}/contents/${PAYLOAD_PATH}" \
  -d "$(python3 -c 'import json,os; print(json.dumps({"message":"payload: "+os.environ["PAYLOAD_PATH"],"content":os.environ["PAYLOAD_B64"]}))' )")

echo "$RESP" | head -c 300
COMMIT_OK=$(echo "$RESP" | python3 -c 'import json,sys; d=json.load(sys.stdin); print("ok" if "commit" in d else "")' 2>/dev/null || echo "")
echo "COMMIT_OK=$COMMIT_OK"
```

On `COMMIT_OK=ok`, the `post-to-slack` workflow takes over and posts to Slack in ~30–60s. Session ends.

## Failure path

If any step fails irreparably, post a plain-text failure message via the Slack MCP (`slack_send_message` to the channel id). Sessions must never end silently.

## Iron rules

- Default bash sandbox throughout. Never disable.
- Session ends with EITHER a successful payload push OR a plain-text failure message via Slack MCP.
- Bullets: `— ` (em-dash + space) only.
- Bold (`*...*`): ONLY title + subheadings.
- Section text under ~3000 chars (Slack section limit).
- No data-freshness caveats unless the data is actually broken (zero everywhere).
- <<Any analysis-specific rules: small-sample thresholds, banned phrasings, etc.>>
````

---

## Reference: existing routines

| Routine | Channel | Schedule | ID |
|---|---|---|---|
| Slide-type signals — weekly | `#customer-signals` | Mon 07:00 UTC | `trig_01QUo6D6Z4xWfhB8hAkj5647` |

---

## Gotchas and known limitations

- **Egress allowlist** (above). Probe before committing to a new third-party host — run a one-off routine that just `curl`s the candidate and reports the result.
- **The Slack MCP doesn't support `blocks` or `unfurl_*` flags.** It's plain-text only. That's why every formatted post has to go through the GitHub Actions proxy. Don't try to make the MCP do more than it can.
- **`*bold*` vs `**bold**` rendering** depends on the path. When sending directly via Slack API (`chat.postMessage` with `blocks` → `mrkdwn` text), single asterisks = bold. When sending through some MCP wrappers that pre-convert Markdown → mrkdwn, double asterisks = bold. We use the direct API (via the GitHub Actions proxy) so **single asterisks** for bold.
- **GitHub PATs with only `contents:write`** cannot write to `.github/workflows/`. Adding/updating workflow files needs the `workflow` scope. The `gh` CLI auth has it; ad-hoc PATs usually don't.
- **`run` on a cron trigger returns 500.** To fire a recurring routine on demand, create a `run_once_at` clone instead.
- **Metabase rows often include aggregate "meta" rows** (e.g. `Any interactive slides`). Exclude these by name before ranking.
- **Slack channel/user IDs cannot be looked up via the bot token** because we didn't grant `users:read`, `users:read.email`, or `channels:read`. Obtain them manually (right-click profile / channel) and paste into the prompt as constants.
- **Local-machine ping ≠ CCR sandbox reachability.** A `curl https://hooks.slack.com/...` from a laptop succeeding tells you nothing about whether the routine can reach it. Probe from inside the sandbox.

---

## Reusable IDs (Slack workspace `TF9179056`)

| Thing | ID |
|---|---|
| Channel `#customer-signals` | `C04QBRY1P34` |
| Channel `#domodu-test` | `C090AJTQEK1` |
| User Dave | `UF9HYBY2G` |
| User Trent | `U05PK31G152` |
| Bot user `interest_scout` | `U0AS9CWT2N7` |

Add new IDs to this table as you discover them.
