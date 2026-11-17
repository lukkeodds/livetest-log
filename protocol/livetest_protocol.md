# Live test — protocol (FROZEN 14 Nov 2026)

**Status: frozen** after the book-side check (amendment 1, §9), before the first bet. Hash in
`docs/freeze.log`, OpenTimestamps stamp next to this file, published to the public log (§7) before the
first bet. Nothing in it changes during the test except by a dated amendment that applies forward only.

Preconditions for the first bet, both met 14 Nov 2026:

- the book-side check on the confirmed cells is done, and the cell it removed is recorded in §9;
- this file is frozen and published.

## 1. Purpose

Measure, with real money, whether the CONFIRMED cells' edge survives contact with the books:

- **CLV on bets actually placed** (primary): slip odds against Pinnacle's de-vigged close, under
  both multiplicative and power de-vig. Profit is reported, not judged — at odds ~2 the SD of
  profit per bet is ~1 stake, so a 3 % return needs thousands of bets; CLV needs tens.
- **Takeability**: how often the feed price is still on the slip, at or above the EV floor.
- **Limits**: when and how each book restricts stakes, and after how many bets.

## 2. What is bet

- Only the **21 cells** left of the 22 CONFIRMED cells of the frozen verdict of 14 Nov 2026
  (`data/confirm/verdict_2026-11-14.txt`, hash in `docs/freeze.log`) after the book-side check (§9), as
  listed in `data/live/watch.json`. Nothing OPEN, REJECTED or DEFERRED.
- Sports scanned per weekday follow the call plan in `TODO.md`. A sport not scanned on a day is not
  bet that day; that is a quota decision taken before the test, not a selection on results.
- **One bet per game**: the board's top pick for the game. Never both sides, never two books.
- Only rows with status `ok` (not stale, suspect, uncheckable, broken, pin_suspect, swap, period
  mix, mapping-flagged). Rows marked `check_sides` / `check_period` only after the side or period
  has been confirmed on the slip.
- Kickoff at least 15 min after the bet.
- **EV floor on the SLIP price: 3 %** (decided by the owner 14 Nov 2026; 4 % was the alternative). EV on the slip =
  `odds_slip × min(q_mult, q_pow) − 1` with Pinnacle q at bet time. If the slip price is lower than
  the feed price, recompute; below the floor, no bet, logged as a skip (`below_floor`).

  Basis (confirmation data, 22 cells, one best row per game, bands by multiplicative EV — slightly
  above the board's min-of-both EV, so the board's 3 % is a little stricter): CLV rises with
  EV in every band and is positive under both de-vigs from 2 % up — at h1, EV 3–4 %: CLV
  +3.9 / +4.6 % (n 80); 4–6 %: +4.2 / +4.5 % (n 84). A 4 % floor drops ~27 % of the bets for
  higher CLV per bet. Caveat: CLV and EV share the Pinnacle de-vig, so this shows the price holds to
  the close, not that the de-vig is right.

## 3. Stake and bankroll

- Bankroll: **5,000 DKK**, set aside for the test. Part of it sits at the books (2,613 DKK on 14 Nov, entered in the board's
  Bankroll section), the rest in reserve for deposits; the catastrophic stop (§5.4) counts both.
  A book with no balance (nordicbet on 14 Nov) is funded when its first recommendation appears; until then a
  recommendation there that cannot be taken is logged as a skip (`other`, note "not funded").
- Stake: **flat 100 DKK** per bet. Check against open exposure: the confirmation data imply
  roughly 10–15 takeable bets a day, nearly all settled the same day, so open bets stay far below
  ~50. Stake confirmed.
- A book's maximum stake below 100 DKK: bet the maximum, record it in `max_stake_seen`.
- No Kelly during the test.

## 4. Logging (append-only, `data/live/`)

- Every bet through the page at the moment it is placed: time, book, slip odds, stake, Pinnacle q
  at bet time, EV on feed and slip, max stake shown if visible.
- Every recommendation NOT taken (price gone, below floor, stake refused, market closed, wrong side
  or period), with the reason, logged before kickoff.
- Every recommendation the scanner notified is in `notified.csv` whether bet or not.
- Close and CLV written by `live.py` after kickoff; settlement entered after the result.
- A correction is a new row. Nothing is edited or deleted.
- A slip screenshot for every bet is kept locally (not published).

## 5. Stop rules

Per cell (book × market type being bet):

1. **CLV.** After 40 bets in the cell: mean CLV ≤ 0 under EITHER de-vig → stop betting the cell.
2. **Takeability.** After 20 attempts: more than half the recommended prices were gone or below
   the floor on the slip → stop the cell.

