# Evaluations

Test scenarios for the elementary-data-quality skill. Each scenario tests whether the skill produces correct, Fabric-specific Elementary guidance.

## Scenarios

### Scenario 1: Missing timestamp_column diagnosis

**Prompt**: "I set up elementary volume_anomalies on my staging payments table, and dbt test passes every day with no warnings. But I can clearly see that yesterday's load was 80% smaller than normal. Why isn't Elementary catching this?"

**Expected behavior**: Claude should immediately identify the most likely cause as a missing `timestamp_column` configuration. It should explain that without `timestamp_column`, Elementary compares cumulative row counts instead of time-bucketed volumes, meaning the 80% daily drop gets averaged into the total. Claude should provide the corrected YAML with `timestamp_column` set at the model config level, and suggest checking `training_period` and `detection_period` as secondary causes.

**Pass criteria**:
1. Identifies missing `timestamp_column` as the primary cause (not sensitivity, not training period)
2. Provides corrected YAML with `config.elementary.timestamp_column` at the model level

### Scenario 2: Test placement for a new bronze source

**Prompt**: "We just added a new dlt pipeline that loads Salesforce accounts into our bronze lakehouse on Fabric. What Elementary tests should I put on this raw table?"

**Expected behavior**: Claude should recommend `schema_changes_from_baseline` (with `fail_on_added: true` and `enforce_types: true`) and `volume_anomalies` (with `fail_on_zero: true`). It should NOT recommend `column_anomalies` or `all_columns_anomalies` on a bronze table (raw data is too noisy). It should provide complete YAML with Fabric-compatible data types in the baseline (e.g., `datetime2` not `timestamp`, `bit` not `boolean`). It should note that `_dlt_load_id` is a string, not a proper timestamp for time bucketing.

**Pass criteria**:
1. Recommends schema baseline + volume tests, does NOT recommend column anomalies on bronze
2. Uses Fabric/T-SQL data types in the schema baseline (`datetime2`, `nvarchar`, `bit`)

### Scenario 3: Severity strategy for new deployment

**Prompt**: "We're rolling out Elementary tests across our entire dbt project on Fabric for the first time. Should all tests use severity error so dbt build fails on bad data?"

**Expected behavior**: Claude should advise against using `severity: error` on all tests from day one. It should explain the severity progression strategy: start with `severity: warn` for anomaly detection tests (they need a baseline training period of 1-2 weeks), use `severity: error` immediately only for `schema_changes_from_baseline` and `volume_anomalies` with `fail_on_zero: true` (these don't need a baseline). Claude should provide a rollout timeline: week 0 deploy with warn, weeks 1-2 monitor and tune sensitivity, week 2+ promote stable tests to error.

**Pass criteria**:
1. Recommends `warn` as default for anomaly tests, `error` only for schema and fail_on_zero
2. Provides a phased rollout timeline with specific week markers

### Scenario 4: Distinguishing dbt package from edr CLI

**Prompt**: "I installed the Elementary dbt package and can run tests. Now I want to set up Slack alerts when tests fail. How do I configure Elementary to send Slack notifications?"

**Expected behavior**: Claude should clearly distinguish the two Elementary components: the dbt package (which they already have, handles test execution and result collection) and the `edr` CLI (a separate Python tool they need to install for reports and alerts). Claude should explain that the dbt package does NOT send alerts -- it only stores results in warehouse tables. They need to install `edr` (`pip install elementary-data[slack]`), configure it with `profiles.yml` pointing to their Fabric warehouse, and run `edr send-report --slack-channel-name <channel>`. It should note that `edr` is a separate installation from the dbt package.

**Pass criteria**:
1. Explicitly states the dbt package cannot send alerts and `edr` CLI is required
2. Distinguishes installation of `edr` as separate from the dbt package

### Scenario 5: Column anomalies monitor selection

**Prompt**: "I want to add Elementary column_anomalies to the email and revenue columns on my stg_customers model on Fabric. What monitors should I use?"

**Expected behavior**: Claude should recommend different monitors for each column type. For `email` (string): `missing_count`, `null_count`, `min_length`, `max_length`. For `revenue` (numeric): `min`, `max`, `zero_count`, `null_count`. Claude should explicitly advise against omitting the `column_anomalies` parameter (which runs all default monitors -- slow and noisy). It should set `timestamp_column` at the model level and use `severity: warn`. The `where_expression` examples, if any, should use T-SQL syntax.

**Pass criteria**:
1. Provides different monitor lists for string vs numeric columns
2. Explicitly sets `column_anomalies` parameter (does not rely on defaults)
