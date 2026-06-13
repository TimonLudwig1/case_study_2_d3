# Q1 — Calibrate Advance Sale Policies · Deep Dive

> **Scope of this document.** A complete analytical roadmap for Q1: the conceptual
> model, the exact data mechanics, the segmentation logic, the allocation
> arithmetic, the per-hotel contrast, and the write-up narrative. It stops short
> of a finished implementation — every code fragment here is a *sketch* meant to
> make the logic unambiguous, not a drop-in solution. The two fixed dependencies
> (the `load_bookings()` loader and the NB3 cancellation classifier) are used
> as-is; everything else you design and justify yourself.

---

## 1. The decision being calibrated

Clara sells two things out of the same room inventory:

- **Advance / discounted rooms** — booked far ahead, cheaper, low risk of going
  unsold but they consume capacity early.
- **Late / full-fare rooms** — booked close to arrival, higher ADR, but uncertain
  in volume.

The advance-sale *limit* is the number of rooms she is willing to sell at the
discounted rate before she closes that bucket and holds the rest for late
high-fare demand. This is the empirical twin of a **Littlewood protection
level**: every room you do *not* release to advance sales is a room *protected*
for late demand. So Q1 is really two coupled numbers per hotel:

```
advance_limit  +  protection_for_late_high_fare  =  capacity
```

Calibrating one fixes the other. The whole question is: **where do you draw that
line, and how do you defend the position?**

### Why "realised bookings only" is load-bearing here

If you calibrate the advance/late price gap on *all* bookings, you contaminate it
with cancellations — and NB1 §7 shows cancellation rises steeply with lead time
(181–365 days: >55%; 365+: >65%). Advance bookings cancel far more than late
ones. Pricing on requests would therefore:

1. Overstate how much advance demand is "real" (much of it evaporates), and
2. Bias the ADR comparison, because the realised price a cancelled advance
   booking would have paid never materialises.

So **filter `is_canceled == 0` before any ADR or volume reasoning in Q1.**
(Cancellation risk is not ignored — it is the subject of Q2. Q1 is about the
price/timing structure of stays that actually happened.)

---

## 2. Homogenising ADR — the canonical-booking abstraction

Raw `adr` is per-room-per-night and conflates a suite-for-four-half-board with a
single-room-bed-and-breakfast. You cannot read a clean "early vs late" price
signal off it. NB1 §9 solves this with the **canonical booking**: a fixed
room/meal/party tuple so that `adr` becomes a comparable per-unit price.

- **Default canonical:** room `A` + meal `BB` + exactly `1 adult` (~12 000 rows,
  the modal combination).
- **Why fix it:** holding room type, board, and party size constant means any
  remaining ADR variation is attributable to *timing and demand*, which is
  exactly the signal Q1 needs.
- **Sensitivity / widening (a decision to justify):** NB1 hints Resort's most
  representative canonical may be `A/HB/2` rather than `A/BB/1`. Run the analysis
  on the default, then re-run on a hotel-specific canonical and check the
  structure survives. Report both — robustness to the canonical definition is a
  credibility point.

**ADR clipping.** The notebooks clip ADR to `between(0, 250)` (and sometimes
`adr_per_adult` to the same). This removes data-entry zeros and a handful of
implausible high outliers that would otherwise drag medians. Keep it; mention it.

Sketch of the canonical filter (mirrors NB1 §9):

```python
canon = bookings[
    (bookings["is_canceled"] == 0)
    & (bookings["reserved_room_type"].isin(rooms_used))   # default ["A"]
    & (bookings["meal"].isin(meals_used))                 # default ["BB"]
    & (bookings["adults"].isin(adults_used))              # default [1]
    & (bookings["adr"].between(0, 250))
    & (bookings["lead_time"] <= 365)
]
```

---

## 3. Defining "meaningful guest segments"

The assignment says identify segments and ask *when they book* and *what they
pay*. There are three candidate segmentation axes in the data; you should lead
with lead time and corroborate with at least one of the others.

### Axis A — Lead time (primary)

This is the axis the advance-sale decision lives on. Operationalise it as the
**early/late cut**: a single lead-time threshold per hotel splitting bookings
into

- **early / advance** = `lead_time >= cut`
- **late / spontaneous** = `lead_time < cut`

NB1 §9's `_hotel_cut_table` (line 998) is exactly this and reports, per side,
**median ADR, mean ADR, and count `n`**. The count matters: a large ADR gap on
five bookings is noise.

### Axis B — Customer type (corroborating, NB1 §6 line 499)

`customer_type` (Transient, Transient-Party, Contract, Group) tells you *who* the
early bookers are. Contract/Group tend to book far ahead at negotiated (lower)
rates; Transient books later at rack. This makes the lead-time gap *actionable* —
you are not discounting anonymously, you are pricing a recognisable segment.