Per book:

3. **Limits.** The book caps the stake below 50 DKK on a market → stop that book for that market;
   record the date and the number of bets it took. This is a measurement, not a failure.

Whole test:

4. **Catastrophic stop.** Bankroll at 2,500 DKK (−50 %) → pause and review before any further bet.
5. **Feed errors.** The same feed problem seen twice (a market mismatched, prices summing below
   100 %, Pinnacle disagreeing with every book) → pause until it is understood.
6. **Checkpoint at 100 bets.** Review CLV, takeability and limits. Nothing changes at the
   checkpoint unless the reason has nothing to do with the results (rule 9).
7. **End.** At 300 bets or 8 weeks, whichever comes first. Then decide per cell on ¼ Kelly,
   using CLV on placed bets.

## 6. Analysis (fixed now, run at the end)

- **Primary question:** is mean CLV on placed bets > 0, under BOTH de-vigs?
- **Test:** one-sided t-test of mean CLV > 0 at 5 %, separately under multiplicative and power
  de-vig; the result is "edge shown" only if both pass. One bet per game, so the game is the unit;
  a robustness line reports a sign test.
- **Minimum n: 60 placed bets with a close.** Below that the result is reported as inconclusive,
  never as "no edge". Basis: CLV SD per bet ~6–7 % in the confirmation data, so 60 bets give ~80 %
  power at a true CLV of 2 %.
- **Reported alongside, not judged:** CLV per cell and per book (all cells shown, including the
  stopped ones), profit and ROI, takeability (bets / recommendations), bets until each limit.
- Bets without a Pinnacle close are reported as a count and excluded from CLV, never imputed.

## 7. Public log

- `publish.py` (launchd, every 5 min) publishes `bets.csv`, `skips.csv`, `notified.csv`,
  `closes.csv` and `settlements.csv` (whitelisted columns) to
  https://github.com/lukkeodds/livetest-log, with a sha256 manifest per change stamped by
  OpenTimestamps. A bet counts as preregistered only if its stamp predates kickoff.
- This protocol, once frozen, is published there before the first bet.

## 8. Fixed for the whole test

Stake, EV floor, the set of confirmed cells, the call plan, these stop rules and the analysis in §6.
A cell confirmed later (DEFERRED or rolling confirmation of OPEN cells) may be ADDED only by a dated
amendment written before its first bet; nothing is removed except by the stop rules above or the
book-side check before the start.

## 9. Amendment 1 — book-side check (14 Nov 2026, before the freeze and before any bet)

The owner opened the slips for the board of 14 Nov 2026 (prices fetched 11:00–11:35Z, preview of the 22
cells, 5 billable calls). Checked: market present at the line, slip price against the feed, and the
market's definition against Pinnacle's. No result of any kind was seen; nothing was bet.

| cell | game(s) checked | finding | decision |
|---|---|---|---|
| handball totals, bwin | Karlskrona–Ystads, Redbergslid–Kungälv, Wisla Plock–Piotrkowianin | line and side as in the feed, price at or above take-at | kept |
| volleyball spreads, bet365 | Al Arabi–Al Shamal −2.5 | set handicap, as Pinnacle's; slip 1.61 against feed 1.58 | kept |
| ice-hockey 1x2, bet365 (SHL) | SHL, 14 Nov | draw is regulation time, as Pinnacle's 1x2; draw 4.25 | kept |
| ice-hockey totals, bwin | — | half-goal lines (4.5, 5.5), regulation time | kept |
| soccer spreads, betinia | — | quarter lines offered (−0.25, +0.75): Asian handicap, as Pinnacle's | kept |
| **ice-hockey totals, nordicbet** | Vimmerby–Nybro, Jukurit–KooKoo, Under 5 | **slip settles including overtime and shootout**; the feed and Pinnacle's market are regulation time | **removed** |

Reason for the removal: a total including overtime is a different product from a regulation-time total
(Under is worth less when more time can be played), so the cell's confirmed CLV against Pinnacle's
regulation-time close measures the definition gap, not a mispricing. The line-mapping check had
flagged the cell (`mapping`) before the slips were opened. The decision rests on the market definition,
not on any CLV, profit or result.

Not checked on a slip: the other 15 cells. Every bet is checked on the slip anyway (§2: side, period,
price); a definition mismatch found there is logged as a skip and recorded in a dated amendment that
applies forward.

The 22-cell figures quoted in §2 (EV-floor basis) are left as written; they were the basis for the floor
and dropping one cell does not change the decision.


## 10. Amendment 2 — recommendations start at the EV floor (14 Nov 2026, 12:15Z)

