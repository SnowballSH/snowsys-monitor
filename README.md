# snowsys-monitor

Scheduled black-box monitoring of my public web surfaces: uptime, redirects,
DNS, certificate expiry, and the public Snowblog API/access-control contract.
Every probe lives in
[`.github/workflows/external-acceptance.yml`](.github/workflows/external-acceptance.yml)
— that file is the probe inventory. All targets are already publicly
observable. A failing run emails the owner. No credentials, no deploy access —
probes only.

## What a run checks

Each run prints its own total (`all N checks passed`, or `F of N checks
failed persistently`). On 2026-09-25 that was 58: 42 named checks plus the
21-day certificate margin on each of the 16 public vhosts. A check retries
once after 10 seconds before it counts as failed.

The gated checks (admin, files, grafana, deploy, llm-admin) expect `403`
because the runner is outside the tailnet. Run from a tailnet device, those
checks fail, as they should.

## How often it actually runs

The cron asks for every 30 minutes (`7,37 * * * *`). GitHub runs scheduled
workflows on a best-effort basis: they "can be delayed during periods of
high loads", and "some queued jobs may be dropped"
([events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)).
Here, most slots are dropped. The 98 scheduled runs from 2026-09-09 to
2026-09-25 had a median gap of 3.9 hours, a 95th percentile of 5.6 hours, and
a maximum of 6.0 hours. Across all 617 scheduled runs since 2026-08-08, the
longest gap was 13.4 hours, during the 2026-08-27/28 throttling. In practice,
this is a check every 2–6 hours, not every 30 minutes.

## Keeping the schedule alive

"In a public repository, scheduled workflows are automatically disabled when
no repository activity has occurred in 60 days"
([disabling and enabling a workflow](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-workflow-runs/disabling-and-enabling-a-workflow)).
This repository can easily go quiet for that long. The `keepalive` job
therefore calls the documented
[enable-a-workflow](https://docs.github.com/en/rest/actions/workflows#enable-a-workflow)
endpoint on every scheduled run, so no commits are needed. This is the
mechanism behind
[liskin/gh-workflow-keepalive](https://github.com/liskin/gh-workflow-keepalive).
It is a single `gh api` call, so the workflow runs that call directly rather
than depending on a third-party action. GitHub does not document that this
call resets the inactivity clock, so the result is checked from outside:

- The private `snowsys` repository's `monitor-deadman` workflow runs every 4
  hours and fails, emailing the owner, in three cases: this workflow is not
  `active`; its newest successful scheduled run is more than 18 hours old;
  or scheduled runs no longer act as the owner. Private repositories are not
  subject to the 60-day disable. The third case matters because GitHub sends
  scheduled-run notifications to the user who re-enabled a disabled
  workflow. If an API re-enable ever made the bot the recipient, failure
  email would stop, and the dead-man would catch it.
- To restore a disabled schedule: `gh workflow enable external-acceptance.yml
  -R SnowballSH/snowsys-monitor`, run as the owner.

## Proving that a red run reaches the owner

Run the workflow once with the forced-failure input. It adds one check that
always fails, so the run goes red:

```sh
gh workflow run external-acceptance.yml -R SnowballSH/snowsys-monitor -f force_failure=true
```

The run should end with `1 of N checks failed persistently` (more if a real
check is also failing), and a failure email should arrive. This proves
delivery for a run the owner triggered. Scheduled runs notify the user
their `actor` shows, and `monitor-deadman` checks that this is the owner.
