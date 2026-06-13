# Q3 — Seasonally Update Your Rules · Deep Dive

> **Scope.** A complete analytical roadmap for making the Q1 advance limits and
> Q2 overbooking buffers season-dependent: how to define seasons from the data
> (per hotel), how each rule should move and *why*, and how to present the result
> as a defensible rules table. Builds directly on Q1 and Q2; reuses the same
> fixed dependencies (loader, classifier).

---

## 1. What "seasonally update" actually means

Q1 and Q2 produced *static* numbers — one advance limit and one buffer per hotel,
implicitly averaged over the whole year. But NB1 §6 shows demand and price are
strongly non-stationary (both hotels peak in August; Resort ADR swings €35→€60).
A single number is the **expected value of a curve** (NB2 phrases it this way);
Q3 asks you to replace each static number with a **schedule indexed by season.**

The deliverable is two small tables per hotel:

```
advance_limit(season)        for season in {peak, shoulder, low}
overbooking_buffer(season)   for season in {peak, shoulder, low}
```

plus the justification tying each cell to an observed pattern.

---

## 2. Defining seasons from the data (not the calendar)

**Do not impose a fixed "summer = Jun–Aug" template.** Derive the bands per hotel
from observed demand, because the two hotels have *different* demand calendars.

### 2.1 The signals to aggregate (NB1 §6, realised bookings)

Per hotel, per month (or per ISO week for finer resolution), compute:

- **Realised booking volume** — `groupby([month, hotel]).size()` on
  `is_canceled == 0`. The primary demand-strength signal.
- **Median ADR** (canonical, NB1 §9) — the price-strength signal; confirms whether
  high volume coincides with high willingness-to-pay.
- **Occupancy proxy** — volume relative to assumed capacity, if you want strength
  in occupancy terms rather than raw counts.

### 2.2 Turning the curve into bands

A simple, defensible rule: rank months by realised volume (or a volume×ADR
composite) per hotel and cut into terciles → **peak / shoulder / low**. Or set
thresholds off the annual median:

```
peak     : month_volume  >=  1.15 * median_month_volume
low      : month_volume  <=  0.85 * median_month_volume
shoulder : otherwise
```

Thresholds are a choice — justify them and show sensitivity. Expect:

- **City:** August peak, **strong May–Jun and Sep–Oct shoulders** (business
  travel), relatively flatter overall — more months land in "shoulder."
- **Resort:** sharper single summer peak, deeper low season — more bimodal.

**This difference in seasonal *shape* is itself a Q3 finding**, not just an input.

### 2.3 Arrival month vs booking month

Key seasonality on **arrival month** (when the room is consumed) for capacity
decisions — that is what occupancy and walk risk attach to. Booking-month
seasonality is a different question (when demand *arrives*) and matters for Q4's
arrival-rate forecast, not for the capacity rules here. State which you use.

### 2.4 Multi-year stability check

"Using past years' aggregates" assumes the seasonal pattern repeats. The dataset
spans multiple years — **verify the monthly profile is stable year-over-year**
before trusting it as a forward rule. If a peak appears in one year only, treat it
as noise, not a season.

---

## 3. How the advance limit should move (Q1 → Q3)

The advance limit balances "sell cheap early volume" against "protect rooms for
late high-fare demand." Seasonality shifts that balance through **two channels**:

### 3.1 The early/late ADR gap by season

Re-run Q1's §9 cut analysis **restricted to each season's months** (the §9 month
picker does exactly this). The gap is *conditional on season*:

- **Peak (esp. Resort summer):** the late-vs-early ADR gap is **wide** (NB1: late
  ~€100+, early ~€60). Late high-fare demand is valuable → **protect more, shrink
  the advance limit.**
- **Low season:** the gap **collapses** (City–Resort gap narrows, both flatten).
  There is little late high-fare premium to protect → **enlarge the advance
  limit**; advance discounting is nearly free and guards against spoilage.

### 3.2 The demand level

Independent of the price gap, raw demand strength matters:

- **Peak:** demand will fill the hotel from late bookings alone → you can afford
  to hold rooms back. Tighten advance.
- **Low:** late demand is thin → holding rooms back risks empty rooms (spoilage).
  Loosen advance to lock in volume early.

Both channels point the **same way**: *advance limit is lower in peak, higher in
low season.* Reinforcing, which makes the rule clean — but show both channels.

### 3.3 Mechanically

Recompute, per season, the Q1 inputs:

```
F_early(cut, season)        from the season-restricted booking curve
ADR_gap(cut, season)        from the season-restricted §9 cut table
advance_limit(season)       via the Q1 Route-1 or Route-2 logic, per season
```

