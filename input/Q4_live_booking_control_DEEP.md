# Q4 — Operate on Live Booking Data · Deep Dive

> **Scope.** A complete analytical roadmap for the adaptive control simulation:
> extracting and replaying a booking strip, the baseline static policy, the
> state-dependent protection rule, integrating the fixed cancellation classifier
> via `q_eff`, the room-release mechanism, and evaluation. Reuses NB2's strip
> extractor and simulator and NB3's `pipe` exactly as written; the *control
> policy* is the part you design.

---

## 1. What Q4 is really asking

Q1–Q3 produced **static** rules (averaged over demand realisations). Q4 drops you
into a **single demand realisation unfolding in time** and asks: given the rules,
how do you actually *decide, request by request*, as bookings and cancellations
stream in — updating your protection limit / overbooking threshold on the fly?

The four moves (NB2 §4): track inventory, prioritise high-fare, stop low-fare when
remaining-demand risk is high, allow overbooking when cancellations are likely.
The governing trade-off is **spoilage vs spillage**. A static rule uses a constant
protection number; a *dynamic* rule makes that number **state-dependent.**

---

## 2. Building the live stream (NB2 §3)

### 2.1 Extract the booking strip

For a chosen `stay_date`, hotel, and room type, select every booking whose stay
covers that night and sort by `booking_date`:

```python
nights = bookings["stays_in_weekend_nights"] + bookings["stays_in_week_nights"]
covers = (bookings["arrival_date"] <= stay_date) \
       & (stay_date < bookings["arrival_date"] + pd.to_timedelta(nights, unit="D"))
covers &= bookings["hotel"] == hotel
covers &= bookings["reserved_room_type"] == room_type      # homogeneous pool
strip = bookings.loc[covers].sort_values("booking_date")
```

Restricting to one room type (NB2 does this) keeps capacity and the fare threshold
referring to a *homogeneous inventory pool* — keep it. The default is Resort,
2016-08-13, room A — the night where NB1's cut analysis found the strongest case
for dynamic protection.

### 2.2 Replay as a time-ordered stream

Sorted by `booking_date`, each row is a **decision moment** arriving in real
lead-time order. The strip has many more rows than rooms — that is requests +
cancellations + shadow demand, exactly the stream Clara would see live.

> **Leakage discipline (critical).** At decision step *t* you may use only
> information available at *t*: the booking's own attributes and the history of
> earlier decisions. You may **not** peek at whether *this* booking later cancels,
> nor at future arrivals. The classifier is leakage-safe by construction (NB3 §6
> drops post-booking columns) — your control loop must hold the same discipline.

---

## 3. The baseline policy (NB2 §4b) — what you must beat

Run the static-protection simulator first and record its outcomes. Its logic:

```
for each request in chronological order:
    if fare_class == "high":          # adr >= split
        if rooms > 0:        accept (rooms -= 1; rev += adr)
        else:                reject  # spillage of high fare
    else:                             # low fare
        if rooms > protection: accept
        else:                  reject
```

Outcomes to log: high/low accepted, high/low rejected, **spillage** (high-fare
rejected), **spoilage** (rooms unsold), realised revenue, inventory trajectory.

**Its known flaw (NB2 lines 266–273):** it accepts each booking once and **never
releases a room when a booking later cancels.** Overbooking + release is the
headline improvement Q4 asks for (§6).

The §4a revenue surface shows the static optimum over (split × protection) for one
realisation — useful to locate a robust *plateau*, but it is hindsight-optimal for
that single strip. A dynamic rule should be robust *across* realisations, not tuned
to one.

---

## 4. The state signal — λ̂ and the S-index (NB2 §4.5, §5)

### 4.1 Arrival rate

```
horizon_days = strip["lead_time"].max() + 1
lambda_hat   = len(strip) / horizon_days        # bookings/day
```

Check **overdispersion** (NB2 §4.5): weekly arrival SD vs √mean. Ratio < 1.5 ≈
Poisson (a constant λ̂ is fine); > 2 means bursty (clustering by channel/weekday/
season dominates → a constant rate will mis-time the policy, and you should
seasonally/weekly-scale λ̂ instead). **Run this check before trusting a flat λ.**

### 4.2 The booking-pressure index

```
S(t) = rooms_left(t) / ( days_to_arrival(t) * lambda_hat )
```

- `S > 1` oversupplied → accept almost everything (spoilage is the risk).
- `S ≈ 1` balanced → apply the static rule.
- `S < 1` undersupplied → protect aggressively for high fare.

