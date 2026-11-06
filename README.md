# splunk_data_replay_assist

A simple shell script to help collect data into a summary index at scale.

`splunk_replay_assist.sh` is an interactive generator. It builds a batch script that replays (backfills) a saved search into its summary index using Splunk's `fill_summary_index.py`. The time range is split into fixed-size `et`/`lt` windows so that each invocation stays within buffer limits.

## Requirements

- Linux host with Splunk installed (default `SPLUNK_HOME=/opt/splunk`)
- bash 4+ and GNU coreutils `date` (no other dependencies)

## Usage

```bash
chmod +x splunk_replay_assist.sh
./splunk_replay_assist.sh
```

You will be prompted for:

| Prompt | Maps to | Default |
|---|---|---|
| App context | `-app` | `search` |
| Saved search name | `-name` | `data_replay` |
| Concurrent searches | `-j` | `2` |
| Dedup | `-dedup` | `true` |
| Start / end time | `-et` / `-lt` | epoch or any GNU `date -d` string, e.g. `2026-04-08 00:00 UTC` |
| Window size | per-run span | seconds or with a unit: `21600`, `6h`, `1d`, `1w` |

Each time you enter is echoed back in UTC and local time for confirmation. The summary then shows how many captures will be created. You can proceed, adjust the window, adjust the dates or quit. On confirmation it writes an executable script with one `run N et lt` line per window.

### Running the generated script

```bash
./replay_data_replay_<et>_<lt>.sh                   # run all windows
START_AT=417 ./replay_data_replay_<et>_<lt>.sh      # resume from window 417
STOP_ON_ERROR=false ./replay_data_replay_<et>_<lt>.sh
LOG=/var/tmp/replay.log ./replay_data_replay_<et>_<lt>.sh
```

The script prompts once for Splunk credentials at runtime; they are never written to disk. Every window is logged with human-readable timestamps.

## Notes

- `fill_summary_index.py` runs one search per scheduled interval of the saved search *within* each window. The window size controls how much work each invocation gets, not the total number of searches.
- Keep `-dedup true` so that resuming and windows that share a boundary do not write duplicate summary data.
- Credentials are passed with `-auth` and are briefly visible in `ps` output for each run. On a shared host, consider an authenticated session instead.

## Version history

| Version | Date | Changes |
|---|---|---|
| 1.0.0 | 2026-11-06 | Initial release: interactive range and window selection, epoch to human-readable date display, capture count with proceed/adjust loop, generated script with logging, resume (`START_AT`) and error handling (`STOP_ON_ERROR`). |

## Licence

GPL-3.0. See [LICENSE](LICENSE).
