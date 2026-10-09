# board-pulse

The heartbeat of the board hygiene job. One file, `pulse.json`, rewritten after
every pass the job makes over the GitHub boards under `condechi-llc`. If this
file is fresh, the boards were just swept. If it is stale, nothing has swept
them, and whatever they show is only as current as the record here.

This repository is public on purpose: an observer on another host reads it
with no token, and nothing in a run record is secret.

## Who writes, who reads

| | Where | What |
| --- | --- | --- |
| Writer | xilonen, `board-hygiene.timer`, every ten minutes | `hygiene/run.sh` in [condechi-llc/dotfiles](https://github.com/condechi-llc/dotfiles) appends the record to its local log first, then PUTs the same record here |
| Reader | nox, hourly cron | `hygiene/observe.sh` from the same repo: opens one issue in dotfiles when the record is older than 25 minutes or says `failed`, closes it when the heartbeat returns |
| Reader | any laptop | `doctor.sh` in dotfiles reads it and says whether the boards are current before you act on them |

The scripts are **not** here. The sweep has one implementation, in
`claude/skills/ship/scripts/` of dotfiles, and the runner, timer, installer and
observer sit beside it in `hygiene/`. A second copy here would be a second
thing to drift. `hygiene/README.md` there is the runbook.

## Reading it by hand

```sh
curl -s https://raw.githubusercontent.com/condechi-llc/board-pulse/main/pulse.json | jq .
```

The two fields that matter: `finished_at` (how old is this) and `status`
(`ok` or `failed`, with `failed_steps` naming which). Everything else is
telemetry: what each script printed, and what GitHub answered.

## The record

Every value is something a script printed or GitHub returned. Nothing is
estimated.

| Field | Meaning |
| --- | --- |
| `run_id` | UTC timestamp of the start plus the runner's pid; unique per pass |
| `host` | the machine that ran it |
| `started_at`, `finished_at`, `duration_s` | wall clock of the pass, UTC |
| `dry_run` | `true` when the pass wrote nothing to any board (first run after install, or a check by hand) |
| `owners` | the GitHub owners swept; boards are discovered, never listed |
| `dotfiles_sha`, `pull_ok` | which commit of the scripts ran, and whether the clone pulled before running |
| `github.login` | the identity the token resolves to. Never the token |
| `github.graphql_remaining_before/after`, `graphql_limit`, `graphql_reset_epoch` | the GraphQL budget around the pass; a pass costs on the order of 60 points of 5,000 an hour |
| `boards_discovered` | how many boards the sweep found linked to a repository |
| `steps.actuals` | `written`: `Actual start`/`Actual end` values stamped; `over_without_landing`: closed items left deliberately unmeasured for a person to rule on |
| `steps.invariants` | `repaired`: states fixed; `passes`: loops until stable; `needing_decision`: states only a human can resolve; `repairable` is the dry-run count |
| `steps.drift` | `matching`/`total`: boards that still match the template |
| `steps.status_update` | `posted`: boards whose status update changed and was published; `unchanged`: boards where it was identical and nothing was posted |
| `status`, `failed_steps` | `failed` when actuals, invariants or the status update exited non-zero. Drift never fails a run |

`pulse.example.json` is a real record from a dry run on 2026-10-06, kept here
so the shape is documented by a specimen rather than by prose.

## History

The commit log of this repo is the run history: one commit per pass, message
`pulse <run_id> <status>`. `git log --format='%s' | head` is the last few
passes; `git log -p pulse.json` is the whole telemetry series. Nothing else is
ever committed here.
