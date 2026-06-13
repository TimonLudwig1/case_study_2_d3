# Q2 — Derive Overbooking Parameters · Deep Dive

> **Scope.** A complete analytical roadmap for sizing a robust overbooking buffer
> per hotel: the probability model, the buffer arithmetic, the walk-away risk
> model, the cost/economics framing, and the per-hotel contrast. The NB3
> cancellation classifier (`pipe`) is the **fixed dependency** — used exactly as
> written. Everything downstream of the probabilities is yours to design.

---

## 1. The decision and its two failure modes

Overbooking accepts more reservations than there are rooms, betting that enough
will cancel or no-show that the hotel still fills without overflowing. Two ways it
goes wrong:

- **Under-book (buffer too small)** → empty rooms despite refundable demand
  having existed. This is **spoilage** — silent, opportunity-cost loss.
- **Over-book (buffer too large)** → more guests show up than rooms. Someone gets
  **walked** (relocated/compensated). This is **spillage** in its harshest form —
  a direct cost *plus* brand damage.

The buffer is the single number that trades these off. Q2 asks for it **per
hotel**, robustly, with the trade-off made explicit.

### Where this sits relative to the notebooks

NB2 §4b's simulator is the **no-release baseline**: it accepts each booking once
and *never frees a room when a booking later cancels* (flagged at lines 266–273).
That is precisely the gap overbooking fills — and NB2 explicitly hands the
release-mechanism design to Q2. NB3 §8.5 gives the buffer calculator template.

---

## 2. The cancellation probability — the engine input

### 2.1 Two sources of `p`, used for different things

1. **Empirical base rate** (NB1 §7). Historical cancellation share, per hotel and
   broken out by lead-time bucket and channel. City ~42%, Resort ~28% overall;
   rises steeply with lead time (181–365 d > 55%, 365+ > 65%). Use this to
   **sanity-check** the model and to understand *structure* (which bookings carry
   the risk).
2. **Calibrated model probabilities** (NB3 §6 `pipe`). Per-booking `p̂(cancel)`.
   Use these to **size the buffer**, because they condition on the actual booking
   attributes (lead time, deposit type, channel, segment, …) rather than a single
   pooled average.

### 2.2 Why calibration, not AUC, is the metric that matters

The buffer arithmetic uses `p̂` *as a number* (`n* = c/(1−p̄)`), not as a ranking.
A model can rank perfectly (high AUC) yet be systematically over-confident, which
would mis-size the buffer. The relevant quality metric is therefore the **Brier
score** and the **calibration curve** (NB3 §6, line 300) — "when the model says
30% it should cancel ~30% of the time." Cite the Brier score and the calibration
plot when you argue the probabilities are trustworthy; do *not* lean on AUC alone.

### 2.3 Scoring the book of business

Use the fixed pipeline exactly as NB3 §8.5 does:

```python
feat_cols = list(pipe.named_steps["pre"].feature_names_in_)
p_cancel  = pipe.predict_proba(book_df[feat_cols])[:, 1]
p_bar     = p_cancel.mean()          # drives the buffer size
p_var     = p_cancel.var()           # drives the walk-away spread
```

`pipe` carries its own imputation and one-hot encoding — do **not** preprocess by
hand. `book_df` must be the set of bookings on the books for the night/period you
are sizing (per hotel).

---

## 3. The point buffer — and why it is only the starting point

The expected-value rule (NB3 §6 prose, §8.5):

```
n*      =  c / (1 − p_bar)          # bookings to sell so that E[check-ins] = c
buffer  =  n* − c
```

Worked example from NB3: `c = 100`, `p_bar = 0.25` → `n* = 133`, buffer = 33.

**The critical caveat:** at `n*`, expected check-ins *equal* capacity. Check-ins
are random, centred on `c`, so they exceed `c` roughly **half the time.** Sizing
to `n*` therefore walks guests ~50% of nights. **`n*` is the ceiling you back off
from, not the answer.** Robustness = choosing a buffer whose *walk-away tail* is
acceptable, which requires modelling the distribution, not just the mean.

---

## 4. The walk-away risk model (the robustness layer)

