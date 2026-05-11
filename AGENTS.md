# Subagent Cheatsheet

Quick reference for when to invoke which agent. Type `@` in chat to bring up the picker, or call by name.

All custom agents are read-only and parallel-safe except `@report-drafter` (write-scoped to `output/reports/`).

## Decision tree

```
What are you trying to do?

├─ I'm not sure how to approach this
│    └─ @solution-designer — asks clarifying questions, gives 2-3 options with tradeoffs
│
├─ Something with SQL
│    ├─ Validate before running
│    │    └─ @query-validator — syntax + dry-run cost check
│    ├─ Understand the schema
│    │    └─ @schema-explorer — Q&A from context/ without DDL dump
│    └─ Profile the result
│         └─ @data-profiler — shape, nulls, cardinality
│
├─ Statistics or modeling question
│    └─ @stats-advisor — right test, assumptions, exact library call
│
├─ Translate SAS code to Python
│    └─ @sas-migrator — DATA/PROC/macros → pandas/statsmodels
│
├─ Review Microsoft analytics work
│    ├─ Excel formulas / dynamic arrays
│    │    └─ @excel-reviewer
│    ├─ Power Query M code (Excel or Power BI)
│    │    └─ @powerquery-reviewer — focuses on query folding
│    └─ DAX measures (Power Pivot or Power BI)
│         └─ @dax-reviewer
│
├─ Tableau or Power BI work
│    ├─ Inventory / governance / permissions
│    │    └─ @tableau-auditor
│    ├─ Performance issues in a workbook/report
│    │    └─ @viz-optimizer
│    └─ Migrating between Tableau and Power BI
│         └─ @migration-mapper
│
├─ Write up findings as a report
│    └─ @report-drafter — markdown to output/reports/
│
└─ General research / investigation (built-ins)
     ├─ @generalist — heavy work that would bloat main context
     ├─ @codebase_investigator — deep file/dependency tracing
     └─ @cli_help — Gemini CLI questions
```

## Example prompts

**Brainstorming:**
- `@solution-designer I need to build a churn dashboard but I don't know where to start`
- `@solution-designer should this transform live in Power Query or a stored procedure?`
- `@solution-designer we're migrating off SAS, help me sequence it`

**SQL:**
- `@query-validator dry-run output/queries/q1_revenue.sql against BigQuery`
- `@schema-explorer what tables hold customer subscription state?`
- `@data-profiler profile output/data/q1_export.csv`

**Stats:**
- `@stats-advisor I have 200 users in two groups, how do I test if conversion differs?`
- `@stats-advisor what's the right way to forecast monthly orders given 18 months of data?`

**SAS:**
- `@sas-migrator translate this PROC LOGISTIC block to Python`
- `@sas-migrator port DATA step from legacy/customer_clean.sas`

**Excel / Power Query / DAX:**
- `@excel-reviewer review the formulas in context/finance_model.xlsx`
- `@powerquery-reviewer check folding for the Customers query in this PBIP`
- `@dax-reviewer review the Sales measures in TMDL/Model.tmdl`

**BI tools:**
- `@tableau-auditor list workbooks not viewed in 90 days under the Finance project`
- `@viz-optimizer why is this PBIX slow? performance.pbix`
- `@migration-mapper plan the migration of the Sales Tableau workbook to Power BI`

**Reports:**
- `@report-drafter draft a Q1 revenue analysis from output/queries/q1_revenue.sql results`

## Parallel patterns

These agents are read-only — fire them concurrently when you need multi-domain analysis:

```
"Profile output/data/customers.csv with @data-profiler
 AND validate output/queries/customer_segments.sql with @query-validator
 AND check the relevant schema with @schema-explorer."
```

The orchestrator will dispatch all three in parallel, each in its own context window, and synthesize their summaries.

## Hand-off pattern

`@solution-designer` always ends with a "next action" pointing at a specialist. Honor the handoff — don't re-ask the same question of the wrong agent.

```
@solution-designer: ...PLAN: 1. validate join → @query-validator
                          2. profile result → @data-profiler
                          3. write up → @report-drafter
NEXT ACTION: validate the join in q1_revenue.sql
```

## What NOT to do

- **Don't ask `@report-drafter` to do analysis.** It writes up findings the orchestrator gives it. Run the analysis first.
- **Don't dispatch two writers at the same target.** State-mutating agents must be sequential. Currently only `@report-drafter` writes — but if you add custom write-scoped agents, never run them in parallel against the same file.
- **Don't bypass `@solution-designer` for vague goals.** Jumping straight to a specialist with a fuzzy question wastes their narrow scope.
- **Don't re-invoke an agent to "double-check."** Each agent returns a compressed summary — trust it. If you need verification, dispatch a different agent (e.g., `@query-validator` then `@data-profiler` on the result).
