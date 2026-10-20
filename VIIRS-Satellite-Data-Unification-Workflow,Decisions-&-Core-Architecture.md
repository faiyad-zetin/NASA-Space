# VIIRS Satellite Data Unification — Workflow, Decisions & Core Architecture

Document Purpose: This document synthesizes the strategic decisions, underlying reasoning, and step-by-step workflow established for unifying duplicate fire detections across the three VIIRS-carrying satellites (Suomi-NPP, NOAA-20, NOAA-21). It serves as an actionable reference for implementation.

Scope note: This covers VIIRS-internal unification only. MODIS↔VIIRS harmonization (including confidence calibration) is a separate, in-progress stage that will reuse this same architecture.

## 1. Executive Summary & Key Decisions

| Decision Point | Final Choice |
| --- | --- |
| Matching radius (VIIRS↔VIIRS) | 750 meters |
| Matching strategy | Two-iteration anchor approach (not a single fixed anchor — see §2.2) |
| Iteration 1 | Anchor = NOAA-20. Window = 65 min. Merges any detection touching NOAA-20 (NOAA-20+SNPP, NOAA-20+NOAA-21, or all three). |
| Iteration 2 | Anchor = NOAA-21. Window = 30 min (25 min actual gap + 5 min bias). Runs only on leftovers from iteration 1 — catches SNPP+NOAA-21 pairs that NOAA-20 missed. |
| Satellite availability by era | Suomi-NPP only: Oct 2011–Nov 2017 (nothing to merge, out of scope). SNPP+NOAA-20: Nov 2017–Nov 2022. All three: Nov 2022 onward. |
| Time math | Must use proper datetime/UTC arithmetic — not raw clock-string subtraction (midnight rollover bug, see §2.6) |
| Fields requiring exact/near-match (not merged) | latitude, longitude, acq_date, acq_time |
| Fields kept as multiple values (not collapsed) | satellite, instrument, confidence |
| Fields reduced to one value | frp, brightness, bright_t31 — via highest-confidence/FRP satellite wins, not averaging |
| Fields dropped post-matching | scan, track (sensor geometry, not fire data — used only during matching) |
| Tiebreak rule when satellites disagree | Take the full record (coordinates, time, FRP, brightness) from whichever matched satellite reported the highest confidence/FRP — not whichever satellite detected it first |
| Single-satellite detections (after both iterations) | Passed through unmodified — no merge logic applied |
| daynight mismatches at day/night boundary | Acknowledged as possible, estimated as <0.1% of cases — not engineered for in MVP |

## 2. Strategic Reasoning ("The Why")

2.1 — Why unify VIIRS satellites at all? Suomi-NPP, NOAA-20, and NOAA-21 all carry VIIRS instruments and can pass over the same afternoon landscape within a roughly 25–60 minute window of each other (when all three are operational). This means a single real-world fire can appear as 2–3 separate detection rows in the raw combined dataset. Left unmerged, this inflates fire counts and corrupts any downstream MODIS↔VIIRS comparison.

2.2 — Why a two-iteration design instead of one fixed anchor? An earlier design used NOAA-21 as a single fixed anchor (drawing the matching circle around every NOAA-21 detection, checking SNPP/NOAA-20 against it). This was rejected for two reasons:

- Satellite launch dates make a single anchor impossible across the full dataset. NOAA-21 only launched November 2022. If the project's data spans back to 2011, using NOAA-21 as anchor means zero unification happens for the entire 2011–2022 period — the anchor satellite simply doesn't exist yet.
- Any single fixed anchor has structural blind spots, independent of the launch-date issue: if the anchor satellite misses a fire (clouds, swath gaps, fire not yet hot enough) that the other two satellites did catch, those two detections never get compared to each other at all — they pass through unmerged even though they're obvious duplicates.

The fix: two sequential matching passes with different anchors, each covering what the other structurally cannot:

- Iteration 1 (NOAA-20 anchor) catches every pairing that involves NOAA-20.
- Iteration 2 (NOAA-21 anchor), run only on detections iteration 1 didn't merge, catches SNPP+NOAA-21 pairs that NOAA-20 missed or wasn't present for.

Running iteration 2 only on iteration 1's leftovers is essential — rerunning it against the full dataset would re-match pairs already merged in iteration 1, producing conflicting or duplicate groups.

2.3 — Why 750m radius? Derived by extension from the MODIS↔VIIRS radius logic (MODIS 1km + VIIRS 375m pixel sizes → 1.5km), roughly halved since both pixels being matched here are VIIRS-sized (375m).

⚠️ Verification flag: The specific "750m" figure was sourced via AI-assisted literature search (not a direct NASA FIRMS doc), citing pixel boundary stretching toward swath edges and allowance for fire-front movement during the overpass gap as justification. This should be independently verified against the cited research before being treated as final — currently a plausible-but-unconfirmed figure, not an authoritative constant.

