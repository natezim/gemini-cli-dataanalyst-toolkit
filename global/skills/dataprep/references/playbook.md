# Phased Migration Playbook

Migrating an enterprise Dataprep estate is a 4-phase project. Don't try to do all four in parallel — each phase has a discrete deliverable and a discrete decision point.

## Phase I — Discovery

**Objective:** know what you have. No code written yet.

**Actions:**
1. Inventory flows: `GET /v4/flows` — list count, owners, last-modified dates.
2. Inventory data sources: every BigQuery table, GCS URI, DB connection referenced across all flows.
3. Inventory outputs: every destination, format, ingestion mode.
4. Inventory schedules: which flows run when, on what cadence, with what downstream dependencies.
5. Inventory webhooks: what notifications fire, where they go.
6. Build the dependency DAG: which flows feed which other flows? Identify entry points and terminal nodes.

**Deliverable:** a spreadsheet or doc listing every flow with its: owner, sources, outputs, schedule, downstream consumers, criticality (P0/P1/P2).

**Decision point:** which flows are dead and can just be turned off? You'll almost always find some — dead flows are zero-cost migration wins.

## Phase II — Compilation

**Objective:** translate every active flow into target code (Python or BigQuery SQL).

**Actions:**
1. Pick the target per flow:
   - Source is BigQuery + transforms are mostly SQL-shaped → BigQuery SQL
   - Source is large/distributed + needs Spark idioms → PySpark
   - Source is small + single-machine notebook work → pandas
2. For each flow, try `@dataprep-migrator` (or the trifacta SDK's `get_pandas()`) for an initial translation.
3. For each untranslatable construct (NUMFORMAT, TBE, custom functions), write the manual replacement.
4. Apply the three risk-area fixes (`migration-risks.md`) explicitly to every translation:
   - Timezone-naive datetime casts
   - Decimal precision scaled to 38 max
   - `coalesce` / `fillna` / `IFNULL` around any null-sensitive operation
5. Store generated code in version control alongside the original flow ID for traceability.

**Deliverable:** for each active flow, a working script in your target language with documented translation notes.

**Decision point:** any flow that needs more than ~50% manual rewrite — is it worth migrating, or is it cheaper to rewrite from scratch using business requirements directly?

## Phase III — Validation

**Objective:** prove the new code produces identical output to the legacy flow.

**Actions:**
1. Run legacy Dataprep flow against representative production data, capture output.
2. Run new code against the same input, capture output.
3. Apply the 4-tier validation framework (`validation.md`):
   - Tier 1: schema parity
   - Tier 2: row count match
   - Tier 3: row-level MD5 checksum match
   - Tier 4: cell-level diff on any mismatches
4. For any tier that fails, fix and re-run. Don't proceed to cutover with known divergence unless it's explicitly accepted (and documented).
5. Run side-by-side for at least one full schedule cycle (e.g., a week of daily runs). Catch edge cases that representative samples miss.

**Deliverable:** for each migrated flow, a parity report showing all four tiers pass on real data.

**Decision point:** any flow where parity can't be achieved — investigate whether the legacy behavior is a bug being relied on, or whether the new code has a real translation error.

## Phase IV — Cutover

**Objective:** retire the legacy flow, route consumers to the new pipeline.

**Actions:**
1. Deploy validated code to your orchestrator:
   - PySpark → Cloud Composer (Airflow) DAG, Dataproc, Databricks job
   - BigQuery SQL → scheduled query, dbt model, or Composer DAG with `BigQueryInsertJobOperator`
   - pandas → scheduled Python script (Cloud Functions, Cloud Run, Composer)
2. Run new and legacy side-by-side for the agreed parity period (typically 2-4 weeks of production cycles).
3. Update downstream consumer connection strings or table references to point at new output.
4. Update monitoring, alerts, and SLAs to track the new pipeline.
5. Turn OFF the legacy Dataprep flow (don't delete yet — pause).
6. After confirmed-stable period (4-8 weeks), delete the legacy flow.

**Deliverable:** Dataprep flow disabled, new pipeline in production with monitoring.

**Decision point:** when can you turn off the Dataprep license / workspace entirely? Track this as a project end-state metric.

## Anti-patterns to avoid

- **Big bang cutover.** Migrate one flow at a time, prove parity, move on. Never cut over multiple flows in the same window.
- **Skipping Phase III.** "It looks right" is not parity. Always run all four validation tiers.
- **Letting the SDK auto-generate without review.** The Trifacta SDK's `get_pandas()` works for simple cases and silently produces wrong code for complex ones. Always review.
- **Ignoring the three risk areas.** Temporal drift, decimal precision, null propagation will bite you. Address them upfront, not after a production incident.
- **Treating the migration as IT work without business signoff.** Downstream consumers should validate the new output on real reports/dashboards before you turn off the legacy flow.

## Companion tools

- `@dataprep-migrator` — translates individual recipes
- `@query-validator` — dry-run cost check before running new BigQuery SQL
- `@data-profiler` — quick parity check on output shape
- `@solution-designer` — for the Discovery phase decisions (what to migrate, what to retire)