This is where Q2 earns its marks. NB3 §8.5 implements a Poisson-binomial check-in
model; understand it rather than just reading its output.

### 4.1 Distribution of check-ins

Each booking `i` shows up independently with probability `1 − p̂_i`. Check-ins are
a **sum of independent non-identical Bernoullis** (Poisson-binomial):

```
E[check-ins]   =  Σ (1 − p̂_i)
Var[check-ins] =  Σ p̂_i (1 − p̂_i)
```

For a book sized at `n*`, `E[check-ins] = c` by construction. The SD is
approximated in NB3 as:

```
expected_var_per_row = mean( p̂_i (1 − p̂_i) )      # over the strip
walks_sd             = sqrt( n* * expected_var_per_row )
```

### 4.2 Expected walks and tail probability

Walks = `max(check-ins − c, 0)`. With check-ins approximately Normal centred at
`c`:

```
E[walks]            =  walks_sd / sqrt(2*pi)        # half-normal mean
P(walks > tol)      =  P(check-ins > c + tol)
                    =  1 − Φ( tol / walks_sd )
```

where `tol = walk_tolerance_fraction * c` (NB3's `buf_walk_tol`, default 2% of
capacity). The Normal approximation to a Poisson-binomial is good when `n` is
moderate and no single `p̂_i` dominates — note this assumption.

### 4.3 The key structural insight: spread matters, not just the mean

Two hotels with the *same* `p_bar` but different *dispersion* of `p̂` have
different walk risk. `Var[check-ins] = Σ p̂_i(1−p̂_i)` is **maximised when
probabilities sit near 0.5** and shrinks when they are near 0 or 1. So a book full
of "either clearly will or clearly won't cancel" bookings (bimodal `p̂`) is
*safer* to overbook than one full of coin-flip bookings, even at equal mean.
Inspect the **distribution of `p̂` per hotel**, not just its mean — this is the
deeper-than-the-formula point.

---

## 5. Choosing the buffer by the risk constraint

The procedure that turns the calculator into an actual recommendation:

1. Fix capacity `c` per hotel (an assumption you must state — the data has no
   capacity column; NB3 defaults to 80).
2. Set a **tolerated walk rate** consistent with brand economics (see §6).
3. Sweep the buffer from 0 upward. For each candidate, compute `P(walks > tol)`
   via §4.
4. **Pick the largest buffer whose `P(walks > tol)` stays under your risk
   appetite** (e.g. ≤ 5–10%). That is your robust buffer — strictly below the
   naive `n* − c`.
5. Report the buffer, the implied `n*`, the expected walks, and the tail
   probability together. A number without its risk is not an answer.

This produces a **buffer–risk frontier** per hotel; your recommendation is the
point on it that respects the walk tolerance.

---

## 6. The economics — what actually sets the tolerance

The buffer is not a pure statistics problem; it is a **cost-asymmetry** problem.
Frame it explicitly:

```
Expected cost of a too-small buffer  =  P(empty room) * (lost ADR per night)   [spoilage]
Expected cost of a too-large buffer  =  P(walk)       * (walk cost + brand hit) [spillage]
```

The optimal buffer balances the marginal versions of these. The walk cost is not
just the relocation bill — it includes compensation, the lost lifetime value of an
angry guest, and review/brand damage. The brief's instruction — *"if you ever
wonder whether a pricing rule is fair, put yourself in the shoes of a guest"* —
applies directly: a walked leisure family on a long-planned holiday is a worse
outcome than a rebooked corporate traveller who walks-in often.

**This asymmetry is why the two hotels get different tolerances even before you
look at their cancellation rates** (§7).

---

## 7. The per-hotel contrast (required)

The two hotels pull the buffer in *partly opposing* directions — make this
tension explicit rather than reporting two numbers:

| Factor | City Hotel | Resort Hotel | Effect on buffer |
|---|---|---|---|
| Cancellation rate | High (~42%) | Lower (~28%) | City supports a **larger** buffer |
| Guest type | Corporate, repeat, walk-in tolerant | Leisure, long-planned, brand-sensitive | Resort wants a **smaller** buffer |
| Walk cost / brand hit | Lower | Higher | Resort wants a **smaller** buffer |
| Refill ability if a walk slot frees | Easier (short lead, spontaneous demand) | Harder (demand books ahead) | City **larger**, Resort **smaller** |