### Axis C — Distribution channel (corroborating, NB1 §5)

TA/TO books earliest and sits mid-band; Direct books later; Corporate pays the
highest ADR/adult at lower volume. Channel is a lever Clara actually controls
(she can open/close advance rates per channel), so it is operationally relevant.

**Recommendation:** segment primarily on lead time (the cut), then *describe* the
early and late buckets by their customer-type and channel mix. That gives you
"the advance bucket is mostly TA/TO + Contract booking 60+ days out at €X; the
late bucket is Transient/Direct at €Y" — a segment story, not just two numbers.

---

## 4. Finding *when* each segment books — the booking-curve geometry

NB1 §8 (line 651) builds, per hotel, two curves over `lead_weeks`:

- **PDF** — share of realised bookings landing in each lead-week bucket (where
  decisions cluster).
- **Reverse-cumulative CDF** — share of the final book *already on the books* W
  weeks before arrival (how much demand is still to come).

The CDF is the one you mine for the advance allocation. Read off:

- **The 50% mark** — the lead time by which half the final realised demand has
  arrived. City's is later (longer lead times overall); Resort's earlier on the
  density but watch the summer subset.
- **The share booked before your chosen cut** — call it `F_early`. This is the
  fraction of realised demand that historically materialises in the advance
  window. It bounds how much advance volume actually exists to sell.

Conceptually:

```
F_early(cut) = (realised bookings with lead_time >= cut) / (all realised bookings)
```

This is a per-hotel, and ideally per-season (Q3), quantity.

---

## 5. Finding *what* each segment pays — the early/late ADR gap

Run the §9 cut logic per hotel and tabulate:

| Group | Median ADR | Mean ADR | n |
|---|---:|---:|---:|
| Late (lead < cut) | … | … | … |
| Early (lead ≥ cut) | … | … | … |

The **defensible discount** is the realised early-vs-late spread:

```
discount_room_nights  ≈  median_ADR_late  −  median_ADR_early
```

Three diagnostics decide whether the segmentation actually bites:

1. **Median separation.** Is `median_ADR_late − median_ADR_early` materially
   positive? NB1 reports this is *weak* for City pooled (~€85 vs €81) and *strong*
   for Resort-in-summer with a long cut (~€100+ late vs ~€60 early). The "late =
   expensive" law is conditional, not universal — say so.
2. **IQR width** (the shaded band in the §9 `c_canon_buckets` chart). A wide IQR
   means there is price dispersion to discriminate on even when medians barely
   move; a narrow band means the cut has little to bite on. **Check IQR, not just
   medians** — NB1 calls this "the other half of the class signal."
3. **Bucket counts.** A "–" in the §9 table means too few canonical records —
   the filter is too narrow, the month set empty, or the cut outside the strip's
   range. Don't build a rule on an empty bucket.

### Choosing the cut per hotel

Sweep the cut (the slider does this from 1–200 days) and pick the threshold that
jointly maximises (a) a meaningful early/late ADR gap and (b) enough volume on
both sides. Expect **different cuts per hotel** — and that difference is itself a
finding:

- **City:** short cut (~30 days default). Corporate/business travel, flatter
  price structure, weak elasticity. The advance/late distinction is muted; a
  near-static rule is defensible.
- **Resort:** longer cut (~60 days default, often pushed to 150+ in summer).
  Leisure/holiday demand books far ahead at real discounts and pays near rack
  late. The advance-sale lever has real teeth here.

> **The trap to name explicitly:** for Resort the clean split only appears once
> you push the cut far enough out that the *late* bucket is essentially the high
> season itself. That is a real effect, but be honest that it is partly a
> seasonal artefact — which is precisely why Q3 re-runs this per season.

---

## 6. From the gap to an allocation number

Two defensible routes. **State which you use and why; ideally compute both and
compare.**

### Route 1 — Demand-share (descriptive, robust)

The advance pool is the realised demand that historically books in the advance
window. Cap advance sales at that share of capacity, minus a protection margin:

```
advance_limit  =  round( F_early(cut) * capacity )  −  protection_margin
```

- `F_early` from the §8 CDF (§4 above).
- `protection_margin` reserves rooms for the late high-fare segment even within
  the early window. Set it from the late segment's realised demand (below).
- **Pros:** purely empirical, no distributional assumption. **Cons:** ignores the
  opportunity cost of displacing a late high-fare booking; it is a volume rule,
  not a revenue-optimal one.

### Route 2 — Protection / Littlewood-flavoured (normative)

Protect enough rooms for late high-fare demand that the *marginal* protected room
is just worth holding. The classic Littlewood condition for a two-class problem:

