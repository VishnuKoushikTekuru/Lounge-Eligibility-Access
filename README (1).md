# British Airways Lounge Eligibility Lookup Model

A Forage / BA Data Science Task 1 project: building a reusable lookup table to estimate
what percentage of passengers on a flight are eligible for each of BA's three Terminal 3
lounge tiers (Concorde Room, First Lounge, Club Lounge), based on high-level flight
groupings rather than specific flight numbers or aircraft types.

## Problem

BA needs to forecast lounge demand for future schedules that don't exist yet, so the
model can't depend on exact flight numbers or aircraft assignments. It needs to work off
structural attributes of a flight (route type, region, time of day) that will still be
meaningful next season, next year, or with a different fleet mix.

## Approach

1. **Explore the schedule data** — checked which fields are stable identifiers (`HAUL`,
   `TIME_OF_DAY`, `ARRIVAL_REGION`) versus noise (`AIRLINE_CD` is constant; `FLIGHT_NO` is
   reused across unrelated routes and dates, so it's not a real route identifier).
2. **Sanity-check the dataset's pre-built `TIER1/2/3_ELIGIBLE_PAX` columns** before trusting
   them. Correlating `TIER1_ELIGIBLE_PAX` against `FIRST_CLASS_SEATS` gives ~0 correlation,
   and short-haul flights (which carry **zero** First Class seats) actually show *higher*
   average Tier 1 eligibility than long-haul flights that do have First cabins. That's
   logically backwards for real premium-cabin data, so these columns were treated as
   noise/filler rather than ground truth.
3. **Build a grouping scheme** using only structural attributes:
   `HAUL` (Short/Long) × `ARRIVAL_REGION` (Europe / North America / Middle East / Asia) ×
   `TIME_OF_DAY` bucketed into Morning vs. Afternoon/Evening → 8 reusable groups.
   A simplified 4-group version (dropping region) was also built for comparison.
4. **Check aircraft mix per group** as supporting evidence (not as a grouping column,
   per the task's instruction to stay fleet-independent) — this showed real cabin
   configuration differences by region (e.g., North America carries a heavier A380/B777
   mix with more First Class seats than Asia).
5. **Build two lookup tables per grouping scheme**: one derived by averaging the
   dataset's predefined tier columns, and one from independent, justified assumptions
   about business vs. leisure passenger mix per group.
6. **Convert percentages into estimated passenger counts** by multiplying each group's
   Tier % against average total seats, giving BA a "typical flight" and "typical day"
   demand estimate per lounge tier.

## Key finding

The predefined `TIER_ELIGIBLE_PAX` columns in the raw dataset are **not usable as ground
truth** — they show almost no variation across genuinely different flight types (short-haul
vs. long-haul, different regions) and fail an obvious physical-logic check (Tier 1 should
be ~0% where there's no First cabin, but isn't). The final lookup table in this repo is
built from independently justified assumptions, anchored to the one verifiable fact in the
data: **short-haul aircraft carry no First Class cabin**, so Tier 1 eligibility is set to 0%
for all short-haul groups.

## Final approach used for submission

**With-region grouping (8 groups), assumption-based percentages.** Region was kept
because it's tied to a real difference in aircraft/cabin mix (see step 4) that a
region-free model would blend away — and that difference is exactly the kind of
signal BA needs to decide where lounge capacity should be prioritized.

## Repo structure

```
├── README.md
├── requirements.txt
├── notebooks/
│   └── Lounge_Eligibility_lookup.ipynb     # full analysis, step by step
├── outputs/
│   └── Lounge_Eligibility_Lookup_Table_-_Task_1_Completed.xlsx   # final submission file
└── data/
    └── README.md                          # notes on the (excluded) source dataset
```

## How to run

```bash
pip install -r requirements.txt
jupyter notebook notebooks/Lounge_Eligibility_lookup.ipynb
```

Place the source file `British Airways Summer Schedule Dataset - Forage Data Science Task 1.xlsx`
in the `data/` folder before running (excluded from this repo — see `data/README.md`).

## Limitations & next steps

- Percentages are estimated from domain reasoning, not fit against a validated ground
  truth (none was available in this dataset).
- Time-of-day buckets are simplified to Morning vs. Afternoon/Evening; a finer bucketing
  (e.g., separating true late-night arrivals) could be tested if BA has real lounge
  headcount data to calibrate against.
- If BA can supply actual historical lounge check-in counts, the ideal next step is
  replacing the assumption-based percentages with ones fit to that real data, using the
  same grouping structure.
