# Live-test log

Every bet, every recommendation (taken or not), every close and every settlement from a live
betting test, published automatically within minutes of being logged. Nothing is edited or deleted:
the logs are append-only and a correction is a new row.

## Files

| file | one row per |
|---|---|
| `data/notified.csv` | recommendation shown by the scanner, whether bet or not |
| `data/bets.csv` | bet placed: book, odds on the slip, stake, Pinnacle fair probability at bet time |
| `data/skips.csv` | recommendation not bet, and why (price gone, below floor, stake refused, ...) |
| `data/closes.csv` | bet closed against Pinnacle's de-vigged closing price: `clv = odds_slip * q_close - 1` |
| `data/settlements.csv` | settled result and payout |

## Proof of time

Each change writes `snapshots/<UTC>.txt` with the sha256 of every file in `data/`, and that manifest
is timestamped with [OpenTimestamps](https://opentimestamps.org) (anchored in Bitcoin a few hours
later). Git commit dates are set by the committer and prove nothing; the `.ots` stamp does.

To check that a bet was public before its kickoff:

```bash
git log --diff-filter=A -S '<bet_id>' -- data/bets.csv   # the commit that first added it
git checkout <commit>
shasum -a 256 data/*                                      # must match snapshots/<UTC>.txt of that commit
ots verify snapshots/<UTC>.txt.ots                        # time of the stamp, from Bitcoin
```
