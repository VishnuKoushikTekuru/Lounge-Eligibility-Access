# British Airways Data Science Program

Two projects completed as part of the BA / Forage Data Science job simulation:
a lounge eligibility forecasting model (Task 1) and a predictive model of
customer booking completion (Task 2).

## Repo structure

```
# this file — covers both projects

├── task1-lounge-eligibility/
│   ├── notebooks/Lounge_Eligibility_lookup.ipynb
│   ├── outputs/Lounge_Eligibility_Lookup_Table_-_Task_1_Completed.xlsx
│   └── data/                          # place source dataset here (excluded)
└── task2-booking-prediction/
    ├── notebooks/Getting_Started.ipynb
    ├── outputs/BA_Booking_Prediction_Summary.pptx
    ├── outputs/final_shap_direction.csv
    ├── outputs/origin_country_summary.csv
    └── data/                          # place source dataset here (excluded)
```

## How to run either project

```bash
pip install -r requirements.txt
jupyter notebook task1-lounge-eligibility/notebooks/Lounge_Eligibility_lookup.ipynb
jupyter notebook task2-booking-prediction/notebooks/Getting_Started.ipynb
```

Each project's source dataset is Forage/BA program material and isn't
redistributed here — download it from your Forage task page and place it in
that project's `data/` folder before running its notebook.

---

## Task 1 — Lounge Eligibility Lookup

### Problem

BA needs to forecast lounge demand for future schedules that don't exist yet, so
the model can't depend on exact flight numbers or aircraft assignments. It needs
to work off structural attributes of a flight (route type, region, time of day)
that will still be meaningful next season, next year, or with a different fleet mix.

### Approach

1. **Explored the schedule data** — checked which fields are stable identifiers
   (`HAUL`, `TIME_OF_DAY`, `ARRIVAL_REGION`) versus noise (`AIRLINE_CD` is
   constant; `FLIGHT_NO` is reused across unrelated routes and dates, so it's
   not a real route identifier).
2. **Sanity-checked the dataset's pre-built `TIER1/2/3_ELIGIBLE_PAX` columns**
   before trusting them. Correlating `TIER1_ELIGIBLE_PAX` against
   `FIRST_CLASS_SEATS` gives ~0 correlation, and short-haul flights (which
   carry **zero** First Class seats) actually show *higher* average Tier 1
   eligibility than long-haul flights that do have First cabins — logically
   backwards for real premium-cabin data, so these columns were treated as
   noise/filler rather than ground truth.
3. **Built a grouping scheme** using only structural attributes: `HAUL`
   (Short/Long) × `ARRIVAL_REGION` (Europe / North America / Middle East /
   Asia) × `TIME_OF_DAY` bucketed into Morning vs. Afternoon/Evening → 8
   reusable groups. A simplified 4-group version (dropping region) was also
   built for comparison.
4. **Checked aircraft mix per group** as supporting evidence (not as a
   grouping column, to stay fleet-independent) — this showed real cabin
   configuration differences by region (e.g., North America carries a
   heavier A380/B777 mix with more First Class seats than Asia).
5. **Built two lookup tables per grouping scheme**: one derived by averaging
   the dataset's predefined tier columns, and one from independent, justified
   assumptions about business vs. leisure passenger mix per group.
6. **Converted percentages into estimated passenger counts** by multiplying
   each group's Tier % against average total seats.

### Key finding

The predefined `TIER_ELIGIBLE_PAX` columns in the raw dataset are **not usable
as ground truth** — they show almost no variation across genuinely different
flight types and fail an obvious physical-logic check (Tier 1 should be ~0%
where there's no First cabin, but isn't). The final lookup table is built from
independently justified assumptions, anchored to the one verifiable fact in the
data: **short-haul aircraft carry no First Class cabin**, so Tier 1 eligibility
is set to 0% for all short-haul groups.

### Final approach used for submission

**With-region grouping (8 groups), assumption-based percentages.** Region was
kept because it's tied to a real difference in aircraft/cabin mix that a
region-free model would blend away — and that difference is exactly the kind
of signal BA needs to decide where lounge capacity should be prioritized.

### Limitations & next steps