You may also let the **cut itself** shift by season (Resort's clean split needs a
longer cut in summer). Note that as a second-order refinement.

---

## 4. How the overbooking buffer should move (Q2 → Q3)

Here the two driving forces can **diverge** — this is the subtle part of Q3 and
worth foregrounding.

### 4.1 Re-estimate `p_bar` per season

Cancellation behaviour is seasonal. Recompute the mean predicted cancellation
probability (NB3 `pipe`) on each season's book of business, or read the empirical
cancellation rate by month (NB1 §7 logic, grouped by month). A higher seasonal
`p_bar` mechanically raises `n* = c/(1−p_bar)` and thus the buffer.

### 4.2 The two forces and their possible conflict

```
Force A (cancellation rate):  higher p_bar  →  larger buffer
Force B (refill / walk-cost): in peak, a walked guest is costlier AND
                              easier to refill; in low season, a walk is
                              pure loss and hard to refill.
```

These do **not** always align:

- If peak season has *higher* cancellations → Force A says bigger buffer, and peak
  refill-ability supports it → **bigger peak buffer.**
- But if peak season has *lower* cancellations (committed holiday bookers with
  deposits) → Force A says *smaller* buffer even though demand is strong. Then the
  buffer shrinks in peak despite full demand, because there is little
  cancellation slack to exploit and walking a peak guest is maximally costly.

**You must check which regime each hotel is in empirically** (cancellation rate by
season, per hotel) rather than assuming "peak ⇒ bigger buffer." This is the most
common Q3 mistake.

### 4.3 The walk-tolerance also shifts

Brand cost of a walk is higher in peak (a ruined high-demand holiday stay) — so
even at equal `p_bar`, you might run a *tighter* walk tolerance in peak,
shrinking the buffer. Tie the tolerance to season explicitly.

---

## 5. Assembling the rules table

The deliverable, per hotel:

| Season | Months (derived) | Demand | ADR gap | `p_bar` | Advance limit | Overbooking buffer |
|---|---|---|---|---|---|---|
| Peak | … | high | wide | … | low | (depends on §4.2) |
| Shoulder | … | mid | mid | … | mid | mid |
| Low | … | low | narrow | … | high | small |

Then a **side-by-side City vs Resort** version so the comparison is legible. The
narrative around it should state, for each adjustment, the *observed pattern* that
justifies it — never "we raised the buffer" without "because cancellation rate in
this season is X% vs the annual Y%."

---

## 6. Recommended visualisations (specific)

1. **Monthly realised volume + median ADR, per hotel** (NB1 §6, the canonical
   seasonality figure) with the derived season bands shaded. This anchors the
   whole answer.
2. **Early-vs-late ADR gap by month/season**, per hotel — small multiples or a
   heatmap. Shows *where* protection earns its keep across the year.
3. **Cancellation rate by month, per hotel** — drives the seasonal buffer and
   exposes the Force-A/Force-B regime.
4. **The rules table rendered as a grouped bar:** advance limit and buffer by
   season × hotel, so the reader sees the schedule at a glance.
5. **Year-over-year monthly volume overlay** — the stability check, demonstrating
   the pattern repeats.

---

## 7. Decisions and assumptions you must justify

- The season-band definitions, the thresholds used, and **why they differ between
  the two hotels.**
- Keying on arrival month (vs booking month) and why.
- For each rule adjustment: the specific observed pattern it rests on.
- Which regime (§4.2) each hotel's buffer is in, *shown from the data.*
- That the seasonal pattern is year-over-year stable (the §2.4 check).
- Minimum sample sizes for thin off-season cells (low-count months produce noisy
  medians/rates — guard against it).

---

## 8. Pitfalls to avoid

- **One season definition for both hotels.** Resort is more peaked than City;
  forcing a shared calendar misstates one of them.
- **Assuming "peak ⇒ bigger buffer" reflexively.** The buffer follows the
  *cancellation rate*, which may be *lower* in peak (committed deposit bookings).
  Check it (§4.2).
- **Moving advance limit and buffer in lockstep.** They respond to different
  signals (price gap / demand vs cancellation rate / walk cost) that can diverge.
- **Confusing arrival-month and booking-month seasonality.**
- **Trusting a single year.** A one-year peak is noise until it repeats.
- **Noisy thin cells.** Low-season months with few canonical bookings give
  unstable medians — enforce a minimum `n`.
- **Forgetting the comparison.** The brief wants City vs Resort *contrasted*, not
  two independent schedules.
- **Re-deriving Q1/Q2 from scratch per season** instead of re-running the *same*
  logic on season-restricted data — consistency of method across seasons is part
  of the "consistent logic" Clara asked for.
