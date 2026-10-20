# MODIS–VIIRS Harmonization Platform
## Universal Roadmap and Source of Truth

## 1. Project purpose

Build a transparent, sensor-aware platform that makes NASA MODIS and VIIRS active-fire observations easier to compare, analyze, and use. The platform should preserve original observations, document how results are produced, quantify uncertainty where possible, and avoid presenting uncertain matches as confirmed individual fires.

The core scientific challenge is that different sensors can report different observations for the same place and period. Differences can arise from spatial resolution, observation time, scan geometry, detection sensitivity, confidence definitions, clouds, and other conditions. Harmonization should improve comparability without hiding real differences or changes in fire activity.

## 2. Guiding principles

- **Preserve provenance:** Every derived result should be traceable to source observations and processing steps.
- **Keep original records:** Never overwrite or discard the original observations when producing standardized or harmonized data.
- **Be sensor-aware:** Retain sensor, satellite, instrument, acquisition time, quality information, and relevant scan geometry.
- **Separate evidence from assumptions:** Distinguish observed detections, candidate matches, candidate event clusters, corroboration, and independently validated fire probability.
- **Represent uncertainty honestly:** Do not claim perfect accuracy or treat a cross-sensor match as proof that two detections represent the same physical fire.
- **Validate before operational use:** Do not promise alert reliability, calibration quality, or forecasting performance before testing.
- **Prefer simple, reproducible methods first:** Add more complex models or infrastructure only when evidence shows they are needed.
- **Prevent false harmonization:** Do not remove a discontinuity or force sensor agreement unless the correction is scientifically justified.

## 3. Overall architecture

The platform follows one shared data foundation with three interfaces:

1. **Scientists — historical time series and archive:** Explore long-term fire activity, seasonal patterns, trends, and sensor-specific records.
2. **Researchers — query and export:** Filter, compare, analyze, and export observations and harmonized products with their metadata and provenance.
3. **Firefighters and operational users — map and alert feed:** View recent detections and relevant geographic context. Any operational alerting must be separately validated for its intended use and clearly show freshness and uncertainty.

All interfaces should use the same documented backend services and validated data products. The interface layer must not independently redefine scientific calculations.

## 4. Workstream 1 — Data processing
### Foundation and harmonization

**Objective:** Build a repeatable, traceable pipeline that prepares MODIS and VIIRS observations for scientifically defensible comparison.

### 4.1 Source acquisition and inventory

- Identify and document the selected NASA data sources and access methods, such as FIRMS and relevant NASA archives or services.
- Confirm product availability, licensing/data-use requirements, fields, spatial and temporal coverage, update frequency, and known limitations.
- Begin with a representative geographic area and historical period for the MVP.
- Record source file identifiers, retrieval details, product versions, and processing dates.
- Retain raw source files unchanged in an archive.

### 4.2 Efficient storage and reproducibility

- Convert suitable source tables to compressed, columnar Parquet files, partitioned by useful dimensions such as year and region where appropriate.
- Use DuckDB for SQL queries directly over Parquet; use pandas or Polars for transformations where appropriate.
- Select only the columns needed for each processing task while retaining a complete raw archive.
- Verify row counts, schemas, missing values, key field distributions, and reproducibility of saved queries.
- Keep processing scripts, configuration, and data-version information under version control where practical.

### 4.3 Standardization

- Define a documented common schema without erasing source-specific fields.
- Preserve original timestamps and add a consistent UTC timestamp representation.
- Add local solar time only where it supports a defined analysis; do not confuse it with UTC or civil time.
- Standardize units, missing-value handling, field names, satellite and instrument identifiers, and data types.
- Preserve relevant fields, including source record identifier, satellite, instrument, acquisition time, coordinates, fire radiative power (FRP), confidence or quality information, scan, track, and product/version metadata when available.
- Keep source confidence definitions distinct rather than assuming that MODIS numeric confidence and VIIRS categorical confidence are directly equivalent.

### 4.4 Scan geometry and footprint effects

- Investigate how scan position and pixel footprint size affect observed counts, especially toward the edge of the MODIS swath.
- Use documented scan/track or footprint information where available; do not assume that fields have identical meanings across products.
- Compare candidate treatments such as retaining footprint size as a model input, area-aware analysis, stratified analysis, or excluding observations under a justified rule.
- Evaluate whether a treatment reduces scan-angle-related artifacts while preserving genuine geographic and temporal patterns.
- Do not regard a flatter count-versus-scan-angle plot alone as proof of a correct correction.

### 4.5 Spatial and temporal alignment

- Define matching windows based on acquisition time and geographic proximity, with explicit, configurable assumptions.
- Represent cross-sensor links as candidate matches with evidence and distance/time differences, not automatic proof of one physical fire.
- Consider aggregation to a common spatial grid (for example, H3 or a regular grid) at a scale suitable for the intended analysis.
- Do not imply that aggregation creates finer-resolution information. Moving from fine observations to a coarser grid is an aggregation; inferring fine-scale detail from coarse observations requires a model and additional validation.
- Keep original detections available alongside any aggregated products.
- If clustering detections into events, label outputs as candidate fire-event clusters until independently validated. Test sensitivity to alternative spatial radii and time windows.

