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