Written during the first afternoon of the test, after four bets had been placed (all with slip EV ≥ 3 %)
and before any of them had a close or a result. It applies forward only.

**Defect.** The board (`live.py` `pick()`) marked the best watched row per game as a recommendation from
EV 2 % (`EV_SHOW`), not from the 3 % floor of §2. A recommendation below the floor in the feed must then be
logged as a `below_floor` skip (§4) and counts against takeability (§5.2), which is meant to measure whether
a price at or above the floor is still on the slip — not whether the board recommended something the floor
already excluded. The code did not match §2; nothing about any result was seen or used.

**Change.** From 12:15Z on 14 Nov 2026 a row is recommended (and notified) only if its feed EV is ≥ 3 %
(`TAKE_EV`). Rows from 2 % to 3 % are still shown, are not bet and are not logged. The slip floor of §2 is
unchanged.

**Already logged.** Two notifications before the change had feed EV below 3 %
(`notified.csv`, 14 Nov 12:01:09Z and 12:10:44Z; 2.27 % and 2.17 %). They stay in the log (append-only) and
are excluded from takeability (§5.2) and from the recommendation count in §6, as "below the floor at
notification". None was bet.

## 11. Amendment 3 — recommendations not seen (14 Nov 2026)

Written on the first evening of the test, after 20 bets and 2 logged skips. It concerns a recommendation
nobody looked at, not any close or result, and applies forward and to the rows already in `notified.csv`.

**Gap.** §4 asks every recommendation not taken to be logged with a reason before kickoff. When the owner is
away, a recommendation can appear and disappear without anyone opening the slip. Logging it as `price_gone`
or `other` would claim an attempt that was never made, and would push takeability (§5.2) down for a reason
that has nothing to do with the books.

