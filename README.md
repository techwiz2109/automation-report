# TC TruRisk Report Automation (`src/`)

Generates the **data sheet** behind the *TotalCloud TruRisk Insights Report* (the customer-facing PPTX deck).
Instead of hand-counting numbers out of ~190 JSON files, you point this tool at the folder SRE produced
and it writes one plain-text file with every number, percentage and "Top 5" table the deck needs, labelled
by slide.

```
SRE runs script.sh on prod   ─►   <username>_<YYYY-MM-DD>/  (≈190 JSON files)
                                          │
                                          ▼
                         python main.py   (this folder)
                                          │
                                          ▼
                 TotalCloud_TruRisk_Report_Data_*.txt  ─►  copy values into the PPTX deck
```

For where every single number comes from (API / Elasticsearch query → file → formula → slide), see
[`../REPORT_DATA_GUIDE.md`](../REPORT_DATA_GUIDE.md).

---

## 1. Requirements

* Python 3.8+ (tested on 3.14)
* **No third-party packages.** Only the standard library is used (`os`, `json`, `math`, `datetime`).

## 2. Quick start

1. **Get the data folder from SRE.** SRE runs `script and data/script.sh` on the customer's POD:

   ```bash
   ./script.sh <customerUuid> <POD> <username> <days>
   # e.g. ./script.sh e7994ce3-.... USPOD1 acme_user 30
   ```

   This creates a folder named `<username>_<YYYY-MM-DD>/` with all the JSON files.
   See [`script and data/hneyw2ms_2026-01-19/`](../script%20and%20data/hneyw2ms_2026-01-19) for a sample.

2. **Edit `config.py`.** Only the first three lines change between runs:

   ```python
   INPUT_FOLDER  = r"/path/to/<username>_<YYYY-MM-DD>"   # folder from SRE
   OUTPUT_FOLDER = r"/path/to/where/the/report/goes"     # created if missing
   REPORT_NAME   = "TotalCloud_TruRisk_Report_Data.txt"
   ```

3. **Run it from inside `src/`.** The imports are relative to this folder:

   ```bash
   cd src
   python main.py
   ```

   You should see `Report generated successfully`. The report is **overwritten** on every run.

4. Open the `.txt`, then fill the deck slide by slide (the headings in the file say "Slide 3", "Slide 5" and so on).

> **Lines like `Error loading <file>: No such file or directory` are not fatal.** A missing file is
> treated as empty, so its numbers come out as **0**. Always read these lines. A 0 in the
> report can mean "the customer really has none" *or* "SRE didn't produce that file".

---

## 3. Folder layout

```
src/
├── main.py                    # Orchestrator: runs every section in slide order
├── config.py                  # Input/output paths + every file name, resource-type and insight mapping
├── handler/
│   ├── loader.py              # Reads JSON from INPUT_FOLDER (missing/invalid → {}), VM TruRisk lookup
│   └── report_append.py       # write_to_text_file(): appends one line to the report
└── core/
    ├── processor.py           # All calculations (counts, percentages, top-5 selection, fallbacks)
    ├── report_writer.py       # Formats processed data into the numbered text sections
    └── storage_db_helpers.py  # Adds "Additional Info" tags (versioning off, old TLS, public endpoint) to storage hits
```

Each report section follows the same three steps:

```
loader.load_*()  →  processor.process_*()  →  report_writer.write_*()
     (raw JSON)          (dict of numbers)          (lines in .txt)
```

---

## 4. What `main.py` produces (in order)

| # | Report section | Processor → Writer | Deck slide |
|---|---|---|---|
| 0 | Customer Information (customer UUID, user UUID, username, timestamp) | `process_username_data` → `write_user_section` | Cover (1–2) |
| 1 | Overall TruRisk Score (AWS, Azure, overall = max) | `process_trurisk_scores` → `write_trurisk_section` | 3 (gauge) |
| 2 | Assets with misconfigurations | `process_assets_with_misconfig` | 3 |
| 3 | Assets with active threats | `process_active_threats` | 3 |
| 4 | CISA known exploitable vulns. **Heading only, not implemented** | — | 3 |
| 5 | Publicly exposed workloads | `process_public_workloads` | 3 |
| 6 | Cloud security control failures | `process_controls` | 3 |
| 7 | IAM users & role misconfigurations | `process_iam_details` | 3 |
| 8 | CIS framework control failures | `process_cis_controls` | 3 |
| Slide 5 | Cloud Risk Summary per cloud (inventory, % at risk, accounts, critical vulns, risk score, CIS failed) | `process_cloud_risk_summary` | 5–11 |
| Slide 6 | Cloud Inventory (totals + 19 resource categories) | `process_inventory_resources` | 12 |
| 7.1 | Workload insights (≤ 6, top 5 resources each) | `process_workload_trurisk_insights` + `process_workload_controls` | 13–17 |
| 7.2 | Storage & database insights (≤ 5) | `process_storage_database_resources` + `process_storage_database_insights` | 19–24 |
| 7.3 | IAM insights (≤ 3) | `process_iam_insights` | 25–26 |
| Slide 23 | Risk reduction recommendations (4 sentences) | `process_risk_reduction` | 27 |
| Slide 24 | Best practices. **To-do, fill manually** | — | 28 |