S is computed on the **natural booking pace** (every request accepted until rooms
run out) — it is a property of *capacity vs observed demand*, **not** of your
admission policy. NB2 stresses: **S informs the policy; the policy does not define
S.** Don't invert that causality.

---

## 5. The adaptive control rule (the design core)

Replace the constant protection level with a **function of S(t)**. NB2 explicitly
warns the hard rule "`S < 1` ⇒ reject low fare" is too coarse, because high-value
demand clusters near arrival. Three progressively better translations — pick and
justify:

### 5.1 Soft multiplier (recommended default)

Scale protection inversely with booking pressure:

```
protection(t) = base_protection * clip(1 / S(t), lo, hi)
accept low-fare request iff rooms_left(t) > protection(t)
```

When S is high (lots of slack) protection relaxes; as S falls below 1 protection
ramps up smoothly. `base_protection` is your Q1 protection level; `lo, hi` bound
the multiplier so it never fully closes or fully opens.

### 5.2 Hysteresis / k-day trigger

Only start protecting once S has stayed below 1 for **k consecutive days** (NB2's
suggestion). Prevents a single noisy dip from prematurely choking low-fare sales.
More robust to the bursty strips flagged in §4.1.

### 5.3 Time-threshold analogue

Use the **first S<1 crossing date** (NB2 marks this with a black rule) as the
moment to switch from "sell low fare" to "protect for high fare" — a dynamic,
data-driven analogue of Q1's fixed protection level. Note the caveat: because
high-fare demand clusters late, protection should often kick in *earlier* than the
crossing suggests.

**Anchor the high/low fare `split` to Q1's per-hotel finding**, not an arbitrary
€100. The split is where the realised early/late ADR gap separates the classes for
*this* hotel.

---

## 6. Integrating the cancellation model (the advanced layer)

This is the NB3 → Q4 bridge and where the marks are.

### 6.1 q_eff — risk-adjusted occupancy (NB3 §8)

Instead of counting raw accepted rooms, count **expected check-ins**:

```python
feat_cols = list(pipe.named_steps["pre"].feature_names_in_)
p_cancel  = pipe.predict_proba(accepted_so_far[feat_cols])[:, 1]
q_eff     = (1 - p_cancel).sum()        # expected guests who actually show
```

The gap between the naive cumulative count and `q_eff` **is your overbooking
headroom** (NB3 §8). Decision rule:

```
accept the next (low-fare) request iff  q_eff + 1  <  capacity + tolerated_walks
```

i.e. compare **`q_eff` against capacity, not the raw room count.** This lets the
policy keep accepting while enough on-the-books bookings are likely to cancel — a
fully dynamic version of the Q2 buffer (NB3 §6 prose, point 3).

### 6.2 The room-release mechanism (fixing the NB2 baseline)

When a booking that you accepted later cancels (its `is_canceled == 1` and you
have reached its cancellation moment in the stream), **return its room** to
inventory. Options (from Q2 §8):

- **Release & re-sell** — `rooms_left += 1`, re-open admission. Maximises
  occupancy.
- **Hold as buffer** — absorb future no-show risk, don't re-sell.
- **Waitlist feed** — give the freed room to an earlier denied high-fare request.

Combined with `q_eff`, releasing is the natural choice: you overbook against
predicted cancellations *and* recover capacity when they realise.

> **Timing subtlety.** The dataset gives `is_canceled` but the *cancellation
> moment* is not a clean timestamp. A defensible convention: treat the
> cancellation as occurring at some point between booking and arrival (e.g. draw
> from the empirical cancel-timing distribution, or place it at arrival as a
> conservative no-show). State your convention — it materially affects when rooms
> free up.

### 6.3 Forecast of remaining demand (optional, NB3 §8 caveat)

The S-index denominator uses a *flat* λ̂. NB3 leaves `D̂_remaining` as an exercise.
A simple, defensible forecast: scale historical bookings-per-remaining-day by a
seasonal (and day-of-week) multiplier from NB1's curves. Use it to make the
denominator forward-looking, so the policy protects earlier when late high-fare
demand is expected to cluster. Mark this as a refinement, not a requirement.

---

## 7. The full adaptive loop (pseudocode sketch)