**Rule.** A notified recommendation (notified at or after the freeze, 11:56:09Z 14 Nov, with feed EV ≥ 3 %)
whose game has no bet and no logged skip, and which is no longer on the board, is **not seen**. It is
derived from the logs, not entered. Not seen is not an attempt: it is excluded from takeability (§5.2,
the denominator of §6's bets / recommendations) and reported as its own count, per book. It is not a reason
to stop or change a cell.

## 12. Amendment 4 — keep the prices behind each recommendation (16 Nov 2026)

Written after 43 bets, 41 settled and 34 closes had been seen on the board. Logistics only: it changes what
`live.py` stores, not what is recommended, bet, closed or analysed. On the owner's decision (16 Nov).

**Gap.** The scanner keeps no response body. What the board saw at a recommendation survives only as the
row in `notified.csv` (odds, EV, `prices_as_of`). The free `historical-odds` path can be fetched again, but
it expires (cycling: ~7 months; other sports not measured) and a later fetch cannot show that the feed was
not revised after the fact.

**Change.** From the commit of this amendment, `live.py scan`:

1. after each fetch of a game with at least one notified recommendation (now or earlier), writes the merged
   Pinnacle + book-group body to `data/live/raw/{fixtureId}.json.gz`, overwritten on every refetch until the
   game leaves the window — so the last file is the last fetch before the bet decision closed;
2. writes the Pinnacle body of every close fetch (`close_bets`) once, to `data/live/raw/{fixtureId}_close.json.gz`.

Bodies are trimmed in time only (entries from 8 h before kickoff plus the last one before, as `screen.trim8`);
no market is dropped. The files are local, not published and not in git; they are evidence to re-derive a
logged EV or close, never an input to a decision. Games notified before this amendment have no file.

## 13. Amendment 5 — cells confirmed after the start (17 Nov 2026)

Written after 44 bets on the board had been seen. On the owner's decision (17 Nov), after the slip check below. §8 allows a later-confirmed cell to be added by a dated
amendment written before its first bet. The source is the frozen verdict of 17 Nov 2026
(`data/confirm/verdict_2026-11-17.txt`, hash in `docs/freeze.log` 06:49:48Z, 0 DEFERRED). The verdict was
taken under the screen rule and its addenda 10–12, without reference to the board's bets or results.

**Cells.** Eight cells are CONFIRMED on 17 Nov that were not CONFIRMED on 14 Nov:

| # | cell | route | slip check | decision |
|---|---|---|---|---|
| 1 | soccer · betinia.dk · teamtotals-team2 · fulltime | DEFERRED → CONFIRMED | 17 Nov (owner): away team total, 90 min, no extra time | added |
| 2 | handball · betano.dk · spreads · fulltime | OPEN → rolling → CONFIRMED | 17 Nov: Nexe–Elverum −1.5 as feed; Buzau +8.5 1.75 / Flensburg −8.5 1.95, as feed. betano rules (betano.dk/artikel/bettingregler, read 17 Nov) 3.7.1: handball settles on regular time, excluding overtime | added |
| 3 | ice-hockey · betano.dk · totals · fulltime | OPEN → rolling → CONFIRMED | 17 Nov (owner): totals offered both with and without overtime, side by side | added |
| 4 | ice-hockey · betinia.dk · totals · fulltime | OPEN → rolling → CONFIRMED | 17 Nov: the two Czech U20 games in the feed not found on the site; totals offered both with and without overtime, side by side (owner) | added |
| 5 | ice-hockey · unibet.dk · totals · fulltime | OPEN → rolling → CONFIRMED | 17 Nov (owner): totals offered both with and without overtime, side by side | added |
| 6 | tennis · betano.dk · spreads-games · result | OPEN → rolling → CONFIRMED | 17 Nov: ±2.5 games ("partier"), as feed; betano rules (betano.dk/artikel/bettingregler, read 17 Nov) 3.3.2: a match started and not completed voids all unsettled bets | added |
| 7 | tennis · betano.dk · totals-games · result | OPEN → rolling → CONFIRMED | betano rules (betano.dk/artikel/bettingregler, read 17 Nov) 3.3.2 as #6 | added |
| 8 | basketball · nordicbet · totals · result | DEFERRED → CONFIRMED (15 Nov) | 17 Nov: total includes overtime, as Pinnacle's | added (nordicbet account funded 17 Nov) |

All are pooled ("ALL <sport>") cells and enter `watch.json` with `competition: "*"`, and with
`source: "confirm verdict 2026-11-17: CONFIRMED (amendment 5)"`.

**Slip check (as §9).** Before the freeze the owner opened at least one slip per cell. Three things are
checked:

- the market is present at the line;
- the slip price agrees with the feed;
- the market definition matches Pinnacle's.

A definition mismatch removes the cell; the reason is the definition, never a CLV, profit or result.
Points to check:

- **Ice-hockey totals (3–5): regulation time or including overtime/shootout.** This is the mismatch that
  removed nordicbet ice-hockey totals in §9. The screen's line check does not catch it: none of the eight
  cells is flagged at book|market level, and nor was nordicbet ice-hockey totals.
- **Soccer team totals (1):** the side is team2 = away, and the line applies to the away team alone in
  90 minutes.
- **Tennis games markets (6, 7):** the market counts games, not sets. Also check what a retirement does to
  the bet (void, or settled on the result).
- **Handball spreads (2):** regular time only, and full lines versus half lines.
- **Basketball totals, nordicbet (8):** including overtime, as Pinnacle's basketball totals. This is the
  book whose ice-hockey totals settle differently, so check the rule on the slip, not by analogy.

**Ice-hockey totals (3–5), how a bet is taken.** All three books list a regulation-time total and an
including-overtime total next to each other. The cells are confirmed on the feed's `totals|fulltime`, which
is regulation time as Pinnacle's. A bet in these cells is placed only on the regulation-time total (the
slip must say so); a slip that offers only the overtime total is a skip, logged as such. Checked 17 Nov 2026: on Liberec–Köln and Tappara–Mannheim (Champions Hockey
League, total 5.5) the feed's prices for all three books matched the slip's regulation-time market, not the
overtime one (`data/live/slipcheck_hockey_2026-11-17.md`). On one AHL game the feed carried identical prices
under both periods, for the books and Pinnacle alike, so the feed cannot tell the two apart there. A bet in
these cells is therefore taken only when the slip's regulation-time price is at or above take-at, as
for every bet (§2).

**What does not change.** Stake, EV floor, take-at rule, stop rules, call plan and the analysis in §6 apply
to the added cells exactly as to the original 21. The added cells' bets count in the totals from their first
bet. They are also reported as a separate line (by `source`), so the original 21-cell test can still be read
on its own. The verdict's CLV for these cells is not used to set any stake or floor.

**Not added by this amendment:**

- the 28 ROLLING cells, which may come later, each by its own amendment once its rolling verdict is CONFIRMED;
- the 2 OPEN cells: basketball bet365 totals is final OPEN, and soccer bwin spreads becomes ROLLING under
  addendum 11.1;
- nordicbet ice-hockey totals, which was removed in §9 and stays out.

## 14. Amendment 6 — the scanner pauses while the confirmation fetch holds the key (17 Nov 2026)

Written after 44 bets on the board had been seen. Logistics only: it changes when `live.py scan` calls the
API, not what is recommended, bet, closed or analysed. On the owner's decision (17 Nov).