- Percentages are estimated from domain reasoning, not fit against a validated
  ground truth (none was available in this dataset).
- Time-of-day buckets are simplified to Morning vs. Afternoon/Evening; a finer
  bucketing could be tested if BA has real lounge headcount data to calibrate
  against.
- If BA can supply actual historical lounge check-in counts, the ideal next
  step is replacing the assumption-based percentages with ones fit to that
  real data, using the same grouping structure.

---

## Task 2 — Predictive Modeling of Customer Bookings

### Problem

BA wants to be proactive rather than reactive with customers — using booking
data to predict who is likely to complete a booking, so marketing can be
targeted before the customer even arrives at the airport.

### Approach

1. **Explored** 50,000 bookings (`customer_booking.csv`) — no missing values,
   but a ~15% positive class imbalance and two high-cardinality categoricals
   (`route`: 799 unique, `booking_origin`: 104 unique).
2. **Prepared the data**: frequency-encoded `route` and `booking_origin`
   (avoiding a one-hot blow-up that would dilute the signal across dozens of
   sparse columns), capped extreme outliers in `purchase_lead` and
   `length_of_stay`, and engineered new features (`total_extras`, time-of-day
   buckets, lead-time buckets, weekend flag).
3. **Trained a RandomForestClassifier** (200 trees, `class_weight="balanced"`
   to handle the imbalance), validated with 5-fold stratified cross-validation.
4. **Evaluated** on a held-out test set: ROC-AUC, precision, recall, F1,
   confusion matrix.
5. **Interpreted the model two ways**:
   - Permutation importance (robust to RandomForest's bias toward
     high-cardinality features, unlike the default impurity-based importance)
   - SHAP values, to get both magnitude *and* direction of each feature's effect
6. **Profiled high-likelihood customers** using predicted probabilities, and
   validated the model's real-world lift by comparing the top-decile actual
   booking rate against the overall base rate.

### Key results

| Metric | Value |
|---|---|
| Cross-validated ROC-AUC (5-fold) | 0.755 (std ±0.006 — stable) |
| Held-out test ROC-AUC | 0.762 |
| Recall / Precision | 74% / 28% |
| Top-decile lift | 2.8x (41.8% vs. 15.0% base rate) |

### Key findings

- **Origin market is the strongest single driver**, but the effect is
  country-specific rather than simply volume-based: Australia (17,872
  bookings) converts at just 5%, while Malaysia (7,174 bookings) converts at
  34%.
- **Shorter trips, shorter-haul flights, and wanting optional extras**
  (baggage, preferred seat, in-flight meals) all push toward completion.
- **Booking via Internet (not Mobile)**, closer to the travel date, for a
  round trip with fewer passengers, also increases the likelihood of completion.
- Most time-based engineered features (weekend flag, hour bucket, flight day)
  contributed negligibly — several were statistically indistinguishable from noise.
- **Best-fit customer profile**: a solo or couple traveler on a shorter,
  round-trip, shorter-haul itinerary who wants most optional extras.

### Verdict

The data is viable for **prioritizing marketing spend** by likelihood-to-book
(the model concentrates real bookers into the top decile at nearly 3x the base
rate), but the ~0.76 ROC-AUC ceiling means it should not be used for
high-confidence predictions about any single customer.

### A note on methodology choices

An example answer for this task encoded `booking_origin` as one-hot region
columns and used RandomForest's default impurity-based importance. Both
choices dilute or bias the origin signal — one-hot splits it across many
sparse columns, and impurity-based importance is known to favor
continuous/high-cardinality features. This project deliberately used frequency
encoding (to preserve the full origin signal in one feature) and permutation
importance + SHAP (to correct for that bias), which is why the results differ
substantially from that example.

### Limitations & next steps

- Origin market's country-specific effect (e.g., Australia vs. Malaysia) isn't
  explained by anything in this dataset — likely reflects factors like local
  competing airlines, currency, or booking culture that aren't captured here.
- If BA can supply richer customer-level data (loyalty status, past purchase
  history, marketing engagement), that would likely raise the ROC-AUC ceiling
  beyond what booking metadata alone can achieve.
