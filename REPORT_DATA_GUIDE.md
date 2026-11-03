# TC TruRisk Insights Report: Data Point Guide

**Who this is for:** anyone preparing a TotalCloud (TC) TruRisk Insights Report for the first time.
**What it explains:** for every number on every slide, what it means, where it comes from, how it is
calculated, and where to find it in the generated data file.

The slide numbers below follow the ACME sample deck (`script and data/TC TruRisk Insights Report.pptx`).

---

## Contents

- [0. Before you start: basics](#0-before-you-start-basics)
- [Slides 1–2: Cover and agenda](#slides-12-cover-and-agenda)
- [Slide 3: Overall Cloud Security Status](#slide-3-overall-cloud-security-status)
  - [How to read a tile](#how-to-read-a-tile)
  - [Data point 1: Overall TruRisk Score](#data-point-1-overall-trurisk-score)
  - [Data point 2: CISA known exploitable vulnerabilities](#data-point-2-cisa-known-exploitable-vulnerabilities)
  - [Data point 3: Assets with misconfigurations](#data-point-3-assets-with-misconfigurations)
  - [Data point 4: Assets with active threats](#data-point-4-assets-with-active-threats)
  - [Data point 5: Publicly exposed workloads](#data-point-5-publicly-exposed-workloads)
  - [Data point 6: Cloud security control failures](#data-point-6-cloud-security-control-failures)
  - [Data point 7: IAM users & role access misconfigurations](#data-point-7-iam-users--role-access-misconfigurations)
  - [Data point 8: CIS framework controls failures](#data-point-8-cis-framework-controls-failures)
- [Slides 5–11: Cloud Risk Summary](#slides-511-cloud-risk-summary)
- [Slide 12: Cloud Inventory](#slide-12-cloud-inventory)
- [Slide 13: TruRisk Insights overview](#slide-13-trurisk-insights-overview)
- [Slides 14–17: Workload insights (Top 5 tables)](#slides-1417-workload-insights)
- [Slide 18: Key Vulnerabilities](#slide-18-key-vulnerabilities)
- [Slides 19–24: Storage & Database insights](#slides-1924-storage--database-insights)
- [Slides 25–26: IAM insights](#slides-2526-iam-insights)
- [Slide 27: Risk Reduction Recommendations](#slide-27-risk-reduction-recommendations)
- [Slide 28: Best Practices Recommendations](#slide-28-best-practices-recommendations)
- [Appendix A: How the data is collected (SRE script)](#appendix-a-how-the-data-is-collected-sre-script)
- [Appendix B: Data source index](#appendix-b-data-source-index)
- [Appendix C: Final checklist](#appendix-c-final-checklist)

---

## 0. Before you start: basics

### 0.1 The pipeline in one picture

```
 ┌──────────────────────┐     ┌───────────────────────────┐     ┌────────────────────┐     ┌──────────────┐
 │ 1. SRE runs          │     │ 2. Folder of ~190 JSON    │     │ 3. You run         │     │ 4. You copy  │
 │    script.sh on the  │ ──► │    files, one per query   │ ──► │    src/main.py     │ ──► │    numbers   │
 │    customer's POD    │     │    <username>_<date>/     │     │    → .txt report   │     │    into PPTX │
 └──────────────────────┘     └───────────────────────────┘     └────────────────────┘     └──────────────┘
```

* **Step 1** is done by SRE because only they can reach the production servers (see [Appendix A](#appendix-a-how-the-data-is-collected-sre-script)).
* **Step 3** turns the raw files into finished numbers. How to run it: [`src/README.md`](src/README.md).
* **Step 4** is manual. The `.txt` file is organized by slide, so you work through it top to bottom.

### 0.2 Words you need to know

| Term | Plain-English meaning |
|---|---|
| **Resource / asset** | Anything in the customer's cloud: a VM, a storage bucket, a database, an IAM user, a load balancer… |
| **Workload / VM** | A virtual machine: AWS **EC2 instance**, Azure **Virtual Machine**, GCP **VM instance**, OCI **Instance** |
| **Inventory** | The full list of resources TotalCloud has discovered, counted per resource type |
| **Control (CID)** | One security rule, e.g. *"Ensure S3 buckets have versioning enabled"*. Each has a numeric **Control ID (CID)** |
| **Evaluation** | TotalCloud checking a resource against a control. The result is **PASS** or **FAIL** |
| **Failed resource** | A resource that failed **at least one** control |
| **Policy / framework** | A named group of controls, e.g. *CIS Amazon Web Services Foundations Benchmark* |
| **TruRisk score** | Qualys' risk score for a resource, **0–1000** (higher is riskier). Stored on each VM as `arsScore` / `truRiskScore` |
| **TruRisk Insight** | A pre-built "toxic combination" finding, e.g. *Publicly exposed VM with TruRisk score > 800*. Each has an **Insight ID** |
| **Connector** | The link between TotalCloud and one AWS **account**, Azure **subscription**, GCP **project** or OCI **tenancy** |
| **QID** | Qualys vulnerability ID. Severity **5** = critical |
| **CISA KEV** | US CISA's *Known Exploited Vulnerabilities* catalog: vulnerabilities attackers are actively using |
| **POD** | The Qualys data center the customer lives on (USPOD1, EUPOD2, INDPOD…) |
| **Look-back window (`days`)** | Most queries only count data updated or evaluated in the last *N* days (e.g. 15 or 30) |

### 0.3 Where the numbers live: the two data sources

Every input file comes from one of two places on the customer's POD:

1. **TotalCloud APIs** (the same APIs the TotalCloud UI uses). They answer questions like *"how many
   EC2 instances does this customer have?"* or *"which resources failed CID 47?"*. These files usually
   have `content` and `numberOfElements` in them.
2. **Elasticsearch (ES) indexes** (the database behind TotalCloud). These are queried directly for things
   the API doesn't offer, like *"average TruRisk score of all VMs"* or *"how many distinct resources
   failed a control"*. These files have `hits` and/or `aggregations` in them.

Every file name starts with the slide number of the old manual template (`03_…` = slide 3,
`07_02_…` = slide 7 storage/DB, and so on).

### 0.4 Rules used everywhere

| Rule | Detail |
|---|---|
| **Percentage** | `left number ÷ right number × 100` |
| **Rounding** | Round half up: 12.5 → 13, 12.4 → 12. (The code uses its own `round_off()`, not Python's `round()`.) |
| **Tiny percentages** | If the true % is above 0 but below 0.5, the report prints **`~1`** so a real risk isn't shown as 0% |
| **Cloud order** | Always AWS → Azure → GCP → OCI |
| **Missing file** | Treated as empty, so the number becomes **0**. The console prints `Error loading <file>` |
| **Number formatting on slides** | The report prints exact numbers (e.g. `2534`). Shortening to `2.5K` is done by hand when filling the deck |
| **Severity badges** | Labels like **Critical / High / Medium** on the slides are **not** calculated by the code. They're chosen by hand using the team's thresholds |

---

## Slides 1–2: Cover and agenda

| On the slide | Where it comes from |
|---|---|
| Customer name (e.g. "ACME CORP") | Typed in manually |
| Report date | Typed in manually (normally the date SRE collected the data, which is the date in the folder name) |
| *Report section: "Customer Information"* | Customer UUID, User UUID and username from the **Customer user lookup** (`03_01_01_userData.json`). Use them to confirm you have the right customer. They don't appear on the slide |

Slide 2 (agenda) is static text.

---

## Slide 3: Overall Cloud Security Status

Slide 3 is the most important slide. It has:

* **One gauge**: the Overall TruRisk Score, plus the overall status label (e.g. *"Immediate Attention"*)
* **Seven tiles**: CISA KEV, misconfigurations, active threats, public workloads, control failures,
  IAM misconfigurations, CIS failures

### How to read a tile


```
 ┌────────────────────────────────────┐
 │  20%                    [Critical] │  ← ① Percentage            ④ Severity badge
 │  500 / 2.5K                        │  ← ② Left number / ③ Right number
 │  Asset with misconfigurations      │  ← Tile title
 └────────────────────────────────────┘
```

| Part | Meaning | Source |
|---|---|---|
| ① **Percentage** | ② ÷ ③ × 100, rounded | Calculated by the code |
| ② **Left number** | How many are **bad** (failed / affected / exposed) | Calculated by the code |
| ③ **Right number** | How many there are **in total** (the population being measured) | Calculated by the code |
| ④ **Badge** | How serious the percentage is | **Manual.** Pick it using the team's thresholds |

The left number is always a **subset** of the right number, so the percentage must be between 0% and 100%.
Anything above 100% means something is wrong with the data.

Below, every tile follows the same layout:
**What it means → Left number → Right number → Percentage → Worked example → Where in the report → Watch-outs**.

---

### Data point 1: Overall TruRisk Score

**What it means:** the average risk of the customer's virtual machines on a 0–1000 scale. It is the big
number in the gauge.

#### Source

| Value | Title | What is asked | Field read |
|---|---|---|---|
| AWS score | **AWS VM average TruRisk score**<br>`03_01_01_aws_overall_tr_score_insightTypeFile.json` | ES index of AWS EC2 instances (`cloudview_instance`): *"For this customer, take every instance with a TruRisk score between 0 and 1000 and give me the **average**"* | `aggregations.average_arsScore.value` |
| Azure score | **Azure VM average TruRisk score**<br>`03_01_02_azure_overall_tr_score_insightTypeFile.json` | Same question on the Azure VM index (`cloudview_virtual_machine`) | same |

What the file looks like (trimmed):

```json
{
  "hits": { "total": { "value": 8518 } },          ← number of VMs that were averaged
  "aggregations": { "average_arsScore": { "value": 454.17 } }   ← the average we use
}
```

#### Calculation

```
AWS TruRisk     = round(AWS average)          e.g. 454.17 → 454
Azure TruRisk   = round(Azure average)        e.g. 371.6  → 372
Overall TruRisk = the HIGHER of the two       → 454
```

The **higher** value is used, not an average of the two, so the slide shows the risk of the
customer's worst cloud.

#### Worked example

| Cloud | VMs averaged | Average | Rounded |
|---|---|---|---|
| AWS | 8,518 | 454.17 | **454** |
| Azure | 40,112 | 371.60 | 372 |
| **Overall** | | max(454, 372) | **454** ← gauge value |

#### Where in the report

```
Slide 3: Overall Cloud Security Status
1. Overall TruRisk Score
1.1 AWS TruRisk Score: 454
1.2 Azure TruRisk Score: 372
1.3 Overall TruRisk Score: 454      ← put this in the gauge
```

#### Watch-outs

* **GCP and OCI are not included.** The script doesn't collect TruRisk scores for them. A GCP-only
  customer gets 0.
* This query has **no look-back window**. It averages every VM in the index for that customer, and VMs
  with a score of 0 pull the average down.
* The status label ("Critical", "Immediate Attention") is chosen **by hand** from the score.

---

### Data point 2: CISA known exploitable vulnerabilities

ACME example: **13% · 137 / 1.6K**

**What it means:** how many of the customer's VMs have at least one vulnerability that appears on the
CISA KEV list.

> ⚠️ **Not automated.** The generated report prints only the heading
> `4. VMs with CISA known exploitable vulnerabilities`. Fill this tile manually.

| Value | What we have today |
|---|---|
| Left number (VMs with a CISA KEV vulnerability) | The script collects the **CISA KEV QID list** (`03_04_01_CISA_known_vulnerability.json`): every QID in the Qualys knowledge base flagged `cisaKnownExploitedVulns: true`, with its CVSS base score and description. **This list is global, not customer-specific.** To get the customer's number it has to be matched against the customer's vulnerability data, and that step isn't coded |
| Right number (total VMs) | Not collected for this tile. Confirm with the team which VM total is used (the deck example uses a different total from the other tiles) |

**Action for a newcomer:** ask the report owner how this tile is currently filled before you send the deck.

---

### Data point 3: Assets with misconfigurations

ACME example: **20% · 500 / 2.5K**

**What it means:** out of all the resources TotalCloud checked, how many failed **at least one** security control.

There are **two ways** to get these numbers, and the code chooses one automatically.

#### Method A (preferred): Inventory counts

| Value | Title | What is asked | How it is computed |
|---|---|---|---|
| ② Left | **Failed resources, from Cloud Inventory**<br>`03_05_0x_<cloud>_inventory_contentTypeFile.json` | TotalCloud Inventory API: *"for each resource type, how many resources exist and how many failed a control?"* | Add up `failedResources` across every resource type, for all four clouds |
| ③ Right | **Total resources, from Cloud Inventory** (same file) | same | Add up `totalResources` across every resource type, for all four clouds |

The inventory file is a list with one row per resource type:

```json
"content": [
  { "resourceType": "EC2_INSTANCE", "totalResources": 6328, "failedResources": 1210 },
  { "resourceType": "BUCKET",       "totalResources": 2118, "failedResources": 640  },
  ...
]
```

#### Method B (fallback): Misconfiguration evaluation counts

| Value | Title | What is asked | Field read |
|---|---|---|---|
| ② Left | **Distinct resources that FAILED a control**<br>`03_02_0x_<cloud>_assets_misconfiguration_failed_evaluation_insightTypeFile.json` | ES control evaluation index: *"in the last N days, how many **distinct** resources have at least one evaluation with result = FAIL?"* | `aggregations.count.value` |
| ③ Right | **Distinct resources that were EVALUATED**<br>`03_02_0x_<cloud>_assets_misconfiguration_total_evaluation_insightTypeFile.json` | Same, but counting resources with **any** evaluation result | `aggregations.count.value` |

#### How the code chooses between A and B

```
For each cloud, read the inventory.
If ANY cloud has total > 0 but failed = 0   →  the inventory failed counts are probably missing
                                             →  use Method B for ALL clouds
Otherwise                                    →  use Method A for ALL clouds
```

Some inventory responses come back with `failedResources: 0` for every type even when resources clearly
fail controls. Without this check, the tile would show a misleading 0%.

#### Calculation

```
Left   = AWS failed + Azure failed + GCP failed + OCI failed
Right  = AWS total  + Azure total  + GCP total  + OCI total
%      = round(Left ÷ Right × 100)
```

#### Worked example (Method B)

| Cloud | Resources evaluated (right) | Resources failed (left) |
|---|---|---|
| AWS | 1,200 | 260 |
| Azure | 800 | 150 |
| GCP | 500 | 90 |
| OCI | 0 | 0 |
| **Total** | **2,500** | **500** |

`500 ÷ 2,500 × 100 = 20%` → the tile shows **20% · 500 / 2.5K**.

#### Where in the report

Method B prints one block per cloud, then the overall block:

```
2. Assets with Misconfigurations
2.1. AWS assets with misconfigurations
2.1.1. Total number of AWS resources evaluated for misconfiguration checks: 1200
2.1.2. Total number of AWS resources failed the misconfiguration checks: 260
... (2.2 Azure, 2.3 GCP, 2.4 OCI)
2.5. Overall Assets with Misconfiguration
2.5.1. Overall total number of resources evaluation for misconfiguration checks: 2500   ← right number
2.5.2. Overall total number of resources failed the misconfiguration checks: 500       ← left number
2.5.3. Percentage: 20                                                                  ← percentage
```

Method A prints only:

```
2.1.1. Overall total number of resources/ assets: 2500         ← right number
2.1.2. Overall total number of failed resources/ assets: 500   ← left number
```

In Method A **no percentage line is printed**. Work it out yourself: 500 ÷ 2500 × 100.

#### Watch-outs

* The ES "distinct count" is **approximate** above about 50,000 resources (Elasticsearch's cardinality
  limit). That's fine for a percentage, but don't quote it to the exact unit.
* Method B counts only resources **evaluated in the last N days**. Method A counts the whole inventory.

---

### Data point 4: Assets with active threats

ACME example: **12% · 300 / 2.5K**

**What it means:** how many VMs have an **active threat** (malware, suspicious behavior, etc.) detected on them.

| Value | Title | What is asked | Field read |
|---|---|---|---|
| ② Left | **AWS VMs with active threats**<br>`03_03_02_all_aws_active_threats_count_resourceCountTypeFile.json` | TotalCloud API: *"count EC2 instances updated in the last N days where `instance.hasThreats = true`"* | `data.count` |
| | **Azure VMs with active threats**<br>`03_03_03_all_azure_active_threats_count_resourceCountTypeFile.json` | Same, for Azure VMs (`virtualmachine.hasThreats = true`) | `data.count` |
| ③ Right | **Total AWS VMs** (Cloud Inventory, AWS) | Inventory row `EC2_INSTANCE` | `totalResources` |
| | **Total Azure VMs** (Cloud Inventory, Azure) | Inventory row `VIRTUAL_MACHINE` | `totalResources` |

The count file is small:

```json
{ "data": { "query": "updated: [now-15d ... now] and instance.hasThreats:true", "count": 12 } }
```

#### Calculation

```
Left   = AWS VMs with threats + Azure VMs with threats
Right  = AWS VMs + Azure VMs
%      = round(Left ÷ Right × 100)
```

#### Worked example

| Cloud | VMs with threats (left) | Total VMs (right) |
|---|---|---|
| AWS | 180 | 1,500 |
| Azure | 120 | 1,000 |
| **Total** | **300** | **2,500** |

`300 ÷ 2,500 = 12%` → **12% · 300 / 2.5K**

#### Where in the report

```
3.1.1. Total Number of AWS Assets with Active Threats: 180
3.1.2. Total Number of AWS VMs: 1500
3.2.1. Total Number of AZURE Assets with Active Threats: 120
3.2.2. Total Number of AZURE VMs: 1000
3.3.1. Overall Total Number of Assets with Active Threat: 300   ← left number
3.3.2. Overall Total Number of Assets: 2500                      ← right number (VMs only)
3.3.3. Percentage: 12
```

#### Watch-outs

* **AWS and Azure only.** GCP/OCI VMs are excluded from both numbers.
* Even though the label says "Assets", the right number counts **VMs only**.
* A result of 0 is common: threat detection needs the Qualys agent on the VMs.

---

### Data point 5: Publicly exposed workloads

ACME example: **23% · 375 / 1633**

**What it means:** how many VMs can be reached from the internet.

| Value | Title | What is asked | Field read |
|---|---|---|---|
| ② Left | **Publicly exposed VMs (TruRisk Insight 10)**<br>`03_05_05_public_vm_with_risk_score_insightTypeFile.json` (a copy of `03_03_01_…`) | ES TruRisk insight index: *"all resources that matched **Insight ID 10, public VM with risk score**, evaluated in the last N days and **not FIXED**"* | `hits.total.value` (the number of matching VMs) |
| ③ Right | **Total VMs, all clouds** (Cloud Inventory) | Inventory rows `EC2_INSTANCE` (AWS), `VIRTUAL_MACHINE` (Azure), `VM_INSTANCE` (GCP), `INSTANCE` (OCI) | `totalResources` for each, added together |

#### Calculation

```
Left   = number of public VMs (Insight 10)
Right  = AWS VMs + Azure VMs + GCP VMs + OCI VMs
%      = round(Left ÷ Right × 100)
```

#### Worked example

| | Count |
|---|---|
| AWS VMs | 900 |
| Azure VMs | 600 |
| GCP VMs | 133 |
| OCI VMs | 0 |
| **Right = total workloads** | **1,633** |
| **Left = public VMs (Insight 10)** | **375** |

`375 ÷ 1,633 = 22.96%` → rounds to **23% · 375 / 1633**

#### Where in the report

```
5. Publicly Exposed Workloads
5.1. Total Number of AWS VMs: 900
5.2. Total Number of Azure VMs: 600
5.3. Total Number of GCP VMs: 133
5.4. Total Number of OCI VMs: 0
5.5. Total Number of Public VMs: 375
5.6.1. Total Workloads: 1633          ← right number
5.6.2. Total Public Workloads: 375    ← left number
5.6.3. Percentage: 23
```

#### Watch-outs

* Insight 10 covers **AWS and Azure** VMs, but the right number includes GCP and OCI too, so a big GCP
  estate makes the percentage look lower.

---

### Data point 6: Cloud security control failures

ACME example: **24% · 179 / 745**

**What it means:** this tile counts **controls (rules), not resources**. Out of all the security
controls TotalCloud ran against the customer's cloud, how many failed for at least one resource?

| Value | Title | What is asked | Field read |
|---|---|---|---|
| ② Left | **Controls failed, per cloud**<br>`03_06_0x_tota[l]_<cloud>_controls_failed_contentTypeFile.json` | TotalCloud Evaluation API: *"list every control evaluated in the last N days that has **at least one FAIL**"* | `numberOfElements` (how many controls are in the list) |
| ③ Right | **Controls evaluated, per cloud**<br>`03_06_0x_total_<cloud>_controls_evaluated_against_contentTypeFile.json` | Same API: *"list every control evaluated in the last N days"* | `numberOfElements` |

One row of that list:

```json
{ "cid": 1, "controlName": "Ensure MFA is enabled for all IAM users that have a console password",
  "service": "IAM", "criticality": "HIGH", "successCount": 129, "failureCount": 58 }
```

#### Calculation

```
Left   = AWS failed controls + Azure + GCP + OCI
Right  = AWS evaluated controls + Azure + GCP + OCI
%      = round(Left ÷ Right × 100)
```

#### Worked example

| Cloud | Controls evaluated (right) | Controls failed (left) |
|---|---|---|
| AWS | 412 | 101 |
| Azure | 250 | 58 |
| GCP | 83 | 20 |
| OCI | 0 | 0 |
| **Total** | **745** | **179** |

`179 ÷ 745 = 24.03%` → **24% · 179 / 745**

#### Where in the report

```
6.1.1 Total number of controls evaluated for AWS Resources: 412
6.1.2 Total number of controls failed for AWS Resources: 101
... (6.2 Azure, 6.3 GCP, 6.4 OCI)
6.5.1. Total Number of Cloud Security Controls Evaluated: 745   ← right number
6.5.2. Total Number of Cloud Security Controls Failed: 179      ← left number
6.5.3. Percentage: 24
```

#### Watch-outs

* A control that fails on 1 resource and one that fails on 10,000 resources each count as **1**.

---

### Data point 7: IAM users & role access misconfigurations

ACME example: **16% · 64 / 397**

**What it means:** out of all identities (users and roles), how many failed at least one control.

All numbers come from the **Cloud Inventory** (same files as Data point 3, Method A). The code
reads specific rows:

| Cloud | "Users" row | "Roles" row |
|---|---|---|
| AWS | `IAM_USER` | `IAM_ROLE` |
| Azure | `USER` | `ROLE` |
| GCP | — | `SERVICE_ACCOUNT` (counted as a role) |
| OCI | `IAM_USER` | — |

| Value | Title | Computed as |
|---|---|---|
| ② Left | **Failed IAM users + failed IAM roles** | Sum of `failedResources` of the rows above |
| ③ Right | **Total IAM users + total IAM roles** | Sum of `totalResources` of the rows above |

#### Worked example

| Cloud | Users (total / failed) | Roles (total / failed) |
|---|---|---|
| AWS | 120 / 30 | 150 / 14 |
| Azure | 60 / 10 | 40 / 6 |
| GCP | — | 27 / 4 |
| OCI | 0 / 0 | — |
| **Total** | **180 / 40** | **217 / 24** |

Left = 40 + 24 = **64**, Right = 180 + 217 = **397**, `64 ÷ 397 = 16.1%` → **16% · 64 / 397**

#### Where in the report

```
7.1.1 … 7.4.2   per-cloud users/roles totals and failures
7.5.5 Total IAM users and roles count: 397          ← right number
7.5.6 Total failed IAM users and roles count: 64    ← left number
7.5.7 Percentage: 16
```

#### Watch-outs

* This relies on the inventory's `failedResources`. If it's 0 everywhere (see Data point 3), this tile
  also shows 0%. Check it against the IAM insights on slides 25–26.

---

### Data point 8: CIS framework controls failures

ACME example: **20% · 60 / 300**

**What it means:** the same idea as Data point 6, but **only** for controls in the official CIS benchmarks:

* AWS: *CIS Amazon Web Services Foundations Benchmark*
* Azure: *CIS Microsoft Azure Foundations Benchmark*
* GCP: *CIS Google Cloud Platform Foundation Benchmark*
* OCI: *CIS Oracle Cloud Infrastructure Foundation Benchmark*

| Value | Title | What is asked | Field read |
|---|---|---|---|
| ② Left | **CIS controls failed, per cloud**<br>`03_08_0x_total_<cloud>_cis_controls_failed_contentTypeFile.json` | Evaluation API filtered by `policy.name = <CIS benchmark>` **and** `result = FAIL`, last N days | `numberOfElements` |
| ③ Right | **CIS controls evaluated, per cloud**<br>`03_08_0x_total_<cloud>_cis_controls_evaluated_against_contentTypeFile.json` | Same, without the FAIL filter | `numberOfElements` |

#### Calculation and example

| Cloud | Evaluated | Failed |
|---|---|---|
| AWS | 62 | 22 |
| Azure | 72 | 14 |
| GCP | 166 | 24 |
| **Total** | **300** | **60** |

`60 ÷ 300 = 20%` → **20% · 60 / 300**

#### Where in the report

```
8.5.1. Total Number of CIS Framework Controls Evaluated: 300   ← right number
8.5.2. Total Number of CIS Framework Controls Failed: 60       ← left number
8.5.3. Percentage: 20
```

#### Watch-outs

* If the customer hasn't **enabled** the CIS policy in TotalCloud, both numbers are 0. That means the
  data wasn't collected, not that the customer is perfectly compliant.

---

### Slide 3 summary

| Tile | Left number | Right number | Clouds |
|---|---|---|---|
| TruRisk score (gauge) | — | — (max of AWS/Azure average) | AWS, Azure |
| CISA KEV | VMs with a CISA KEV vuln | total VMs | **manual** |
| Misconfigurations | resources that failed ≥ 1 control | resources evaluated | all 4 |
| Active threats | VMs with active threats | VMs | AWS, Azure |
| Public workloads | public VMs (Insight 10) | VMs | all 4 (right number) |
| Control failures | controls with ≥ 1 FAIL | controls evaluated | all 4 |
| IAM | failed users + roles | users + roles | all 4 |
| CIS failures | CIS controls with ≥ 1 FAIL | CIS controls evaluated | all 4 |

---

## Slides 5–11: Cloud Risk Summary

**What it shows:** one **card per cloud** with its size and risk. Slides 5–11 are layout variants. Keep
the one that matches the customer's connected clouds (one cloud, two clouds, or three clouds) and delete
the rest.

Example AWS card from the ACME deck:

```
 11261            Total Inventory
 11261 (42% at Risk)
 46 Cloud Accounts · 869 Critical vulns found · 51 CIS controls failed
 Risk Score – Medium   414
```

| Card item | Title / source | How it is computed | AWS | Azure | GCP | OCI |
|---|---|---|:-:|:-:|:-:|:-:|
| **Total Inventory** | Cloud Inventory for that cloud | Sum of `totalResources` across all resource types | ✔ | ✔ | ✔ | ✔ |
| **(% at Risk)** | Cloud Inventory for that cloud | Sum of `failedResources` ÷ Total Inventory × 100 | ✔ | ✔ | ✔ | ✔ |
| **Cloud Accounts / Subscriptions / Projects** | **Connector count**: AWS `05_01_03_total_aws_accounts.json`, Azure `05_02_03_azure_subscrition.json`, GCP `05_03_03_gcp_projects.json` (each file is just a number, e.g. `312`); OCI `05_04_03_…` (a list, so `numberOfElements` is used) | Number of connected accounts / subscriptions / projects | ✔ | ✔ | ✔ | ✔ |
| **Critical vulns found** | **Critical vulnerabilities grouped by QID**: AWS `05_01_04_aws_new_critical_vulnerability_…`, Azure `05_02_04_azure_critical_vulnerability_…` | Vulnerability API: *"severity-5 vulnerabilities on VMs in the last N days, grouped by QID"*. `numberOfElements` = **number of distinct critical vulnerabilities** (not the number of affected VMs) | ✔ | ✔ | — | — |
| **Risk Score** (number) | Same AWS/Azure TruRisk averages as [Data point 1](#data-point-1-overall-trurisk-score) | Rounded average | ✔ | ✔ | — | — |
| **Risk Score label** (Low/Medium/High) | — | **Manual** | | | | |
| **CIS controls failed** | Same as [Data point 8](#data-point-8-cis-framework-controls-failures) (left number, per cloud) | `numberOfElements` | ✔ | ✔ | ✔ | ✔ |

#### Worked example (AWS card)

| Item | Raw value | On the card |
|---|---|---|
| Sum of `totalResources` | 11,261 | **11261** Total Inventory |
| Sum of `failedResources` | 4,730 | 4730 ÷ 11261 = 42.0% → **(42% at Risk)** |
| Connector count | 46 | **46 Cloud Accounts** |
| Distinct severity-5 QIDs | 869 | **869 Critical vulns found** |
| AWS TruRisk average | 413.8 | **414** |
| AWS CIS controls failed | 51 | **51 CIS controls failed** |

#### Where in the report

```
Slide 5: Cloud Risk Summary
5.1. AWS Cloud Risk Summary
5.1.1. AWS Total Inventory Count: 11261
5.1.2. AWS Failed Resources Count: 4730
5.1.3. AWS Inventory at risk percentage: 42
5.1.4. AWS Cloud Accounts: 46
5.1.5. AWS Critical Vulnerabilities Found: 869
5.1.6. AWS Risk Score: 414
5.1.7. AWS CIS controls failed: 51
5.2. Azure … / 5.3. GCP … / 5.4. OCI …   (GCP/OCI have no vuln count or risk score lines)
```

> The headings 5.1, 5.2 here appear under **"Slide 5: Cloud Risk Summary"**. Don't confuse them
> with section **5. Publicly Exposed Workloads** in the Slide 3 block.

#### Watch-outs

* **% at Risk always uses the inventory** (there's no fallback here). If inventory `failedResources` is 0,
  this shows 0%. Compare it with Slide 3's misconfiguration tile.
* For multi-cloud layouts, the big "Total Inventory" in the middle is the **sum of all clouds** (see Slide 12).

---

## Slide 12: Cloud Inventory

**What it shows:** how big the customer's cloud is, split by resource category, with a total and a
per-cloud breakdown.

**Source for everything:** the **Cloud Inventory** of each cloud (`03_05_0x_<cloud>_inventory_…`). The code
adds up `totalResources` for the resource types listed under each tile:

| Tile on slide | AWS types | Azure types | GCP types | OCI types |
|---|---|---|---|---|
| **Storages** | BUCKET | STORAGE_CONTAINER | STORAGE | BUCKET |
| **Load Balancers** | LOAD_BALANCER | LOAD_BALANCER | LOAD_BALANCING | LOADBALANCER |
| **Database** | RDS | SQL_SERVER_DATABASE + COSMOS_DB | SQL + SPANNER_INSTANCE_DATABASES | DB_SYSTEMS + AUTONOMOUS_DATABASES |
| **IAM Roles** | IAM_ROLE | ROLE | SERVICE_ACCOUNT | — |
| **API Gateway** | API_GATEWAY | API_MANAGEMENT_SERVICE | — | API_GATEWAYS |
| **IAM Policies** | IAM_POLICY | — | — | — |
| **IAM Users** | IAM_USER | USER | — | IAM_USER |
| **Serverless** | LAMBDA | FUNCTION_APP | CLOUD_FUNCTION | FUNCTIONS_APPLICATIONS |
| **Clusters** | EKS_CLUSTER | KUBERNETES_CLUSTER | K8S_CLUSTER | KUBERNETES_CLUSTERS |
| **Containers** | EKS_NODEGROUP | — | K8S_NODE | — |
| **Workloads** | EC2_INSTANCE | VIRTUAL_MACHINE | VM_INSTANCE | INSTANCE |
| **Cloud Accounts** | *connector counts from Slide 5, not inventory* | | | |

| Other number on the slide | How it is computed |
|---|---|
| **Total Inventory** (e.g. 2524) | AWS total + Azure total + GCP total + OCI total, counting **every** resource type, not just the tiles |
| **Per-cloud totals** (e.g. 1.2K / 795 / 687) | That cloud's total inventory |

**Worked example (Storages):** AWS 120 buckets + Azure 50 containers + GCP 30 buckets + OCI 0 = **200 Storages**.

**Where in the report:** `Slide 6: Cloud Inventory`. `6.1.x` gives the per-cloud and overall totals,
`6.2.1 … 6.2.12` the tiles (per cloud and total), and `6.3.x` a one-line summary of every tile. `6.2.13 …
6.2.19` ("Extra Resources": VPCs, security groups, NACLs, internet gateways, WAF policies, container repos,
resource groups) are extras in case the slide needs them.

**Watch-out:** the tiles don't add up to Total Inventory. That's expected, because many resource types
(KMS keys, snapshots, queues…) have no tile.

---

## Slide 13: TruRisk Insights overview

**What it shows:** three tabs (**Workloads**, **Storage & Databases**, **IAM**). Each tab is a bar
list of the top insights with their counts, e.g. *"123 · Publicly exposed VM with TruRisk score > 800"*.

The bars are the **insight titles and totals** the code selects for slides 14–26:

| Tab | Report section | Max bars |
|---|---|---|
| Workloads | `7.1. Workload Insights` | 6 |
| Storage & Databases | `7.2. Storage and Database Insights` | 5 |
| IAM | `7.3. IAM Insights` | 3 |

Each entry in the report looks like `7.1.1. <Title>. (Total - 123)`. The title is the bar label and the
total is the bar number. How each list is picked is explained in the next three sections.

---

## Slides 14–17: Workload insights

**What they show:** slide 14/16 is a bar chart of up to 6 workload insights. Slides 15/17 are detail
tables: *"Top 5 out of N <insight>"* with **Impacted Workload, TruRisk Score, Additional Context**.

### Step 1: Try TruRisk Insights first (best findings)

The code walks through this **priority list** and keeps every insight that has affected resources, up to 6:

| Priority | Insight ID | Title |
|---|---|---|
| 1 | 1 | Publicly exposed VM with TruRisk score > 800 |
| 2 | 3 | Publicly exposed VM with confirmed vulnerability |
| 3 | 108 | VM with publicly exploitable vulnerability |
| 4 | 109 | VM with vulnerability that poses potential privilege escalation risk |
| 5 | 110 | VM with unauthenticated exploitable vulnerability |
| 6 | 106 | Publicly exposed VM with vulnerability associated to ransomware |
| 7 | 18 | Public VM with critical exploitable vuln and unencrypted EBS volumes |
| 8 | 17 | Publicly exposed VM with vulnerability detected in last 7 days |
| 9 | 16 | Publicly exposed VM with no encryption on EBS volumes |
| 10 | 15 | Publicly exposed VM with a critical exploitable vulnerability |
| 11 | 101 | Public VM with ransomware vuln and unencrypted EBS volumes |
| 12–17 | 111, 32, 33, 4, 5, 13 | Wormable vuln, data destructive permissions, EIP hijacking, SSH brute force, malware, RDP brute force |

For each insight, two sources are used:

| What | Title / source | Field |
|---|---|---|
| **Is it present, and the total (N)?** | **TruRisk Insights master list** (`07_00_TruRisk_Insights.json`): the TotalCloud API list of every insight with its affected-resource count | `totalAffectedResources` for that `id`. If it's missing or 0, the insight is **skipped** |
| **Which resources (Top 5)?** | **Insight detail hits** (`07_01_<id>_…_insightTypeFile.json`): ES TruRisk insight index, `insightId = <id>`, last N days, not FIXED | each hit's `resourceName`, `resourceId`, `truRiskScore`, `cloudIdentifier` (account/subscription), `region` |

The hits are **sorted by TruRisk score, highest first**, and the first 5 go in the table.

### Step 2: If fewer than 6 were found, fill up with CSPM controls

Up to 6 bars in total, in this order:

| Order | CID | Cloud | Title in report | Source title |
|---|---|---|---|---|
| 1 | 115 | AWS | VM with no encryption on EBS volumes | **EBS volumes attached to EC2 not encrypted** (`07_01_115_…`) |
| 2 | 50130 | Azure | VM with endpoint protection not installed | `07_01_50130_…` |
| 3 | 50037 | Azure | VM with end-to-end encryption using encryption at host disabled | `07_01_50037_…` |
| 4 | 52020 | GCP | VM with IP forwarding enabled | `07_01_52020_…` |
| 5 | 50314 | Azure | VM with trusted launch disabled | `07_01_50314_…` |
| 6 | 52091 | GCP | VM launched with shielded VM disabled | `07_01_52091_…` |
| 7 | 52028 | GCP | VM with connecting to serial ports enabled | `07_01_52028_…` |
| 8–10 | 40001, 40002, 40049 | OCI | Secure boot disabled / in-transit volume not encrypted / legacy metadata endpoint | `07_01_400xx_…` |

* All of these come from the **Evaluation API for one CID, FAIL results only**.
* **Total (N)** = `numberOfElements` (the true total of failing resources).
* **Resource name and TruRisk score:** control results don't contain a TruRisk score, so for AWS and
  Azure the code looks the VM up in the **All VM list** (`07_01_aws_all_vm_pg0…pg4_…`,
  `07_01_azure_all_vm_pg0…pg4_…`) by resource ID and uses its `name` and `arsScore`. If the VM isn't
  found, the score shows **NA**.
* **CID 115 is special:** it is evaluated per **EBS volume**, not per VM. The code reads the VM's
  ID from the evidence field *"Instance Id"* and removes duplicates, so one VM with three unencrypted
  volumes appears once.

### Step 3: Sort

All selected insights are sorted by **Total, largest first**. That order is the bar order on the slide.

### How the detail table is filled

```
7.1.1. Publicly exposed VM with TruRisk score > 800. (Total - 123)
	1. Resource Name: Billing Server, Resource ID: i-0b22a…, TruRisk Score: 945, Cloud Identifier: 1245…, Region: us-east-1
	...
```

| Table column | From the report line |
|---|---|
| Title "Top 5 out of **123** …" | `(Total - 123)` |
| **Impacted Workload** | `Resource Name` (top) + `Resource ID` (below) |
| **TruRisk Score** | `TruRisk Score` |
| **Additional Context** (ports like *RDP 3389*, *HTTPS 443*) | **Not in the report.** Look it up in the TotalCloud UI for that resource, or leave it out |
| **Contributing Factors / What's at stake / Qualys Recommends** | Template text for that insight type |

---

## Slide 18: Key Vulnerabilities

> ⚠️ **Not automated.** The data is collected but the code doesn't process it.

| Column | Source |
|---|---|
| **Security Vulnerability** (title + CVE) | **AWS key vulnerabilities by QID** (`12_01_aws_new_key_vulnerability_workloads_contentTypeFile.json`) and **Azure** (`12_02_azure_key_vulnerability_workloads_…`): `title`, `cveIds` |
| **Impacted Workloads** | Same file, field **`size`** = number of VMs that have this QID |
| **CVSS 3.1 Base Score / QDS Score / Description** | Not in these files. Look up the QID in the Qualys KnowledgeBase |

Usual approach: sort the rows by `size` (most impacted VMs first) and pick the top entries.

---

## Slides 19–24: Storage & Database insights

**What they show:** slide 19/22 is a bar list of up to 5 storage/DB insights. Slides 20–21 and 23–24 are
"Top N" tables with **Impacted Workload, Meta Data (account, region), Additional Context**.

### Step 1: Prepare the building blocks

| Building block | Source title(s) | Used for |
|---|---|---|
| **Public storage list** | AWS: **Publicly accessible S3 buckets** (`07_02_01_aws_publicly_accessible_S3_…`, `s3.isPubliclyAccessible = true`)<br>Azure: **Public blob containers**, CID 50012<br>GCP: **Public Cloud Storage buckets**, CID 52030<br>OCI: **Public Object Storage buckets**, CID 40003 | Insight 1, and checking whether a resource is public |
| **Public database list** | AWS: **Publicly accessible RDS** (`07_02_01_aws_publicly_accessible_DBs_…`, `rds.publiclyAccessible = true`, engine from `engine`)<br>GCP: **Cloud SQL MySQL open to the world**, CID 52033<br>OCI: **Autonomous DB accessible from everywhere**, CID 40058 | Insight 2, and checking whether a DB is public |
| **"Additional Info" tags** | AWS CID 48 → *Bucket Versioning Disabled*<br>GCP CID 52010 → *Object Versioning Disabled*<br>Azure CID 50181 → *TLS version outdated*<br>Azure CID 50198 → *Public Endpoint Connection* | Added to any storage row with the same resource ID. Rows with **more tags are listed first**, which gives the "critical findings" |

### Step 2: Pick up to 5 insights, in this order

| # | Insight title in report | Source | When it's used / special rule |
|---|---|---|---|
| 1 | **Publicly accessible storage resources** | Public storage list (all clouds) | If any exist. Total = sum over clouds. Top 5 **round-robin** (AWS, Azure, GCP, OCI, AWS…) so each cloud is shown |
| 2 | **Publicly accessible database resources** | Public DB list | If any exist. Round-robin |
| 3 | **Publicly exposed storage with cross account access** | TruRisk **Insight 67** (`07_02_67_…`) → `hits.total.value` | If it has hits |
| 4 | **Database instances with IAM authentication disabled**<br>or **Publicly exposed** database instances with IAM authentication disabled | AWS **CID 73** (engine from evidence *Engine*) | Uses the "Publicly exposed" version (public DBs ∩ CID 73) when there are **≥ 5** public ones **and** they're **≥ 30%** of all CID 73 failures |
| 5 | **Storage resources with access logging disabled**<br>or **Publicly exposed** storage … | AWS **CID 47** + GCP **CID 52031** | Uses the "Publicly exposed" version if **any** of them are also public |
| 6 | Auditing policy for databases not configured | Azure CID 50095 | |
| 7 | Storage access policy has wildcard in principal | AWS CID 46 | |
| 8 | Database workloads with no data encryption | Azure CID 50001 | |
| 9 | Database servers with Microsoft Entra authentication disabled | Azure CID 50035 | |
| 10 | Publicly accessible storage accounts | Azure CID 50461 | **Only if #1 found nothing** |
| 11 | Storage resources publicly accessible via ACL | AWS CID 45 | |
| 12 | Storage not encrypted using customer-managed keys | GCP CID 52108 | Public version by the same ≥ 5 and ≥ 30% rule |
| 13 | Database workloads allowing root login from any host | GCP CID 52035 | |
| 14 | Storage accounts with secure transfer disabled | Azure CID 50011 | |

The code stops at 5 insights. **Total** = `numberOfElements` of the control file (or the special totals
noted above). Finally the list is **sorted by Total, largest first**.

### How the table is filled

```
7.2.1. Publicly accessible storage resources (Total - 20)
	1. Resource ID: empinfo, Account ID: 124568795453, Region: us-east-2, Cloud Type: AWS, Additional Info: Bucket Versioning Disabled
```

| Table column | From the report line |
|---|---|
| Title "10 out of **20** …" | `(Total - 20)` |
| **Impacted Workload** | `Resource ID` (bucket / DB name) |
| **Meta Data** | `Account ID` + `Region` |
| **Additional Context** | `Additional Info` tags, and `Engine` for databases (e.g. *MYSQL Database*) |

Extra context on the ACME slide (e.g. *Block Public Access Disabled*, *Encryption – Google Managed Key*)
is added manually from the TotalCloud UI.

---

## Slides 25–26: IAM insights

**What they show:** a bar list of up to 3 IAM insights (slide 25) and a *"Top 5 out of N"* table (slide 26).

Insights are picked in this order until there are 3, then sorted by Total:

| # | Insight title | Source title | Total | Additional Context printed |
|---|---|---|---|---|
| 1 | IAM users with admin privileges and MFA disabled | TruRisk **Insight 41** (`07_03_41_…`) | `hits.total.value` | MFA Disabled, Admin Privileges |
| 2 | Custom Subscription Role with Admin Privileges | Azure **CID 50073** | `numberOfElements` | Admin Privileges |
| 3 | Service accounts with admin privileges | GCP **CID 52002** | `numberOfElements` | Admin Privileges |
| 4 | Root users with MFA disabled | AWS **CID 15** (resource shown as `Root`) | `numberOfElements` | MFA Disabled, Admin Privileges |
| 5 | IAM users with console access and MFA disabled | AWS **CID 1** + OCI **CID 40017** (round-robin) | AWS + OCI `numberOfElements` | MFA Disabled |

Table mapping: **Impacted Workload** = `Resource ID` (user/role name) and **Additional Context** =
`Additional Info`. Values like *"S3 Buckets, KMS"* (what the user can reach) on the ACME slide are manual.

---

## Slide 27: Risk Reduction Recommendations

Four boxes. Each sentence is generated from the **first rule that matches**.

### 01. Reducing TruRisk Score

Check these insights in order. The first one with at least 1 hit (`hits.total.value > 0`) is used:

| Order | Insight | Sentence printed |
|---|---|---|
| 1 | 3: Public VM with confirmed vulnerability | Patch **confirmed** vulnerabilities in **N** publicly exposed workloads to lower TruRisk score. |
| 2 | 15: Public VM with critical exploitable vuln | Patch **critical exploitable** vulnerabilities in **N** publicly exposed workloads… |
| 3 | 108: Publicly exploitable vuln on VM | Patch **publicly exploitable** vulnerabilities in **N** workloads… |
| 4 | 17: Public VM, vuln found in last 7 days | Patch vulnerabilities **detected in last 7 days** in **N** publicly exposed workloads… |
| none | — | Patch CISA vulnerabilities in **___** workloads… *(fill manually)* |

### 02. Enhancing Security

| Order | Condition | Sentence |
|---|---|---|
| 1 | Insight 18 or 16 has hits. Use whichever total is bigger (18 on a tie) | Secure **N** publicly exposed workloads (with critical exploitable vulnerabilities) using EBS encryption |
| 2 | else AWS CID 115 has ≤ 1000 failing volumes | Secure **N** AWS workloads with EBS encryption *(N = unique EC2 IDs from "Instance Id")* |
| 3 | else Azure CID 50130 > 0 | Secure **N** workloads by installing endpoint protection and improve CIS benchmark |
| 4 | else Azure CID 50037 > 0 | Secure **N** workloads with end-to-end encryption using encryption at host |
| none | | Nothing for Enhancing Security |

### 03. Optimizing Storage

| Condition | Sentence |
|---|---|
| Public storage total (slides 19–21, insight 1) **> 1** | Merge **N** storages into one public account. Apply least-privilege access; enable encryption and logging… |
| Public storage = 0, but Azure **public storage accounts** (CID 50461) > 0 | Merge **N** storage account into one public account… |
| otherwise | Nothing for Optimizing Storage |

### 04. Controlling Access

| Order | Condition | Sentence |
|---|---|---|
| 1 | Public databases > 0 | Update rules for **N** databases to allow access only from known network |
| 2 | AWS CID 73 (IAM DB auth disabled) > 0 | …only with approved IAM authentication |
| 3 | Azure CID 50035 (Entra auth disabled) > 0 | …only with approved Microsoft Entra authentication |
| none | | Nothing for Controlling Access |

**Where in the report:** `Slide 23. Risk Reduction Recommendations` → `01.` … `04.`

**Watch-out:** phrases like *"from 872 to 450"* or *"improve CIS benchmark by 20%"* on the ACME slide are
**not** calculated. Add them by hand only if you can justify them; otherwise remove them.

---

## Slide 28: Best Practices Recommendations

> ⚠️ **Not automated.** The report prints *"To-Do"*. The script already collects the inputs:

| Box on slide | Source title | How to get the number |
|---|---|---|
| **Limit access to resources** (workloads using default SGs with allow-all outbound) | Not collected yet | Manual, from the TotalCloud UI |
| **Identify sensitive data and secure it** (workloads / buckets / DBs with no encryption) | **Resources failing encryption controls**, per cloud (`24_01_01_aws_…`, `24_01_02_azure_…`, `24_01_03_gcp_resources_with_no_encryption.json`): ES failed evaluations whose control name contains "encrypt", grouped by service type | Read the resource count per `serviceType` bucket (EC2/VM → workloads, S3/Storage → buckets, RDS/SQL → DBs) |
| **Principle of least privilege** (IAM users & roles with high permissions) | **IAM users with high permission** (`22_03_IAM_user_with_high_permission_hitsTypeFile.json`) + **IAM roles with high permission** (`22_03_IAM_role_…`) | `hits.total.value` of each, added together |
| **Allow secure access** (admin users without MFA) | **Admin users with MFA disabled** (`20_01_adminPriviledges_MFA_disabled_contentTypeFile.json`, insight 5028) | `numberOfElements` |

---

## Appendix A: How the data is collected (SRE script)

SRE runs `script and data/script.sh` on a machine inside the customer's POD:

```bash
./script.sh <customerUuid> <POD> <username> <days>
```

| Argument | What to give | Why |
|---|---|---|
| `customerUuid` | The customer's TotalCloud UUID | Every query filters on it |
| `POD` | `USPOD1`–`4`, `EUPOD1`–`2`, `AEPOD`, `AUPOD`, `INDPOD`, `CAPOD`, `UKPOD`, `QAPOD4` | Selects the right Elasticsearch host and API gateway |
| `username` | Any user of that customer | Used to find the `userUuid` the APIs need, and to name the folder |
| `days` | e.g. `15` or `30` | The look-back window for nearly all queries |

What the script does:

1. Creates a folder `<username>_<YYYY-MM-DD>` and works inside it.
2. Looks up the user (`03_01_01_userData.json`) and extracts `userUuid`.
3. Runs about 190 queries (Elasticsearch searches and TotalCloud API calls), each saved to its own JSON file.
4. Copies some files under extra names (e.g. the inventory files appear as `03_05_…`, `05_0x_…`, `06_0x_…`) so each slide has "its own" file.

SRE zips that folder and sends it to you. You set `INPUT_FOLDER` in `src/config.py` to it and run `python main.py`.

---

## Appendix B: Data source index

The input files the code reads, with a readable title. `<c>` = `01` AWS, `02` Azure, `03` GCP, `04` OCI.

| Title | File | Used on |
|---|---|---|
| Customer user lookup | `03_01_01_userData.json` | Cover check |
| AWS / Azure VM average TruRisk score | `03_01_01_aws_…`, `03_01_02_azure_overall_tr_score_insightTypeFile.json` | Slide 3 gauge, Slide 5 |
| Distinct resources evaluated / failed (misconfig fallback) | `03_02_0<c>_<cloud>_assets_misconfiguration_{total,failed}_evaluation_insightTypeFile.json` | Slide 3 misconfig |
| AWS / Azure VMs with active threats | `03_03_02_…`, `03_03_03_…_active_threats_count_resourceCountTypeFile.json` | Slide 3 threats |
| Cloud Inventory (per cloud) | `03_05_0<c>_<cloud>_inventory_contentTypeFile.json` | Slides 3, 5, 12 |
| Publicly exposed VMs (Insight 10) | `03_05_05_public_vm_with_risk_score_insightTypeFile.json` | Slide 3 public workloads |
| Controls evaluated / failed | `03_06_0<c>_…_controls_{evaluated_against,failed}_contentTypeFile.json` | Slide 3 controls |
| CIS controls evaluated / failed | `03_08_0x_total_<cloud>_cis_controls_{evaluated_against,failed}_contentTypeFile.json` | Slide 3 CIS, Slide 5 |
| Connector count (accounts / subscriptions / projects / tenancies) | `05_01_03_…`, `05_02_03_…`, `05_03_03_…`, `05_04_03_…` | Slides 5, 12 |
| Critical vulnerabilities grouped by QID | `05_01_04_aws_new_critical_vulnerability_…`, `05_02_04_azure_critical_vulnerability_…` | Slide 5 |
| TruRisk Insights master list | `07_00_TruRisk_Insights.json` | Slides 13–17 |
| Workload insight hits | `07_01_<insightId>_…_insightTypeFile.json` | Slides 14–17, 27 |
| Workload control failures | `07_01_<CID>_…_contentTypeFile.json` | Slides 14–17, 27 |
| All AWS / Azure VMs (name + TruRisk lookup) | `07_01_{aws,azure}_all_vm_pg{0-4}_contentTypeFile.json` | Slides 15/17 |
| Public storage / public databases | `07_02_01_aws_publicly_accessible_{S3,DBs}_…`, `07_02_50012_…`, `07_02_52030_…`, `07_02_40003_…`, `07_02_52033_…`, `07_02_40008_oci_autonomous_database_public_…` | Slides 19–24, 27 |
| Storage tag sources | `07_02_48_…`, `07_02_52010_…`, `07_02_50181_…`, `07_02_50198_…` | Slides 20–21 |
| Storage/DB control failures | `07_02_<CID>_…_contentTypeFile.json` | Slides 19–24 |
| Cross-account public storage (Insight 67) | `07_02_67_…_insightTypeFile.json` | Slides 19–21 |
| IAM admin without MFA (Insight 41) | `07_03_41_…_insightTypeFile.json` | Slides 25–26 |
| IAM control failures | `07_03_<CID>_…`, `07_02_40017_oci_console_MFA_disabled_…` | Slides 25–26 |

Collected but **not yet used by the code**: `03_04_01` (CISA KEV list), `12_0x` (key vulnerabilities),
`14_0x` / `17_0x` (S3/RDS details), `20_01`, `22_03`, `24_01_0x` (best practices).

---

## Appendix C: Final checklist

1. ☐ The console run showed **no `Error loading` lines**, or you know why each one appeared (each one is a number that silently became 0).
2. ☐ The folder date and `days` window are what the customer expects.
3. ☐ Every percentage is between 0 and 100, and every left number ≤ right number.
4. ☐ The VM totals agree across Slide 3 (threats, public workloads), Slide 5 and Slide 12.
5. ☐ Manual items are filled: cover name/date, severity badges and status label, CISA tile, Slide 18,
   context columns, recommendation deltas, Slide 28.
6. ☐ Unused cloud-summary layouts (slides 5–11) are deleted.
7. ☐ The data folder and `.txt` report stay inside the team. They contain customer account IDs and resource names.