### 4.6 Confidence and corroboration

- Start with documented sensor-specific quality filters and test how results change under alternative thresholds.
- Explore a shared calibrated score only if suitable independent reference data and a defensible calibration target are available.
- Distinguish sensor-provided confidence, cross-sensor corroboration, and the probability that an observation represents a real fire.
- Agreement between sensors is evidence of corroboration, not ground truth: both sensors can miss an event, and both can share sources of error.
- If calibration is performed, use held-out data and evaluate calibration curves and suitable metrics. Do not interpret a corroboration rate as a true-fire probability without independent validation.
- Physical variables such as FRP or brightness temperature may be useful additional features, but they are not automatically substitutes for detection confidence.

### 4.7 Harmonized data products

Produce clearly versioned data products that may include:
- Standardized observation records.
- Candidate cross-sensor matches.
- Common-grid summaries.
- Candidate event clusters.
- Sensor-specific counts and quality summaries.
- Validated harmonized metrics, only where the method and assumptions support them.
- Provenance metadata, quality flags, uncertainty indicators, and known limitations.

Every derived product should document its input sources, processing version, parameters, assumptions, and validation status.

## 5. Workstream 2 — Validation and backend

**Objective:** Demonstrate that the data products are scientifically defensible, reproducible, and accessible through stable services.

### 5.1 Validation strategy

- Define testable questions and success criteria before selecting a final method.
- Validate on representative regions, seasons, sensor conditions, and time periods.
- Where possible, use independent reference data or a defensible reference methodology. Clearly document when independent truth data are unavailable.
- Compare sensor-specific and combined counts; inspect missingness, data gaps, spatial artifacts, seasonal consistency, and sensitivity to processing parameters.
- Use temporal holdouts for forecasting or trend-prediction evaluation; avoid random splits that leak temporal information.
- Compare advanced methods with simple baselines, such as seasonal climatology or a suitable previous-period baseline.
- Use metrics appropriate to the task: count errors such as MAE/RMSE for count prediction; precision-recall and Brier score for suitable labeled probability tasks; calibration curves for probability calibration; and uncertainty coverage checks for prediction intervals.
- Investigate apparent historical discontinuities instead of forcing them to disappear. Correct only effects supported by evidence, while preserving the possibility of real changes in fire activity.
- Record results, failure cases, uncertainty, and known limitations in a reproducible validation report.

### 5.2 Trends, anomalies, and forecasting

- Begin with descriptive time series, seasonal baselines, day-of-year percentiles, robust anomaly scores such as median/MAD, and change-point analysis where useful.
- Add forecasting or machine learning only when the intended user need is clear and the model outperforms a simple baseline on held-out future periods.
- Candidate later methods may include STL decomposition, change-point tools, gradient-boosted trees, or probabilistic methods, subject to data suitability.
- Keep descriptive harmonization, anomaly detection, and forecasting as separate functions with separate validation criteria.
- Do not describe a historical anomaly score as a forecast or a fire probability unless it has been designed and validated for that purpose.

### 5.3 Backend services and data access

- Build a modular backend API (FastAPI is a recommended option) over the shared, versioned data products.
- Provide documented endpoints for observations, geographic/time filters, sensor comparisons, aggregated summaries, metadata, provenance, and exports.
- Ensure all interfaces use consistent definitions and calculations.
- Include pagination or bounded queries, input validation, clear error responses, and API documentation.
- Make data freshness, processing version, quality status, and known limitations accessible to clients.
- Keep the initial system simple. Introduce streaming systems, caches, queues, or distributed processing only when measured workload or latency requirements justify them.

### 5.4 Testing and operational foundations

- Test schema validation, transformations, spatial/temporal matching, aggregation, provenance, and API behavior.
- Use repeatable environments and deployment, for example Python, Pytest, Docker, and CI/CD.
- Monitor pipeline success/failure, data freshness, record counts, API health, and processing errors.
- Define recovery procedures and retain logs sufficient to investigate failures.
- Apply appropriate access control, secret management, dependency maintenance, and security practices.
- Establish explicit operational acceptance criteria before providing alerts for real-world response decisions.

## 6. Workstream 3 — Frontend MVP

**Objective:** Demonstrate the validated data pipeline through a small, understandable interface backed by the shared API.

### 6.1 MVP interface

Build a focused interface with:
- An interactive map showing detections or validated aggregates.
- A historical activity calendar or time-series view.
- Filters for date range, region, satellite/sensor, and quality status.
- Side-by-side or layered sensor comparisons where appropriate.
- A record-detail view showing source sensor, time, location, quality fields, and provenance.
- Clear labels distinguishing raw detections, candidate matches, candidate event clusters, and validated harmonized products.
- Visible data freshness, processing version, uncertainty, and limitations.
- Export of selected records or summaries where practical.

The MVP should demonstrate the complete path from original observations to processed results and user-facing display. It should not imply that every candidate match is a confirmed fire or that every harmonized metric is validated.