So: **City — higher cancellation rate *and* lower walk cost both argue for an
aggressive buffer.** **Resort — lower cancellation rate *and* higher brand
sensitivity both argue for a conservative buffer.** The factors happen to align
within each hotel here, which makes the recommendation clean — but say *why* they
align rather than asserting the numbers.

---

## 8. Designing the room-release mechanism (the NB2 hand-off)

NB2 explicitly invites Q2 to fix the simulator's "never release on cancellation"
simplification (lines 266–273). Three design options — pick one and justify:

1. **Release on cancellation** — when a booking cancels, return its room to
   inventory and re-open sales. Simplest; maximises occupancy; the overbooking
   buffer is then a *standing* cushion sized by `p_bar`.
2. **Hold cancelled rooms as buffer** — don't re-sell, treat the freed room as
   absorbing future no-show risk. More conservative; fewer walks; more spoilage.
3. **Waitlist feed** — released rooms feed a waitlist of denied earlier requests.
   Best occupancy/revenue but operationally heaviest.

The buffer interacts with the choice: a *release* policy needs a smaller standing
buffer (you recover capacity dynamically); a *hold* policy needs a larger one.
State the interaction.

---

## 9. Recommended visualisations (specific)

1. **Cancellation rate by lead-time bucket and by channel, per hotel** (NB1 §7
   style) — the base-rate evidence and the structural "where the risk lives"
   story.
2. **Distribution (histogram) of per-booking `p̂` per hotel** — shows whether risk
   is concentrated or diffuse, and underpins the variance/spread argument (§4.3).
3. **Buffer–risk frontier:** x = buffer, y = `P(walks > tolerance)`, one line per
   hotel, with the chosen buffer marked and the tolerance as a horizontal rule.
   This *is* the answer figure.
4. **Calibration plot** (NB3 §6) — justifies trusting the probabilities for a
   numeric decision.
5. **Expected-walks vs buffer** annotated with the `n*` point, to visualise the
   ~50%-walk trap of sizing to the mean.

---

## 10. Decisions and assumptions you must justify

- `p_bar` from the calibrated model vs the raw historical rate (and *why* the
  model — calibration/Brier, conditioning on booking attributes).
- Capacity `c` per hotel (no data column — explicit assumption).
- The tolerated walk rate, anchored in the walk-cost vs spoilage-cost asymmetry,
  per hotel.
- **Independence of cancellations.** The Poisson-binomial model assumes
  independent Bernoullis. Block bookings (a tour operator cancelling 20 rooms at
  once) are *correlated* and fatten the tail — flag this as a limitation and, if
  you can, segment block vs transient.
- **Cancellations vs true no-shows.** `is_canceled` captures cancellations; a
  day-of no-show may not be separately encoded. Note whether your buffer is
  sized against cancellations only and how a no-show would add to it.
- The Normal approximation to the Poisson-binomial (valid when no single `p̂`
  dominates).
- The room-release mechanism (§8) and its interaction with the buffer size.

---

## 11. Pitfalls to avoid

- **Sizing to `n*` and calling it robust.** It walks guests ~50% of the time.
- **Justifying the model with AUC.** Brier/calibration is the relevant metric for
  a buffer.
- **Assuming independent cancellations** without flagging block-booking
  correlation.
- **One buffer for both hotels.** Their cancellation rates *and* brand
  sensitivities differ — the buffer must too.
- **Ignoring spread of `p̂`.** Two books with equal `p_bar` but different
  dispersion have different walk risk.
- **Treating the §8.5 calculator output as the answer.** NB3 says explicitly these
  are not case answers — they are inputs you must wrap in the walk-cost economics.
- **Forgetting Q3.** The buffer should ultimately be season-dependent; size the
  baseline here, but note cancellation behaviour shifts seasonally.
- **Re-selling a released room without bounding total exposure** — a release
  mechanism plus a fixed buffer can compound into more overbooking than intended.
