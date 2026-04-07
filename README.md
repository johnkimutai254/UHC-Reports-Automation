# UHC-Reports-Automation

# Automating UHC custom reports after the monthly cloud run

This document describes how to run **UHC custom (front door) filtered** SCRAPPY reports on the same cloud platform as the standard monthly batch, **after** that batch completes. It is aimed at engineers and analysts who own scheduling and infra changes.

For context on *what* these reports are, see the [UHC Front Door Reporting](README.md#uhc-front-door-reporting) section in `README.md` and the notebook `UHC_Custom_Reporting.ipynb` (Jira: REPAUTO-2502, REPAUTO-2986).

---

## 1. How the monthly cloud run works today

Production schedules live in **`platform/serviceConfig.yaml`** under `cronJobs`.

The standard monthly job is **`customerinsights-monthly-reports`**:

- **Schedule:** `0 1 6 * *` (01:00 UTC on the 6th of every month; confirm the intended timezone with your platform team—Kubernetes cron schedules are typically cluster-local).
- **Command:** runs `run_reports.py` in batch mode with monthly cycle, queries, and publish:

```text
cd /src/modules/standard_reports/project_scrappy/ && exec poetry run python run_reports.py --mode batch --monthly --execute-query --publish-reports
```

That job produces the full client batch for the **inferred** monthly period (see `analyst_selections.infer_recent_period` for monthly date logic).

UHC custom reporting **does not** use the batch `--custom-uhc-report-type` flag on the full batch run. That flag injects SQL filters via `CUSTOM_UHC_REPORT_TYPE` in `global_functions.run_query` and is intended for **targeted manual runs** (see below), not for the entire monthly client list.

---

## 2. What “UHC automation” should run

The maintained automation path in code is the same as **`UHC_Custom_Reporting.ipynb`**:

- **Three aggregation IDs** (stable unless Sales Cloud / config changes):

  | Population | ID |
  |------------|-----|
  | UHC Employer & Individual (E&I), `custom1` | `aAW5d000000Xwt5GAC` |
  | UHC Individual & Family Plan (IFP), `custom1` | `aAW5d000000XwtAGAS` |
  | UHC Surest, `client_populations` | `aAW5d000000Xw4mGAC` |

- **Five `--custom-uhc-report-type` values** (each run uses one at a time):

  - `uc`, `epc`, `uc_no_bh`, `epc_no_bh`, `no_bh`

The notebook runs `run_reports.py` **once per report type** (five executions), each time with **manual** mode, **monthly** cycle, the same date range, the three comma-separated aggregation IDs, and `--execute-query` (and typically publish when you want production outputs).

That is the pattern to replicate in cloud: **five sequential manual runs** after the monthly batch, **or** one wrapper that loops in order.

**Note:** `UHC Front Door Reporting.ipynb` is a separate, analyst-driven workflow (Google Drive / Slides steps). This doc focuses on **`run_reports.py` + `CUSTOM_UHC_REPORT_TYPE`**, which matches `UHC_Custom_Reporting.ipynb`.

---

## 3. Why a separate job after the monthly batch

1. **Isolation:** Custom UHC filters must not be applied to non-UHC clients in batch mode.
2. **Same reporting period:** You want the same inferred month as `customerinsights-monthly-reports` (or an explicitly aligned date range).
3. **Operational clarity:** Failures, retries, and alerts for UHC can be separated from the full batch.
4. **Runtime:** Five manual runs add wall-time and API load; running them after the batch avoids competing for the same window as the full client set.

---

## 4. Recommended scheduling strategy

Kubernetes `CronJob` does **not** express “start after job A finishes.” You only have **time-based** schedules. Use one of these:

| Approach | Pros | Cons |
|----------|------|------|
| **Later cron on the same calendar day** | Simple, matches existing `serviceConfig.yaml` style | If the monthly job overruns, UHC could start too early—mitigate with a generous delay or monitoring |
| **Next calendar day** | Strong separation | Period inference must still match the monthly run (usually OK if both run in the same month for the same “complete month” logic—validate once) |
| **External orchestrator** (e.g. workflow engine with dependencies) | True “after success” semantics | More infra to own |

**Practical default:** schedule the UHC job **several hours after** the monthly job on the **same** day (e.g. monthly at `0 1 6 * *`, UHC wrapper at `0 8 6 * *` or `0 1 7 * *`—tune with your team). Confirm timezone and whether the monthly job ever slips past its window.

---

## 5. Commands to run (local or cloud)

Working directory in the container (same as existing cron jobs):

```text
/src/modules/standard_reports/project_scrappy/
```

**Option A — explicit dates** (match the monthly period if you know `REPORT_START_DATE` / `REPORT_END_DATE`):

```bash
poetry run python run_reports.py --mode manual --monthly \
  --report-start-date "YYYY-MM-DD" \
  --report-end-date "YYYY-MM-DD" \
  --aggregation-ids "aAW5d000000Xwt5GAC,aAW5d000000XwtAGAS,aAW5d000000Xw4mGAC" \
  --execute-query \
  --publish-reports \
  --custom-uhc-report-type epc
```

Repeat for `uc`, `uc_no_bh`, `epc_no_bh`, and `no_bh`.

**Option B — inferred dates** (omit `--report-start-date` / `--report-end-date` so `analyst_selections` infers the recent complete month, same as a typical local run without explicit dates). Only use this if you have verified inference matches production on the day the cron runs.

**Shell wrapper example** (single CronJob, sequential runs):

```bash
#!/usr/bin/env sh
set -eu
cd /src/modules/standard_reports/project_scrappy/
for T in uc epc uc_no_bh epc_no_bh no_bh; do
  poetry run python run_reports.py --mode manual --monthly \
    --aggregation-ids "aAW5d000000Xwt5GAC,aAW5d000000XwtAGAS,aAW5d000000Xw4mGAC" \
    --execute-query --publish-reports \
    --custom-uhc-report-type "$T"
done
```

Add `--report-start-date` / `--report-end-date` to the loop if you require **lockstep** alignment with the monthly batch job’s period (e.g. computed by a small step that reads the same rules as batch or passes through CI).

---

## 6. Adding a new CronJob in `serviceConfig.yaml`

Follow the same pattern as `customerinsights-monthly-reports`:

1. Add a new entry under `cronJobs:` with a unique `name` (e.g. `customerinsights-uhc-monthly-reports`).
2. Use the same `image` and `iamRole` as other SCRAPPY jobs unless platform requires otherwise.
3. Set `command` / `args` to run the wrapper script or inline `sh -c '...'` as in existing jobs.
4. Set **resources** (`requests`/`limits`): start from the monthly job (**17Gi** memory in the current config) and adjust after observing peak usage—UHC runs fewer clients but five passes.
5. Deploy with your team’s standard **customerinsights-analytics** release process.

**Illustrative snippet** (schedules and wrapper path must be finalized by your team):

```yaml
  - name: customerinsights-uhc-monthly-reports
    iamRole: customerinsights-analytics
    image: customerinsights-analytics
    command: "sh"
    args:
      - "-c"
      - |
        cd /src/modules/standard_reports/project_scrappy/ && exec sh /src/path/to/uhc_monthly_wrapper.sh
    schedule: "0 8 6 * *"
    resources:
      requests:
        memory: "17Gi"
        cpu: "3.4"
      limits:
        memory: "17Gi"
```

Replace the wrapper path with a real script checked into the repo if you add one (recommended for reviewability).

---

## 7. `marcom_prep` and optional skips

`execute_reports.py` runs **`marcom_prep.py`** after a successful monthly publish when `publish_reports` is true. UHC manual monthly runs may trigger that path as well.

If UHC runs should **not** feed marcom the same way as the full batch:

- Set **`SKIP_MARCOM_PREP`** environment variable for the UHC job (see `execute_reports.py`—supported values include `1`, `true`, `yes`, `on`).

Confirm with stakeholders before skipping.

---

## 8. Validation checklist

Before turning on production:

1. **Dry run** in a lower environment or with `--publish-reports` omitted (if your process allows) and verify outputs.
2. **Date alignment:** Compare inferred or explicit `REPORT_START_DATE` / `REPORT_END_DATE` with the same monthly period as `customerinsights-monthly-reports`.
3. **Aggregation IDs:** Reconfirm in Salesforce / internal config that the three IDs are still correct for E&I, IFP, and Surest.
4. **Google APIs:** Cloud batch jobs already use container credentials; ensure UHC runs do not require interactive OAuth (the notebook mentions Quickstart prompts—those are manual-notebook concerns; cloud should use the same service path as monthly batch).
5. **Alerts:** Attach the same logging/monitoring as other `customerinsights-*` cron jobs so failures page the channel (e.g. `#cloud-platform-deployment` per `serviceConfig.yaml` notifications).
6. **Downstream:** The notebook copies/renames some Drive artifacts; if automation must replicate naming, add that as a follow-up script or API step—do not assume `run_reports.py` alone matches every notebook post-step.

---

## 9. Related files

| File | Role |
|------|------|
| `platform/serviceConfig.yaml` | Cron schedules and container commands for monthly/quarterly jobs |
| `run_reports.py` | CLI → environment variables for `execute_reports.py` |
| `analyst_selections.py` | Reads `CUSTOM_UHC_REPORT_TYPE`, `AGGREGATION_IDS`, batch vs manual |
| `global_functions.py` | Applies UHC SQL filters when `CUSTOM_UHC_REPORT_TYPE` is set |
| `UHC_Custom_Reporting.ipynb` | Reference for IDs, report types, and ordering |
| `README.md` | UHC Front Door narrative and notebook-based instructions |

---

## 10. Summary

To automate UHC custom reports **after** the cloud monthly run:

1. Add a **second CronJob** scheduled **after** `customerinsights-monthly-reports`.
2. Run **manual** monthly `run_reports.py` with the **three UHC aggregation IDs** and each **`--custom-uhc-report-type`** in sequence (`uc`, `epc`, `uc_no_bh`, `epc_no_bh`, `no_bh`).
3. Align **report dates** with the monthly batch (explicit or verified inference).
4. Decide **`SKIP_MARCOM_PREP`** for UHC runs if needed.
5. Validate in staging, then ship the `serviceConfig.yaml` change through your deployment pipeline.
