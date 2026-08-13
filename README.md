# kolosal-uptime

Off-box availability monitoring for **ocr.kolosal.ai**.

This repo contains no application code. It exists to answer one question from
outside all Kolosal infrastructure: *is the OCR API actually up?*

## Why it is a separate public repo

Two constraints forced this shape.

**It must not run on the box it watches.** On 2026-08-13 the OCR host went
dark at the network level — no ICMP, no SSH, no HTTPS. Every service lives on
that one host, so a monitor deployed alongside them would have died in the same
instant. The outage was reported by a customer before anyone internally noticed.

**It must be free to run often.** `KolosalAI/ocr.kolosal.ai` is private, and
private-repo Actions minutes are billed with a 1-minute minimum per run. A
5-minute cadence there costs ~8,640 min/month against a 2,000-minute free tier.
Public repos get unlimited free minutes, so the same cadence here costs nothing.

The tradeoff accepted: this repo's issue history makes downtime windows
publicly visible. That is the same information a public status page publishes
deliberately, but it is a real disclosure — if it ever stops being acceptable,
move the workflow into the private repo and drop the cadence to `*/30`.

## What it does

`.github/workflows/uptime.yml` runs every 5 minutes and probes
`https://ocr.kolosal.ai/health`.

A probe counts as healthy only when **all three** hold:

1. `curl` exits 0
2. HTTP status is `200`
3. the body contains `"status":"healthy"`

The third check is not redundant. Caddy fronts the app, and it can return a
`200` while the service behind it is broken — a status-code-only check would
call that healthy.

**Failure requires 3 consecutive failed attempts, 20s apart.** A single failed
curl from a GitHub runner is more often a runner hiccup than a real outage.

### Alerting

| Event | Action |
|---|---|
| 3 failed probes, no open alert | Opens an issue labeled `uptime` (emails you) |
| 3 failed probes, alert already open | Silent — no duplicate |
| Successful probe, alert open | Comments the recovery time and closes it |

Dedup matters: at a 5-minute cadence a one-hour outage triggers 12 runs. One
outage produces exactly one issue, not twelve emails.

The issue body embeds the curl exit code, HTTP status, and DNS/connect/TLS
timings, plus a table translating those into a likely cause. The 2026-08-13
incident required deriving all of that by hand.

Host names, IPs, SSH users and provider details are deliberately **not** in this
repo — it is public. The triage runbook lives in the private product repo under
`docs/ops/README.md`.

### Known limits

- **Detection latency is ~5–20 min, not 5.** GitHub's cron scheduler lags under
  load and occasionally skips ticks. This is not a sub-minute pager.
- **Alerts go to email**, via GitHub issue notifications. If that is not loud
  enough, add a Telegram step (`TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID` as
  repo secrets) to the failure path.
- **Prod only.** Staging is deliberately unmonitored — alerts you learn to
  ignore are worse than no alerts.
- `keepalive.yml` commits monthly so GitHub does not disable the schedule after
  60 days of inactivity.

## Setup

```bash
gh auth refresh -h github.com          # local token is currently invalid
gh repo create KolosalAI/kolosal-uptime --public --source=. --push
gh workflow run uptime.yml --repo KolosalAI/kolosal-uptime   # verify manually
```

Confirm the first manual run is green, then confirm the schedule takes over
within ~10 minutes. No secrets are required — the workflow uses the built-in
`GITHUB_TOKEN` with `issues: write`.

## Verifying it actually catches an outage

Do not trust an untested monitor. Point `TARGET` at a dead endpoint
(`https://ocr.kolosal.ai:9999/health`), dispatch manually, and confirm an issue
opens. Then restore `TARGET` and confirm the next run closes it.