## 5. Shared conventions in the code

* **Rounding:** `round_off()` rounds half up (`12.5 → 13`), unlike Python's banker's `round()`.
* **Percentages:** `failed / total × 100`, rounded as above. If the true value is between 0 and 0.5
  the report prints `~1` rather than `0`, so a small non-zero risk doesn't show as 0%.
* **Cloud order** is always AWS → Azure → GCP → OCI.
* **Totals vs. rows:** counts come from the file's `numberOfElements` (or `hits.total.value` for
  Elasticsearch files), which is the *true* total. The `content`/`hits` rows are capped by the
  script's page size (usually 1000), so "Top 5" picks are made from at most the first 1000 rows.
* **Top-5 lists** are sorted by TruRisk score (workloads) or by number of `additionalInfo` tags
  (storage). When several clouds contribute, they're **round-robined** (AWS, Azure, GCP, OCI, AWS…) so
  every cloud is represented.

## 6. Changing what the report looks at

Almost everything is driven by `config.py`, so most changes need no code:

| To… | Edit |
|---|---|
| Point at a new customer | `INPUT_FOLDER`, `OUTPUT_FOLDER`, `REPORT_NAME` |
| Rename an input file / adapt to a new script version | `SUMMARY_FILE_MAPPING`, `*_FILES` dicts |
| Count a new resource type in Slide 6 | `RESOURCE_TYPE_MAPPING` (also add lines in `write_inventory_resource_section`) |
| Add or reorder a workload TruRisk insight | `WORKLOAD_INSIGHT_FILES` (list order = priority; insight must exist in `07_00_TruRisk_Insights.json`) |
| Change which CSPM controls back-fill workload insights | `WORKLOAD_CONTROL_PRIORITY` + `WORKLOAD_CONTROL_FILES` |
| Storage/DB or IAM control sources | `STORAGE_DB_CONTROL_FILES`, `IAM_CONTROL_FILES` |
| More VM pages for the TruRisk-score lookup | `WORKLOAD_VM_PAGE_RANGE` (must match the `pg0..pgN` files the script writes) |

The *selection order* of storage/DB and IAM insights is hard-coded in `processor.py`
(`process_storage_database_insights`, `process_iam_insights`).

## 7. Known gaps & gotchas

* **Paths are hard-coded** in `config.py` (currently Windows paths). Edit them on every run.
* **Section 4 (CISA KEV)** prints only a heading. The script does fetch `03_04_01_CISA_known_vulnerability.json`
  but nothing processes it.
* **Slide 24 (Best Practices)** is not automated. The script already fetches its inputs
  (`24_01_0x_*_resources_with_no_encryption.json`, `22_03_IAM_*_with_high_permission_hitsTypeFile.json`).
* **Insights 111, 4, 5 and 13** are listed in `WORKLOAD_INSIGHT_FILES`, but `script.sh` never
  produces their files, so they're always skipped (with an `Error loading` line).
* **Older data folders** (e.g. the 2026-01-19 sample) contain `07_01_aws_all_vm_contentTypeFile.json` instead
  of the paged `07_01_*_all_vm_pg{0..4}_…` files. On those, CSPM-control workload hits show
  `TruRisk Score: NA` and an empty AWS resource name.
* **Misconfiguration fallback is global:** if *any* cloud's inventory has resources but 0 failed, *all*
  clouds switch to the Elasticsearch counts. In inventory mode the writer prints only the two overall
  totals and **no percentage line**.
* **GCP/OCI have no TruRisk score or critical-vuln count.** The overall TruRisk is `max(AWS, Azure)`.
* There are no tests.

## 8. Sample run

The repo contains a real (Jan 2026) data folder. To try the tool without editing `config.py`,
temporarily set `INPUT_FOLDER` to `../script and data/hneyw2ms_2026-01-19` and `OUTPUT_FOLDER` to any
scratch directory. It produces a ~460-line report. That folder holds **real customer data**, so
don't share the output.