**Gap.** Only one process may call oddspapi at a time. `confirm.py nightly` (launchd 00:05, rolling confirmation,
screen rule addendum 11) takes an exclusive lock on `data/confirm/.fetch.lock` (addendum 12). The scanner did
not look at it, so a board left running past midnight shared the key with the nightly job.

**Change.** From the commit of this amendment:

1. `live.py scan` tests that lock before its fixture lists, before each game's fetch and before the close
   fetches. While the lock is held it makes no call of any kind and waits, testing once a minute. It writes one
   `PAUSE` line and one `RESUME` line to `data/live/scan.log`. The test takes the lock and releases it at
   once; the scanner never keeps it.
2. `confirm.py fetch`/`nightly` retry the lock for 10 s before exiting. Without the retry, a scanner test that
   coincides with the job's start could make the nightly job exit for the night. A real second fetch still
   holds the lock for hours, so the job still refuses then.

A game whose prices would have been fetched during the pause is fetched after it, if it is still in the window.
Recommendations during the nightly hours (CPH 00–about 03) can therefore be missed; they are not logged as
"not seen" (amendment 3) because they were never shown.

Tested in a sandbox with a lock held by another process: the scanner paused and resumed. Under constant
probing the confirm-style start took the lock 200 of 200 times; it took it 198 of 200 times without the
retry. Against a real holder it gave up after 10 s.

## 15. Amendment 7 — the live test moves to a server (17 Nov 2026)

Written after 44 bets on the board had been seen. Logistics only: it changes where and how the scanner,
the board, the nightly confirmation job and the public log run, and how a recommendation reaches the owner.
It does not change what is recommended, bet, closed or analysed. On the owner's decision (17 Nov).

**Why.** Until now everything ran on the owner's Mac, started by hand. It stopped when the lid was closed
(16 Nov: one morning lost to sleep), and a recommendation was seen only on that Mac.

**Change.** From the cutover recorded in `docs/freeze.log` (`cutover` line):

1. **Where.** A VPS (OVHcloud VPS-1, Ubuntu 24.04, EU). It is reachable only over the owner's Tailscale
   network: no port is open to the internet, and SSH accepts keys only. The Mac no longer calls oddspapi.
   Its launchd jobs are disabled (plists renamed `.disabled`). The key and every live-test file move to the
   server, which from then on is their only writer.
2. **How.** systemd units, in `deploy/systemd/`:
   - `oddspapi-scan` runs `board.py`, which starts `live.py scan` with the day's sports from the call plan
     (Tue–Fri five sports, Sat five, Sun six, Monday none), restarted at 00:01 Europe/Copenhagen;
   - `oddspapi-board` runs `server.py` on the Tailscale address only;
   - `oddspapi-nightly` runs `confirm.py nightly` at 00:05 Europe/Copenhagen (screen rule addendum 11);
   - `oddspapi-publish` runs `publish.py` every 5 minutes.
   A unit that fails sends a Pushover alert, at most one per hour per unit.
3. **Notification.** `live.py notify()` sends a new recommendation to the owner's phone through Pushover
   (`push.py`) instead of a macOS notification. Title and text are unchanged, and the row in `notified.csv`
   is written as before. If a push fails, it is printed to the scan log and does not stop the scanner. Such a
   recommendation is "not seen" in the sense of amendment 3 only if no bet or skip follows.
4. **Board address.** `server.py` binds to the address in `BOARD_BIND`. The default stays 127.0.0.1; on the
   server it is the Tailscale address.

**Checks fired before the server runs unattended** (`test_checks.py`, artificial positives, no API calls):
`stale`, `broken`, `pin_suspect`, `mapping`, `uncheckable` and `suspect` rows each get their status, and a
`suspect` row is never recommended. The double-bet guard refuses a second bet on the same game (HTTP 409). The
fetch-lock pause sees the nightly job's lock and continues once it is released. A push without keys returns
False without raising. `test_swap.py` and `test_period.py` (sides, periods, one bet per game) pass on the
server too. The Pushover delivery path and the systemd failure alert are tested on the server after the keys
are in place; their result is added to `docs/freeze.log` as a `cutover` line.

**Shared key.** The cycling project (`cycling-model`) calls oddspapi with the same key. Its scripts must not
run on the Mac while the server runs; they run on the server, or not at all.

## 16. Amendment 8 — sport in the notification (17 Nov 2026)

Written after 44 bets on the board had been seen. Logistics only, on the owner's request: the push text
starts with the sport in Danish (e.g. "Ishockey · Over 5.5 (reg. time) @ 1.91 …"). Title, recommendation,
`notified.csv` and everything else are unchanged.
