# Email Engagement Intelligence

A daily report that turns Mailchimp's campaign numbers into answers a B2B founder
can act on: **who** engaged, **how deeply**, and **whether it repeated**.

Mailchimp's dashboard tells you a campaign got a 38% open rate. It doesn't tell
you that the same four people at one target account opened the last three emails
and two of them clicked the pricing link. In a market with a few hundred
potential buyers, that second fact is the one that matters.

This tool pulls campaign performance plus link- and contact-level activity,
builds an Engagement Intelligence Report, and emails a short digest with the full
HTML/PDF report attached. It ran in production every morning for a B2B team,
on GitHub Actions, with no server to maintain.

## What the report contains

- **Executive summary**: the day's headline numbers, written as sentences rather than a wall of tiles
- **Campaign drill-down**: per-campaign opens, clicks and unsubscribes, newest first
- **Who clicked what**: contact-level click activity mapped to the links that matter
- **Repeat engagement**: contacts (with their company) who keep coming back across campaigns
- **Behavioral segmentation**: deterministic, documented buckets: no engagement → opened only → single clicker → repeat clicker → multi-link engager. Buckets only, with no opaque score.
- **Data-quality section**: which campaigns were excluded and why, with anomaly checks (for example Apple Mail Privacy Protection proxy artifacts), so the numbers can be trusted
- **Honest click rates**: built on Mailchimp's true people-count (`unique_subscriber_clicks`) rather than its per-link count, which can exceed 100% on multi-link emails
- **5 charts**: open-rate trend, click-rate trend, emails sent, opens vs clicks, campaign comparison (matplotlib, headless)

## How it's built

```
Mailchimp API ──▶ mailchimp_report_client.py   fetch campaigns + activity (incremental, since=)
                  mailchimp_engagement.py       per-contact / per-account engagement model
                  mailchimp_engagement_store.py state that survives between runs
                  mailchimp_metrics.py          the aggregates
                  mailchimp_report_summary.py   plain-language executive summary
                  mailchimp_report_charts.py    matplotlib → PNG
                  mailchimp_report_render.py    HTML report + PDF (weasyprint)
                  mailchimp_report_email.py     digest email over SMTP, attachments
generate_mailchimp_report.py  orchestrates the run
```

Design decisions that kept it reliable:

- **Incremental fetching.** Each campaign keeps its own event log, and every run asks Mailchimp only for activity since the last one.
- **State lives in the repo.** Actions runners are thrown away after each run, so the workflow commits the engagement logs and the daily snapshot back to the repo. Generated reports are 30-day artifacts and are never committed.
- **`TEST_MODE` safety switch.** The report generates normally but no email leaves until the switch is turned off, even when `--send` is passed.
- **Tested.** `pytest tests/`: 75 passing.

## Lessons from running it daily

**GitHub's scheduler warms up slowly.** In the first week the scheduled run fired
4.5 hours late on day 1 and not at all on day 2, then ran on time from day 3. Manual
runs worked throughout. If daily runs go quiet, check the Actions tab before
debugging the report. If punctuality matters, an external timer calling the
`workflow_dispatch` API is more reliable than native cron.

**`ubuntu-latest` moves under you.** When it switched to Ubuntu 26.04, the apt step
failed for three days (`libgdk-pixbuf2.0-0` has no installation candidate). The fix
was dropping a package that weasyprint 53+ no longer needs. If a run dies in about
10 seconds with exit code 100, read the apt step's log first.

## Run it

```bash
pip install -r requirements.txt
cp .env.template .env        # Mailchimp key, audience, SMTP; keep TEST_MODE=true at first
python3 tools/generate_mailchimp_report.py           # generate only
python3 tools/generate_mailchimp_report.py --send    # email too (blocked while TEST_MODE=true)
pytest tests/
```

To deploy on GitHub Actions, add the variables from `.env.template` as repository
secrets and uncomment the `schedule` lines in `.github/workflows/daily-report.yml`.
This copy ships with the schedule off.

---

Part of my GTM engineering work: [github.com/cyberanon1111](https://github.com/cyberanon1111) · [clarafold.com](https://clarafold.com)