```
rooms_left = capacity
accepted   = []                      # for q_eff scoring
for request in strip_sorted_by_booking_date:
    t        = request.booking_date
    S        = rooms_left / (days_to_arrival(t) * lambda_hat)
    prot     = base_protection * clip(1/S, lo, hi)        # §5.1
    q_eff    = expected_checkins(accepted)                # §6.1, via pipe
    is_high  = request.adr >= split                       # split from Q1

    if is_high:
        if rooms_left > 0:
            accept(request); rooms_left -= 1; accepted.append(request)
    else:  # low fare
        room_ok      = rooms_left > prot
        overbook_ok  = q_eff + 1 < capacity + tolerated_walks   # §6.1
        if room_ok and overbook_ok:
            accept(request); rooms_left -= 1; accepted.append(request)

    # release step (§6.2): if a previously-accepted booking cancels at t
    for b in accepted_that_cancel_at(t):
        rooms_left += 1; accepted.remove(b)
```

Every branch is a **methodological choice** you must defend; the sketch shows the
*structure*, not a tuned solution.

---

## 8. Evaluation — proving the adaptive rule earns its keep

Compare baseline (§3) vs adaptive (§7) on the **same strip**, logging:

- Realised revenue
- Spillage (high-fare rejected)
- Spoilage (rooms unsold at arrival)
- **Walk-aways** (check-ins > capacity at arrival)

Then — and this is required by the brief — **run both a City night and a Resort
night and compare.** The expected result (NB2 header, lines 41–49):

- **Resort in summer:** strong late/early price gap → dynamic protection
  meaningfully beats static (protect rooms for late high-fare, release only after
  the deep-advance discount window closes).
- **City pooled:** flat price structure (late ~€85 vs early ~€81) → the adaptive
  rule looks almost identical to static. **That near-equivalence is itself a
  finding** — dynamic control is only worth its operational cost where the class
  signal is real. Don't hide the null result; it is the point.

### Robustness, not a single optimum

One strip is one demand realisation. Re-run across several comparable stay nights
(same hotel/season) and report the *distribution* of outcomes. NB2's §4a plateau
message applies: prefer a policy that sits on a **flat, robust region**, not one
tuned to a sharp peak on a single night. "Transparent, robust to noise" (the
brief) beats "optimal on one night."

---

## 9. Recommended visualisations (specific)

1. **Booking strip scatter** (NB2 §3): ADR vs booking_date, coloured by eventual
   outcome (realised / cancelled), arrival line marked. Sets the scene.
2. **Inventory trajectory, baseline vs adaptive overlaid** — rooms remaining vs
   request index, with the dynamic protection level drawn as it moves.
3. **S(t) along the stream** with the S<1 crossing marked (NB2 §5).
4. **Naive cumulative vs `q_eff`** (NB3 §8) — visualises the overbooking headroom
   the release mechanism exploits.
5. **Outcome comparison bars** (revenue / spillage / spoilage / walks) baseline vs
   adaptive, **per hotel** side by side — the answer figure.
6. Optional: §4a revenue surface to show your operating point sits on a robust
   plateau.

---

## 10. Decisions and assumptions you must justify

- The state→protection mapping (soft 1/S multiplier vs k-day hysteresis vs
  crossing threshold) and its bounds.
- Whether/how you integrate `q_eff` and the **room-release mechanism**, including
  the cancellation-timing convention (§6.2).
- The high/low fare `split`, anchored to Q1's per-hotel finding.
- Capacity and tolerated walk rate (consistent with Q2).
- Whether λ̂ is flat or seasonally/weekly scaled (justified by the overdispersion
  check).
- That the chosen night(s) are representative — show peak and off-peak, and
  multiple realisations.

---

## 11. Pitfalls to avoid

- **Look-ahead leakage.** Using a booking's future cancellation or future arrivals
  in the decision at time *t*. Keep the control loop as leakage-safe as the
  classifier.
- **Treating S as the policy.** S is an input that reads structural pressure off
  the strip; the policy is what you build *from* S (NB2 stresses this).
- **Keeping a flat λ on a bursty strip.** Run the overdispersion check first.
- **Omitting the room-release mechanism.** It is the headline improvement Q4 asks
  for and the fix to the baseline's known flaw.
- **Reporting one night as proof.** Demand realisations vary; robustness across
  strips matters more than one optimum.
- **Demonstrating only on Resort.** You must show City too — the *smaller* gain
  there is a finding, not a failure.
- **Tuning to the §4a hindsight optimum.** That is in-sample over-fitting to one
  realisation; prefer the robust plateau.
- **An arbitrary fare split.** Anchor it to Q1's realised early/late ADR
  structure for that specific hotel.
- **Confusing `q_eff` (expected check-ins) with raw room count** — comparing the
  wrong one against capacity defeats the overbooking logic.
