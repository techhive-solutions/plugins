---
name: google-workspace-steward
description: Find what is using Google Workspace pooled storage. Use when the user asks for the largest Google Drive files in their organization or for one user, who is using the most Drive storage, or why their Workspace storage is full. Runs an org-wide scan through the google-workspace-steward tools and presents a ranked report.
---

# Workspace Steward

Workspace Steward reads Google Drive **metadata only** (file name, size, owner, type,
last modified) across a Google Workspace organization. It never reads file contents
and cannot open, move, share or delete files.

## Tools

| Tool | Use it for |
|---|---|
| `start_scan` | An org-wide scan. Returns a `job_id` immediately, plus the org's `tier` and `max_files`. |
| `get_scan_status` | Progress of a running scan: `status` and `progress.done` / `progress.total` users. |
| `get_scan_results` | The finished report. While the scan is still running it returns the status instead. |
| `largest_files_for_user` | One user's largest files, by email. Fast, no job. |

## Org-wide scan

1. Call `start_scan`. The defaults (25 files per user, up to 500 overall) are right
   for most requests. Only pass `include_suspended=true` if the user asks about
   suspended accounts.
2. Tell the user the scan has started and that large organizations can take a few
   minutes. Note the `job_id`.
3. Call `get_scan_status` to follow progress and report it briefly
   ("120 of 300 users scanned"). Don't call it in a tight loop; check a few times,
   and if the scan is still running after several checks, give the user the
   `job_id` and offer to fetch the results when they ask again.
4. When `status` is `done`, call `get_scan_results`.

If the user only asks about one person, use `largest_files_for_user` with their
email instead of a full scan.

## Presenting the report

- Start with one line of totals: users scanned, files in the report,
  `total_size_human`.
- Then a ranked table: rank, file name, owner, size (convert `bytes` to KB/MB/GB),
  type (simplify `mimeType`, e.g. "Video (MP4)", "Google Doc"), last modified (date only).
- If the user asks who uses the most, also group the rows by `owner` and sum their sizes.
- List any `warnings` (for example users that could not be scanned) after the table.
  Never drop them silently.
- Only owned files in each user's My Drive are covered. Shared drives are not
  scanned yet, so say so if the user asks about them.
- Suggest concrete next steps, such as asking the owners of the largest files to
  delete or archive them. Steward can't do it for them.

## Free and premium

Each result says the org's `tier`.

- **free**: the report is capped at the top 3 files, whatever was requested. When
  results are capped, mention once, briefly, that the premium tier returns the full
  report, and that an org admin can manage the plan at
  https://gws-steward.zuhalsystems.ai/onboard. Don't repeat it in later replies or
  push it.
- **premium**: full report; nothing to mention.

The cap is enforced by the server. Don't try to work around it, for example by
calling `largest_files_for_user` for every user.

## Errors

- **"not authenticated"**: the user needs to sign in. In Claude this happens when
  connecting the Workspace Steward connector; ask them to reconnect it.
- **"no service-account key registered for <domain>"**: the organization hasn't
  been set up yet. A Google Workspace **super admin** has to register their
  organization's service-account key at https://gws-steward.zuhalsystems.ai/onboard.
  A personal gmail.com account has no organization to scan.
- **"<email> is not in your org"**: users can only be looked up in the signed-in
  user's own domain.
- **"no job with id ..."**: the job expired (results are kept for 24 hours) or the
  id is wrong. Start a new scan.