### 6.2 Recommended implementation choices

These are recommendations, not requirements mandated by the project:
- **Processing:** Python with pandas or Polars.
- **Storage/query:** Parquet and DuckDB for files; PostgreSQL with PostGIS when persistent multi-user spatial querying is needed.
- **Raw archive:** S3-compatible object storage or an appropriate available storage service.
- **API:** FastAPI.
- **Frontend:** React with TypeScript.
- **Map:** MapLibre GL JS.
- **Charts:** Plotly or Apache ECharts.
- **Scheduling:** Start with simple scheduled jobs; consider Prefect if orchestration needs grow.
- **Testing/deployment:** Pytest, Docker, and CI/CD.

Choose tools based on team skill, deployment environment, workload, and challenge constraints. Avoid unnecessary infrastructure complexity in the MVP.

## 7. Validation gates

Progress should depend on evidence, not only completion of tasks.

### Gate A — Data readiness
- Source access and product selection are documented.
- Required fields and their meanings are understood.
- Raw data are preserved and standardized outputs are reproducible.
- Data quality checks and source lineage work.

### Gate B — Harmonization validity
- Matching and aggregation assumptions are explicit.
- Candidate matches and clusters are not misrepresented as ground truth.
- Scan geometry and other relevant artifacts have been investigated.
- Sensitivity tests reveal the stability and limitations of the method.

### Gate C — Calibration and analytical validity
- Any common score or metric has a clear target and defensible validation method.
- Independent evidence is used where available.
- Held-out evaluation and appropriate metrics are reported.
- Apparent discontinuities are investigated rather than automatically removed.

### Gate D — Product readiness
- Map, calendar, exports, and API agree with the same underlying data definitions.
- Results are traceable to source observations and processing versions.
- Freshness, uncertainty, and limitations are clearly shown.

### Gate E — Operational readiness
- Performance, recovery, security, monitoring, and alert latency are tested against defined requirements.
- Alerting is not presented as operationally reliable until it has passed the relevant validation and acceptance process.

## 8. Main risks and mitigations

| Risk | Mitigation |
|---|---|
| Sensor observations disagree | Preserve sensor-specific records and model differences explicitly. |
| Candidate matches are mistaken for confirmed fires | Label them as candidate matches and retain match evidence and uncertainty. |
| Confidence scales are treated as equivalent | Keep original confidence definitions and calibrate only with a defensible target. |
| Edge geometry creates spatial artifacts | Test footprint-aware approaches and check for preserved real patterns. |
| Clustering merges separate fires or splits one event | Test multiple parameters and validate against suitable evidence. |
| Harmonization erases real trends | Investigate discontinuities and correct only supported sensor effects. |
| MVP expands too quickly | Limit initial geography, period, interfaces, and analytical claims. |
| Advanced models add complexity without benefit | Require measurable improvement over simple baselines. |
| Operational alerts are trusted prematurely | Validate separately for the intended operational use and display freshness and uncertainty. |

## 9. Definition of MVP success

The MVP is successful when the team can:
1. Acquire and process representative MODIS and VIIRS records reproducibly.
2. Preserve original observations and trace displayed results to their sources.
3. Explain how candidate matches, common-grid summaries, or other derived products were produced.
4. Show sensor-specific results and comparisons without claiming unsupported equivalence.
5. Demonstrate quality checks, sensitivity analysis, and documented limitations.
6. Display the processed data in a working map and historical view backed by a common API.
7. Explain what the method can establish, what remains uncertain, and what requires further validation.

## 10. Long-term direction

After the MVP and validation gates are met, expand the same foundation into the full three-audience platform:
- Historical analysis and archive tools for scientists.
- Flexible querying and export for researchers.
- A carefully validated map and alert feed for firefighters and operational users.
- More extensive coverage, automation, analytical models, monitoring, reliability, and support as justified by evidence and user needs.

The long-term platform should remain modular and versioned. Scientific methods, data products, APIs, and interfaces should evolve without losing reproducibility or obscuring the origin and uncertainty of results.

## 11. Working decisions and open questions

Decisions established in this roadmap:
- Use the three workstreams: **Data processing (foundation and harmonization)**, **Validation and backend**, and **Frontend MVP**.
- Use a limited, representative dataset for the first end-to-end demonstration.
- Adopt efficient storage, reproducible transformations, transparent spatial/temporal matching, and provenance.
- Treat the technical methods as candidates to test rather than assumptions already proven.
- Preserve the original platform vision of three interfaces sharing one validated data foundation.

Questions to resolve during implementation:
- Which exact FIRMS products and other reference datasets will be used?
- What geographic region and historical period best represent the challenge?
- What independent evidence is available to validate matches or probabilities?
- What grid resolution, temporal matching window, and event-clustering rules are justified?
- Which harmonized metrics can be supported by the available data?
- What are the judging rubric, deployment constraints, and intended users' acceptance criteria?

This document is the shared roadmap and source of truth. Specific implementation choices may be refined as evidence becomes available, but changes should preserve provenance, transparent assumptions, and scientific validation.
