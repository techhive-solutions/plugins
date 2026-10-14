---
name: google-workspace-steward
description: Find what is using Google Workspace pooled storage. Use when a Google Workspace super admin asks for the largest Google Drive files in their organization, who uses the most Drive storage, how storage is growing, or why their Workspace storage is full; for one person's files, look up just that user. Presents a ranked report through the google-workspace-steward tools.
---

# Google Workspace Steward

Google Workspace Steward reads Google Drive **metadata only** (file name, size, owner,
type, last modified) across a Google Workspace organization. It never reads file
contents and cannot open, move, share or delete files.

Every tool is for **super admins of the user's Google Workspace only**. The server
checks this at sign-in and refuses everyone else (see Errors).

## Tools

| Tool | Use it for |
|---|---|
| `start_scan` | An org-wide scan. Returns a `job_id` immediately, plus the org's `tier` and `max_files`. |
| `get_scan_status` | Progress of a scan: `status` and `progress.done` / `progress.total` (users and shared drives). |
| `get_scan_results` | The finished report. While the scan is still running it returns the status instead. |
| `largest_files_for_user` | One user's largest files, by email. Fast, no job. Premium only. |
| `storage_top_users` | Users ranked by total Drive storage, from Google's daily usage reports. No scan needed. |
| `storage_growth` | How total storage changed over 30/90/180 days, and who grew the most. |

## One person or the whole organization

- About **one person** ("what's using alice@... storage?"): call
  `largest_files_for_user` with their email. Don't start an org-wide scan. It's
  Premium only: on the free plan it returns an error, so offer an org-wide scan
  instead.
- **Who uses the most storage, or how fast it's growing**: use `storage_top_users`
  or `storage_growth` (see Storage trends). No scan needed.
- **The largest files across the organization**: run an org-wide scan.

## Org-wide scan

1. Call `start_scan` with the defaults (25 files per user, up to 500 overall). Only
   pass `include_suspended=true` if the user asks about suspended accounts.
2. Call `get_scan_status` **once**. You can't wait between calls, so don't poll in a
   loop.
   - `done`: call `get_scan_results`.
   - `failed`: tell the user the scan failed and show the `error` field. Don't retry
     automatically; offer to start a new scan if they want.
   - `queued` or `running`: tell the user the scan is running (for example "120 of
     300 users and shared drives scanned so far"), that large organizations can
     take a few minutes, and give them the `job_id`. When they ask again, call
     `get_scan_status` or `get_scan_results` with that `job_id`.

## Presenting the report

The finished report from `get_scan_results` has the ranked files in `results` and
its own `warnings` list. Use those. Don't repeat warnings you saw in
`get_scan_status` while the scan was running; the report's `warnings` is the final
list.

- Start with one line of totals: users and shared drives scanned, files in the
  report, `total_size_human`.
- Then a ranked table from `results`: rank, file name, owner, size (convert `bytes`
  to KB/MB/GB), type (simplify `mimeType`, e.g. "Video (MP4)", "Google Doc"), last
  modified (date only).
- Files in shared drives are in `shared_drive_results`, with the drive's name
  instead of an owner. Show them as a separate table. `shared_drives` lists those
  drives: its `top_n_bytes` is only the sum of the drive's largest files, not the
  drive's total size.
- If the user asks who uses the most, also group the rows by `owner` and sum their
  sizes.
- List the report's `warnings` (for example users or drives that could not be
  scanned) after the tables. Never drop them silently.
- Suggest concrete next steps, such as asking the owners of the largest files to
  delete or archive them. Steward can't do it for them.

## Storage trends

For "who uses the most storage" or "how fast is our storage growing", use the
report tools instead of a scan. They're instant, but Google's usage data is 1-5
days old.

- `storage_top_users`: users ranked by total Drive storage.
- `storage_growth`: change in total storage over 30, 90 and 180 days, and the
  users who grew the most.
- Always say which day the numbers are from (`as_of`).
- If a period has a `note` (for example users who left the org), pass it on.
- If the answer has an `error` saying storage trends are off, show the `error`
  and the `fix` as they are. Scans still work without them.

## Free and premium

Each result says the org's `tier`, and `start_scan` returns `max_files`: the most
files the report will contain.

- **free**: the report is capped at the top 3 files in total, whatever was
  requested, and user lists from the storage tools at 3 users. You can tell a
  result is capped when `max_files` is lower than the number you asked for (500 by
  default), or when a storage-tool answer has `capped_by_tier`. When results are
  capped, mention once, briefly, that the premium tier returns the full report, and
  that a super admin can manage the plan at
  https://gws-steward.zuhalsystems.ai/onboard. Don't repeat it in later replies or
  push it.
- **premium**: full report; nothing to mention.

The cap is enforced by the server. On the free plan `largest_files_for_user`
isn't available at all (it would let the report be rebuilt user by user), so
don't try to work around the cap.

## Errors

- **"not authenticated"**: the user needs to sign in. In Claude Code: run `/mcp`
  and authenticate `google-workspace-steward`. In the Claude app: reconnect the
  Google Workspace Steward connector.
- **"these tools are for super admins…"**: the signed-in user isn't a super admin of
  their Google Workspace (regular users and delegated admins don't qualify). Tell
  the user in one line, as it is. Every tool is gated the same way, so don't retry
  or try another tool. Suggest asking a super admin to run it, or reconnecting with
  a super admin account. If they were just made an admin, or just registered the
  key, reconnecting signs them in again with the new status.
- **"no service-account key registered for <domain>"**: the organization hasn't
  been set up yet. A Google Workspace **super admin** has to register their
  organization's service-account key at https://gws-steward.zuhalsystems.ai/onboard.
  A personal gmail.com account has no organization to scan.
- **"Looking up one user's files is a Premium feature"**: the org is on the free
  plan. Say so once, offer an org-wide scan (top 3 files) instead, and mention
  that a super admin can upgrade at https://gws-steward.zuhalsystems.ai/onboard.
- **"<email> is not in your org"**: users can only be looked up in the signed-in
  user's own organization.
- **"no job with id ..."**: the job expired (results are kept for 24 hours) or the
  id is wrong. Start a new scan.
