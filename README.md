# inbox-triage-action

A GitHub Actions template. Each day it reads a mailbox or a form feed. It groups the messages with Gemini. It posts one summary to Slack.

## Files

- `.github/workflows/inbox-triage.yml` — the schedule and the settings.
- `scripts/triage.py` — fetch, scrub, group, post.
- `scripts/requirements.txt` — one dependency (`requests`).

## Set up

1. Copy the three files into your repository. Keep the paths.
2. Add these repository secrets (Settings → Secrets and variables → Actions):

| Secret                                    | Use                                           |
| ----------------------------------------- | --------------------------------------------- |
| `GEMINI_API_KEY`                          | Key from Google AI Studio                     |
| `SLACK_WEBHOOK_URL`                       | Slack incoming webhook for the target channel |
| `IMAP_HOST`, `IMAP_USER`, `IMAP_PASSWORD` | Mailbox source. Use an app password.          |
| `FEED_URL`, `FEED_TOKEN`                  | Form feed source (optional token)             |

3. In the workflow, set `SOURCE`, `TRIAGE_CONTEXT`, `CATEGORIES`, and `REPORT_TITLE`.
4. Run the workflow by hand with **dry_run** on. Read the log. Check that the scrubbed items look correct.
5. Run it again with dry_run off.

## Sources

| `SOURCE` | Needs                                     | Notes                                                                                   |
| -------- | ----------------------------------------- | --------------------------------------------------------------------------------------- |
| `imap`   | `IMAP_HOST`, `IMAP_USER`, `IMAP_PASSWORD` | Opens the folder read only. Messages stay unread.                                       |
| `http`   | `FEED_URL`                                | JSON list of objects, or an object with a list under `FEED_LIST_KEY` (default `items`). |
| `file`   | `FEED_FILE`                               | Same JSON shape from a local file. Use it for tests.                                    |

For `http` and `file`: `FEED_TEXT_FIELD` (default `text`) and `FEED_DATE_FIELD` (default `created_at`, ISO 8601) name the fields.

## Safety

- The script removes email addresses, URLs, and phone-like numbers before it calls Gemini.
- It tells Gemini that message text is data, not instructions. This lowers the risk of prompt injection. It does not remove it. Do not give the job any secret it does not need.
- Gemini returns structured JSON. The script builds the Slack message itself. Model text never goes to Slack as free text, except the one-line summary.
- Messages go to the Gemini API. Check that your data policy allows this.
- The job has `contents: read` permission only.

## Settings

| Variable         | Default                                | Meaning                                            |
| ---------------- | -------------------------------------- | -------------------------------------------------- |
| `LOOKBACK_HOURS` | `24`                                   | Age limit for messages                             |
| `MAX_ITEM_CHARS` | `1000`                                 | Characters kept for each message                   |
| `BATCH_SIZE`     | `100`                                  | Messages for each Gemini call                      |
| `GEMINI_MODEL`   | `gemini-2.5-flash`                     | Check the current model names before you change it |
| `CATEGORIES`     | `Bugs,Requests,Questions,Praise,Other` | Comma list                                         |
| `TRIAGE_CONTEXT` | `A product inbox.`                     | One line that tells Gemini what the inbox is       |

## Test locally

```sh
pip install -r scripts/requirements.txt
echo '[{"text":"App crashes on login. Mail me at a@b.com","created_at":"2030-01-01T00:00:00Z"}]' > /tmp/feed.json
SOURCE=file FEED_FILE=/tmp/feed.json LOOKBACK_HOURS=999999 DRY_RUN=1 python scripts/triage.py
```

## Limits

- No state between runs. If a run fails, the next run covers the same 24 hours plus new mail. Some messages can appear in two reports.
- Mail with only HTML parts gives an empty body. Only the subject is used.
- Very large inboxes: raise `BATCH_SIZE` carefully, or filter at the source.