2.4 — Why different time windows per iteration? Iteration 1's window (65 min) reflects the NOAA-20↔Suomi-NPP overpass gap. Iteration 2's window (30 min) reflects the tighter NOAA-21↔Suomi-NPP gap (~25 min + 5 min bias) — using iteration 1's looser window for iteration 2 would risk false matches, since 65 min is too wide a gap for SNPP+NOAA-21 pairs specifically.

2.5 — Why reject simple averaging for merging values? Averaging is appropriate only where an in-between value is physically meaningful (e.g., coordinates). It is not appropriate for confidence (VIIRS uses categorical N/L/H — no mathematical midpoint) or for selecting "the" canonical record (blending fields across satellites could produce a record no sensor actually reported). Hence: take the full record from whichever matched satellite reported the highest confidence/FRP, rather than averaging fields independently.

2.6 — Why raw time subtraction is a bug, not an edge case. Matching fires near midnight (e.g., one satellite logs 23:45, another logs 00:10 the next day) breaks naive abs(time1 - time2) logic — it computes a false ~23-hour gap instead of the true ~25-minute gap. This recurs daily for fires active near overpass times crossing midnight UTC. Must use proper datetime objects, never string/clock arithmetic.

2.7 — Why daynight mismatches were deprioritized. Mismatches only occur for fires detected within minutes of the day/night cutoff at a specific satellite's overpass — a narrow slice of total detection times. Estimated well under 0.1% of matched pairs. Deliberately scoped out of MVP.

## 3. Step-by-Step Workflow

- Ingest raw VIIRS detections from all available satellites for the relevant time period (note: Suomi-NPP only pre-Nov 2017; SNPP+NOAA-20 pre-Nov 2022; all three after).
- Iteration 1 — NOAA-20 anchor:
- For each NOAA-20 detection, draw a 750m radius circle centered on its (lat, lon).
- Check all Suomi-NPP and NOAA-21 detections against that circle.
- A detection counts as a match if it falls within the circle and within 65 minutes of the NOAA-20 detection's time (proper UTC datetime math).
- Merge all matched groups (see step 4 for merge rules).
- Iteration 2 — NOAA-21 anchor, leftovers only:
- Take only detections not merged in iteration 1.
- For each remaining NOAA-21 detection, draw a 750m radius circle centered on its (lat, lon).
- Check remaining Suomi-NPP detections against that circle, within a 30-minute window.
- Merge all matched groups (step 4).
- Merge matched groups (applied in both iterations):
- Select the record from the satellite with the highest confidence/FRP in the group as canonical (its lat/lon/acq_date/acq_time/brightness/bright_t31/frp are kept as-is).
- Retain satellite and instrument values from all matched rows.
- Retain confidence values from all matched rows (needed for MODIS↔VIIRS calibration later).
- Drop scan/track after matching — no further use.
- Whatever remains unmatched after both iterations = genuine single-satellite detections. Pass through unmodified, no merge logic applied.
- Output: one unified VIIRS fire-detection table, each row representing one physical fire event, with multi-satellite confirmation data preserved.

## 4. Essential References & Technical Specifications

**Column definitions:**

- scan, track — pixel size (km) along scan/flight direction; varies due to sensor geometry (bowtie effect toward swath edges). Used only during matching, not retained post-merge.
- bright_t31 — brightness temperature from a secondary thermal band; helps distinguish real fire signal from false positives.
- frp — Fire Radiative Power (megawatts); most direct available measure of fire intensity.
- confidence — MODIS: numeric 0–100%. VIIRS: categorical Low/Nominal/High. Not directly comparable — core harmonization problem for the next stage, unresolved here.
- type — fire source classification (0 = vegetation fire, 1 = volcano, 2 = static/industrial, 3 = offshore). Confirmed both sensors can report all four types.

**Satellite launch & availability:**

- Suomi-NPP: launched Oct 28, 2011
- NOAA-20 (JPSS-1): launched Nov 18, 2017
- NOAA-21 (JPSS-2): launched Nov 10, 2022

**Satellite overpass timing (source: orbital pass data, team-collected):**

- Aqua (MODIS) ↔ Suomi-NPP (VIIRS): 0–15 min
- Aqua (MODIS) ↔ NOAA-20/21 (VIIRS): 50–60 min
- Terra (MODIS) ↔ any VIIRS: 3–4 hours — excluded from calibration; flagged as possible future "sustained fire" signal, not used in MVP
- NOAA-20 ↔ Suomi-NPP: ~60 min (iteration 1 window)
- NOAA-21 ↔ Suomi-NPP: ~25 min (iteration 2 window, +5 min bias → 30 min)
- NOAA-20 ↔ NOAA-21 gap: not yet independently verified — currently inferred from the other two deltas, not directly sourced

**Open items carried forward to MODIS↔VIIRS stage (not part of this document's scope):**

- Confidence calibration: converting MODIS % ↔ VIIRS N/L/H (candidate approach: derive empirical bands from matched-pair data).
- MODIS↔VIIRS time window: candidate 60–65 min, not yet finalized.
- MODIS↔VIIRS radius: candidate 1.5km, not yet finalized.
- 750m radius figure (§2.3) needs independent source verification.
- NOAA-20 ↔ NOAA-21 overpass gap needs direct verification.
