# Case Study 2 — Reasoning Guide Index

Conceptual roadmap for the Viador dynamic-pricing case study. Start here, then
read the per-question deep dives. Everything is **analysis and reasoning** — the
two fixed dependencies (the `load_bookings()` loader and the NB3 cancellation
classifier `pipe`) are used as-is; everything else you design and justify.

---

## Read in this order

| # | Document | What it gives you |
|---|---|---|
| 0 | *(this file)* | Orientation, the fixed dependencies, the cross-cutting rules |
| 1 | [`Q1_advance_sale_policies_DEEP.md`](./Q1_advance_sale_policies_DEEP.md) | Segmentation, early/late ADR gap, advance-limit allocation (two routes) |
| 2 | [`Q2_overbooking_parameters_DEEP.md`](./Q2_overbooking_parameters_DEEP.md) | Buffer `n*=c/(1−p̄)`, Poisson-binomial walk risk, cost economics |
| 3 | [`Q3_seasonal_rules_DEEP.md`](./Q3_seasonal_rules_DEEP.md) | Season bands from data, how Q1/Q2 rules shift by season |
| 4 | [`Q4_live_booking_control_DEEP.md`](./Q4_live_booking_control_DEEP.md) | Strip→stream replay, state-dependent protection, `q_eff` integration |

> The original chat response holds the **Phase 1 notebook summaries** and the
> compact Phase 3 guides. These deep-dive files extend Phase 3 with formulas,
> decision conditions, pseudocode sketches, and the subtle traps per question.

---

## The source notebooks

| File | Role | Feeds |
|---|---|---|
| `EDA_marimo_notebook.py` | EDA, canonical-booking bridge, early/late cut tool, booking curves | Q1, Q3 |
| `dynamic_control_marimo_notebook.py` | Booking-strip extractor, static simulator, λ̂, S-index | Q4 baseline |
| `prediction_models_marimo_notebook.py` | Cancellation classifier, calibration, `q_eff`, buffer calculator | Q2, Q4 advanced |
| `assignment.md` | The four case questions | — |

Intended data flow:

```
NB1 (EDA)         ──►  Q1 segmentation + Q3 seasonality
NB2 (control)     ──►  Q4 baseline (strip → simulator → λ̂ → S-index)
NB3 (prediction)  ──►  Q2 buffer (P(cancel) → n*) + Q4 advanced (q_eff in S-index)
```

---

## The two fixed dependencies (use exactly as written)

1. **Data loader** — `load_bookings()` (NB1 line 100; duplicated in NB2/NB3).
   Builds `arrival_date`, `booking_date = arrival_date − lead_time`,
   `lead_weeks`, `adr_per_adult`. Do not propose an alternative loader.
2. **Cancellation classifier** — the **§6 `pipe`** in NB3 (lines 181–244): 10k-row
   sample, seed 42, 12 numeric + 8 categorical leakage-safe features,
   `ColumnTransformer` → `RandomForestClassifier`, scored via
   `pipe.predict_proba(df[feat_cols])[:,1]`. Do not modify its logic, features,
   or training.

> ⚠️ The NB3 **playground classifier** (line 373, 8k rows, seed 7, toggleable
> model) is a teaching toy — *not* the fixed dependency. Use the §6 `pipe`
> downstream.

---

## Rules that bind every question

- **Realised bookings only** for pricing/segmentation — filter `is_canceled == 0`
  before any ADR or volume reasoning (Q1, Q3). Cancellation *risk* is Q2's
  subject, not a pricing input.
- **Two-hotel lens** — answer for City, for Resort, then **compare**. Never pool;
  the City/Resort gap is dominated by a level shift, and seasonal shape differs.
- **Canonical-booking homogenisation** before reading ADR signals (room/meal/party
  fixed; default A/BB/1, ADR clipped `(0, 250)`).
- **Capacity is never in the data** — every advance-limit / buffer / control
  number rests on a stated capacity assumption.
- **Calibration > AUC** wherever a probability is used as a *number* (the
  overbooking buffer, `q_eff`) — cite the Brier score.
- **Robust, not optimal** — prefer flat plateaus and rules that survive noise over
  a single hindsight optimum (Q4 especially).

---

## Open flags carried in the deep dives

- `bookingLimits.py` / `demand_management.py` are **referenced but not in this
  folder** (linked week-1 artefacts). Q1 reconstructs the Littlewood logic
  conceptually — you don't need the files.
- **Cancellation timing is not a clean timestamp** — `is_canceled` is a flag, not
  a "cancelled-on" date. Q4's room-release mechanism needs an explicit timing
  convention (options sketched in the Q4 file).
- **No-shows vs cancellations** — the data encodes cancellations; day-of no-shows
  may not be separable. Note this when sizing the Q2 buffer.