```
protect rooms for high fare up to the quantile q* of late high-fare demand
such that:   P(late_high_demand > q*)  =  ADR_low / ADR_high
advance_limit  =  capacity  −  q*
```

- `ADR_low / ADR_high` = the realised early vs late canonical medians from §5.
- `P(late_high_demand > q*)` comes from the empirical distribution of realised
  late high-fare bookings per night for that hotel/room type (count realised
  late high-fare arrivals across comparable historical nights, form the survival
  curve, invert).
- **This is the conceptual template referenced as `bookingLimits.py` in NB1
  (line 40).** That file is *not in this folder* — it is a linked week-1
  artefact. The logic above is what it implements; you do not need the file, you
  need the condition.
- **Pros:** revenue-grounded, defends the trade-off directly. **Cons:** needs a
  per-night demand distribution and the two-class abstraction; more assumptions
  to justify.

### Reconciling the two

Route 1 gives you a volume ceiling; Route 2 gives you a revenue-optimal protection
level. If they roughly agree, you have a robust recommendation. If they diverge,
the divergence tells you something: a much smaller advance_limit from Route 2 than
Route 1 means late high-fare demand is valuable enough to protect aggressively
(true for Resort summer); near-agreement means the price gap is too weak to bother
protecting (true for City pooled).

---

## 7. The per-hotel contrast (required by the brief)

Every Q1 conclusion must be stated for City, for Resort, *and then compared.* The
expected shape of the contrast:

| Dimension | City Hotel | Resort Hotel |
|---|---|---|
| Guest mix | Corporate / business, transient | Leisure / holiday, families, groups |
| Lead-time profile | Long lead but flat price | Strong early-discount → late-rack gradient |
| Cut (early/late) | Short (~30 d) | Long (~60–150+ d) |
| ADR gap early vs late | Weak (few €) | Strong in summer (€60 → €100+) |
| Advance-sale lever | Low value — near-static rule fine | High value — real revenue at stake |
| Recommended advance_limit | Generous (little to protect) | Tighter, season-dependent |

The *interpretive* sentence the instructor wants: **"the same logic produces
different rules because the underlying demand elasticity differs — City's price
is timing-insensitive so advance sales cost little; Resort's price is strongly
timing- and season-sensitive so the advance limit is a genuine revenue
control."**

---

## 8. Recommended visualisations (specific)

1. **Per-hotel reverse-cumulative booking curve** (NB1 §8 style) with the chosen
   cut drawn as a vertical line and `F_early` annotated. Shows *how much demand
   is still to come* at the cut.
2. **Median-ADR-by-lead-bucket with shaded IQR**, both hotels overlaid (the §9
   `c_canon_buckets` chart). The single most persuasive figure for "is there a
   class signal."
3. **Early-vs-late ADR bar/table per hotel** (median + mean + n) at the chosen
   cut — the headline number.
4. **Cut-sensitivity curve:** ADR gap (late − early) on the y-axis vs cut on the
   x-axis, per hotel. Shows the reader you didn't cherry-pick a cut and where the
   gap stabilises.
5. **Optional segment-mix bars:** customer_type / channel composition of the
   early vs late bucket — turns two numbers into a segment story.

---

## 9. Decisions you must consciously justify

- The canonical-booking definition, and whether you used a hotel-specific
  canonical (and that the structure survived the change).
- The lead-time cut per hotel and *why they differ*.
- The corroborating segmentation axis (customer_type vs channel) and why.
- Route 1 vs Route 2 (or both) for the allocation, and the capacity you assumed.
- The ADR clip `(0, 250)` and the realised-only filter.
- An explicit statement that the Resort summer signal is partly seasonal (hands
  off cleanly to Q3).

---

## 10. Pitfalls to avoid

- **Pooling the two hotels.** The City/Resort gap is dominated by a level shift
  (all-year canonical medians €83 vs €42); pooling destroys the signal.
- **Pooling all months for Resort.** Hides the high-season regime Clara cares
  about. At minimum flag it; Q3 fixes it properly.
- **Using all bookings instead of realised stays.** Biases both price and volume
  via lead-time-correlated cancellation.
- **Raw ADR without the canonical homogenisation.** You'd be comparing suites to
  singles.
- **Reading the gap off medians alone.** A flat median with a wide IQR still
  carries a class signal; a flat median with a narrow IQR does not.
- **Over-claiming "late = expensive" for City.** The data says it's weak there —
  asserting otherwise is the classic mis-read.
- **Building a rule on an empty/thin bucket** (the "–" cells). Enforce a minimum
  `n` before trusting a median.
- **Forgetting the advance_limit and protection level are the same decision** seen
  from two sides — don't set them independently.
