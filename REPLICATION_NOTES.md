# Replication Notes: COVID-19 Liquidity Shock

## 1. Metadata

| | |
|---|---|
| **Name** | Replication of "The Savings of Corporate Giants" (Olivier Darmouni and Lira Mota, *Review of Financial Studies*, 2024) |
| **Link** | You can find the paper [here](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3543802) and the authors' hand-collected data [here](https://www.corporategiants.net/). |
| **Goal** | Replicate Section 4.3, "The 2020 Liquidity Shock: 'Cash' is Back": Figure 5 (aggregate portfolio dynamics, 2015–2021) and Table 3 (difference-in-differences regressions on COVID-19 exposure). |
| **Language** | Python (pandas, statsmodels, matplotlib). The authors' code is in R. |
| **Status** | Figure 5 replicated. Table 3: main results replicated; differences in columns 3–5 diagnosed (Section 4). |

---

## 2. Data preparation

### 2.1 Data access

I do not have programmatic WRDS access, so every WRDS table was downloaded manually through the WRDS Web Query interface instead of through the database connection used in the authors' scripts. Several issues in Section 3 come from this difference.

| File | WRDS path | Filters | Variables |
|---|---|---|---|
| `comp_names.csv` | Compustat – Capital IQ → North America – Daily → Fundamentals Annual | Entire date range | `gvkey`, `conm`, `sic`, `naics` |
| `comp_funda.csv` | Compustat – Capital IQ → North America – Daily → Fundamentals Annual | `datadate` ≥ 1990; INDL / STD / D / C | Identifiers (`gvkey`, `datadate`, `fyear`, `conm`, `fdate`, `cik`) and the 38 financial variables in the authors' `import_comp_funda.R`, including `at`, `che`, `ch`, `naicsh`, `sich` |
| `comp_fundq.csv` | Compustat – Capital IQ → North America – Daily → Fundamentals Quarterly | `datadate` ≥ 1990; INDL / STD / D / C | Identifiers (`gvkey`, `datadate`, `conm`, `datacqtr`, `fyearq`, `fqtr`, `datafqtr`, `fyr`, `cik`) and the 34 financial variables in `import_comp_fundq.R`, including `atq`, `cheq`, `chq`, `saleq` and the year-to-date flows `dltisy`, `dltry`, `dvy`, `prstkcy`, `sstky` |
| `crspq.ccmxpf_lnkhist.csv` | CRSP → Quarterly Update → CRSP/Compustat Merged → Link History | Link type LC, LU, LS | `gvkey`, `lpermno`, `lpermco`, `linkdt`, `linkenddt` |
| `crspq.stocknames.csv` | CRSP → Stock – Version 2 (CIZ) → Names | Entire date range | `permno`, `permco`, `secinfostartdt`, `secinfoenddt`, `siccd`, `ticker`, and default fields |

Non-WRDS inputs:

| File | Source |
|---|---|
| `fis_annual.parquet`, `fis_quarterly.parquet` | Authors' hand-collected 10-K data (annual 2000–2019, quarterly 2020Q1–2021Q4) |
| `employment-exposure.dta` | Chodorow-Reich, Darmouni, and Luck (2022); industry-level COVID-19 employment shock |

### 2.2 Directory structure

```
replication data collection/
├── data wrds/                     raw WRDS downloads (CSV)
├── fis/                           hand-collected FIS data (parquet)
├── raw/extra_data/                employment-exposure.dta
├── data/processed/compustat/      compa_clean.parquet, compq_clean.parquet
├── data/processed/final/          qdata.parquet
├── output/covid_shock/            tax_reform_covid_agg.png, reg_covid_DID.csv
├── covid_data_prepare.ipynb       data pipeline (steps 1–4 below)
└── covid_shock.ipynb              analysis (step 5 below)
```

Data folders are excluded from the repository because WRDS data cannot be redistributed.

### 2.3 Preprocessing steps

The authors' pipeline has more than ten R scripts. Only the parts that feed Section 4.3 were rebuilt.

| Step | Authors' script | What it does | Output |
|---|---|---|---|
| 1 | `create_compa_clean.R` | Clean Compustat annual; fill gaps in each firm's years; link to CRSP; fill missing SIC/NAICS codes from historical codes, CRSP, and the names table | `compa_clean` |
| 2 | `create_compq_clean.R` | Clean Compustat quarterly; merge industry codes; convert year-to-date flows to quarterly values; add calendar quarter | `compq_clean` |
| 3 | `create_fdata.R` (subset) | Merge annual FIS with the Compustat sample (positive assets and cash; no financials, utilities, or government); build five asset categories | `fdata` |
| 4 | `create_qdata.R` | Stack annual FIS (as fiscal Q4) with quarterly FIS; merge with Compustat quarterly for firms in the 2019 FIS sample | `qdata` |
| 5 | `covid_shock.Rmd` | Merge COVID exposure by 3-digit NAICS; align fiscal and calendar dates; balanced panel; Figure 5; Table 3 | figure, table |

Step 3 keeps only the parts needed for `qdata`. The bond spread, Fama-French industry, portfolio income, and benchmark return merges were skipped; they add columns but do not change the rows that `qdata` uses.

---

## 3. Replication process and records

### 3.1 Pipeline log

Key numbers printed while running `covid_data_prepare.ipynb`:

| Step | Check | Result |
|---|---|---|
| Annual | Duplicated (gvkey, fyear), removed | 11 |
| Annual | Rows with a valid CRSP PERMCO | 64.36% (authors' code comment: about 63%) |
| Annual | NAICS missing: raw → after all filling steps | 22.28% → 1.28% |
| Annual | SIC missing: raw → after all filling steps | 30.64% → 7.55% |
| Quarterly | Raw rows → clean rows | 1,691,508 → 1,131,862 (28,709 firms) |
| Quarterly | NAICS still missing | 0.59% |
| fdata | Firm-years / firms | 3,670 / 199 |
| qdata | Rows / firms | 3,687 / 156 |
| COVID | After merging COVID exposure | 3,495 rows / 150 firms |
| COVID | Balanced panel (10 time points) | 1,190 rows / 119 firms |

### 3.2 Problems encountered and how they were solved

**Problem 1. Web Query exports differ in format from database downloads.**
- Column names were partly upper case → converted all to lower case.
- `gvkey` was read as an integer, so `001690` became `1690` and would not match the FIS data → converted to text and padded to six digits.
- The CRSP link table includes all link types → kept LC, LU, LS as in the original query.
- `linkenddt` uses "E" for active links → parsed with `errors="coerce"`, so active links become missing, which the original code treats as "still valid".
- CRSP Version 2 names use `secinfostartdt` / `secinfoenddt` → renamed to `namedt` / `nameenddt`.
- `popsrc` is a query filter but not an exported column → verified instead that no `(gvkey, fyear)` pair is duplicated, which rules out mixed international records.

```python
def load(name):
    df = pd.read_csv(os.path.join(RAW, name), low_memory=False)
    df.columns = df.columns.str.lower()
    if "gvkey" in df.columns:
        df["gvkey"] = df["gvkey"].astype(str).str.zfill(6)
    return df
```

**Problem 2. The names table multiplied the data (the most serious issue).**

The authors use `comp.names`, which has one row per firm. It is not available through Web Query, so I took company fields from Fundamentals Annual, which has one row per firm-year. Merging it into the annual data duplicated firm-years, and merging the annual data into the quarterly data duplicated quarters. The pipeline stopped at the authors' primary key check ("Compustat data primary key violated"). Diagnostic:

```python
d_q = comp.duplicated(["gvkey", "cyqtr"], keep=False)
d_m = comp.duplicated(["gvkey", "mdate"], keep=False)
print("duplicated calendar quarter:", d_q.sum(), "| duplicated month:", d_m.sum())

print("rows in compq (raw):", len(compq))
print("rows in comp (now):", len(comp))
print("duplicated gvkey in names:", names["gvkey"].duplicated().sum())
print("duplicated (gvkey, fyear) in adata:", adata.duplicated(["gvkey", "fyear"]).sum())
```

```
duplicated calendar quarter: 8901746 | duplicated month: 8901746
rows in compq (raw): 1691508
rows in comp (now): 9307463
duplicated gvkey in names: 166748
duplicated (gvkey, fyear) in adata: 3106570
```

The equal counts for the two duplicate checks showed that whole rows were being copied, not that a few firms had conflicting dates. The quarterly file grew from 1.7 million to 9.3 million rows.

*Fix:* keep one row per firm before merging, matching the structure of `comp.names`. After the fix, all duplicate counts are zero and the primary key check passes.

```python
names = names.drop_duplicates("gvkey", keep="last")
```

**Problem 3. Duplicated quarterly records.** The raw quarterly file has 962 duplicated `(gvkey, datadate)` pairs, mostly from firms changing their fiscal year end. All of them disappear when rows with a missing calendar quarter (`datacqtr`) are dropped, which is consistent with Compustat leaving `datacqtr` blank when a calendar quarter cannot be assigned uniquely. No change to the original logic was needed.

**Problem 4. Industry codes in different formats.** NAICS is numeric in Compustat but must be cut to three characters to merge with the exposure file. Converting a float such as `325412.0` directly to text gives wrong prefixes, so codes are first converted to integers. In R, the same merge fails outright if one side is numeric and the other text, which suggests the exposure file stores `naics_code` as text.

```python
def to_code(s):
    return pd.to_numeric(s).round().astype("Int64").astype("string")
```

**Problem 5. Smaller engineering issues.**
- A local helper module named `utils` clashed with an installed package of the same name. The few helper functions were moved into the notebooks.
- The FIS `cik` column mixes numbers (FIS) and text (Compustat), which parquet cannot store; it is stored as text. It is not used in the analysis.

### 3.3 Results

**Figure 5.** The replicated figure matches the published Figure 5.

![Figure 5 replication: aggregate portfolio dynamics, 2015–2021](figures/figure5_replication.png)

Aggregates behind the figure (trillion USD):

| Quarter | Cash-like | Corporate bonds | U.S. government | Total FA |
|---|---|---|---|---|
| 2015Q4 | 0.457 | 0.258 | 0.263 | 1.158 |
| 2016Q4 | 0.474 | 0.301 | 0.321 | 1.263 |
| 2017Q4 | 0.567 | 0.340 | 0.343 | 1.396 |
| 2018Q4 | 0.435 | 0.284 | 0.320 | 1.175 |
| 2019Q4 | 0.558 | 0.215 | 0.270 | 1.181 |
| 2020Q1 | 0.662 | 0.211 | 0.266 | 1.266 |
| 2020Q2 | 0.733 | 0.216 | 0.302 | 1.394 |
| 2020Q3 | 0.720 | 0.219 | 0.321 | 1.413 |
| 2020Q4 | 0.697 | 0.233 | 0.303 | 1.398 |
| 2021Q4 | 0.670 | 0.255 | 0.281 | 1.405 |

The figure shows both stories in the paper:
- **After the 2017 tax reform:** total financial assets fell from 1.40 to 1.17 trillion USD by 2018Q4, and corporate bonds fell from 0.34 to 0.21 trillion by 2019Q4.
- **During the COVID-19 shock:** cash-like holdings rose from 0.56 to 0.73 trillion between 2019Q4 and 2020Q2, while corporate and government bonds barely moved. Almost all new financial assets were held as cash.

**Table 3.**

| Outcome | Paper | Replication | R² (paper / replication) |
|---|---|---|---|
| Total FA / AT | 0.019*** (0.005) | 0.018*** (0.005) | .918 / .918 |
| Cash-like / AT | 0.013*** (0.004) | 0.016*** (0.004) | .757 / .763 |
| Marketable securities / AT | 0.006** (0.003) | 0.002 (0.003) | .945 / .947 |
| U.S. government bonds / AT | 0.003** (0.001) | 0.002* (0.001) | .979 / .979 |
| Corporate bonds / AT | 0.003 (0.002) | −0.001 (0.002) | .858 / .859 |
| Observations | 720 | 714 | |

The main result replicates. Firms more exposed to the shock increased financial assets and cash-like holdings relative to total assets, with the same sign, similar size, and the same significance level. Exposure is a standardized (z-score) industry measure, so a one standard deviation higher exposure is associated with cash-like holdings about 1.6% of total assets higher. R² values are almost identical, so the underlying data are very close.

---

## 4. Differences from the original and diagnostics

### 4.1 Where Table 3 differs

The differences are in columns 3–5. The replication assigns slightly more of the increase to cash and less to securities: column 2 is 0.003 higher and column 3 is 0.004 lower. The corporate bond coefficient changes sign, but it is insignificant in both versions. The sample has 714 observations instead of 720, which is 119 firms instead of 120 over six quarters. Diagnostics 4.2 and 4.3 look for the cause.

### 4.2 Sample reconciliation

Each step where firms leave the sample was traced.

| Step | Firms | Reason for losses | Same as authors? |
|---|---|---|---|
| FIS sample | 200 | | |
| Still in Compustat in 2019 | 158 | 42 firms delisted, acquired, or merged before 2019 | Yes |
| Pass industry filters | 156 | Honeywell (SIC 9997) and PayPal (NAICS 522320) excluded | To verify |
| Matched to COVID exposure | 150 | 6 firms in NAICS 312 or 316 | Very likely |
| Balanced panel | 119 | 29 firms miss one or more of the 10 time points | Yes (structural) |

**Diagnostic A: firms that never reach `qdata`.**

```python
# Trace where firms in the FIS sample drop out of the pipeline:
# count FIS firms that reach fdata and qdata, then show the 2019 Compustat
# records (assets, cash, industry codes) of firms that never reach qdata
fis19 = set(fis_annual.loc[fis_annual["year"].astype(str) == "2019", "gvkey"])
print("FIS firms in 2019:", len(fis19))
print("  in fdata:", len(fis19 & set(fdata["gvkey"])))
print("  in qdata:", len(fis19 & set(qdata["gvkey"])))

lost = fis19 - set(qdata["gvkey"])
print(adata.loc[adata["gvkey"].isin(lost) & (adata["fyear"] == 2019),
                ["gvkey", "conm", "at", "che", "sic", "naics"]].to_string())
```

```
FIS firms in 2019: 200
  in fdata: 199
  in qdata: 156
           gvkey                         conm      at      che   sic   naics
287293    001300  HONEYWELL INTERNATIONAL INC  58679.0  10416.0  9997  999977
394583    024616          PAYPAL HOLDINGS INC  51333.0  10761.0  7389  522320
```

```python
# For firms missing from qdata, check whether they still exist in Compustat:
# a last year before 2019 means the firm was delisted, acquired, or merged
# before the COVID sample period, so its exclusion is expected
chk = adata[adata["gvkey"].isin(lost)].groupby("gvkey").agg(
    conm=("conm", "last"), first_year=("fyear", "min"), last_year=("fyear", "max"))
print(chk.sort_values("last_year").to_string())

# Firms in the FIS sample with no Compustat annual record at all would point to a download or gvkey problem
print("not in Compustat annual at all:", sorted(lost - set(adata["gvkey"])))
```

Result: 42 firms have a last Compustat year before 2019, from Enron (2000), Texaco, and Compaq to Twenty-First Century Fox (2018). No FIS firm is missing from Compustat altogether, so the download has no gaps. The FIS file has one row per firm and year for all 200 firms, including years after a firm disappeared, so "200 firms in 2019" counts sample firms, not active firms.

Findings:
- **Honeywell is excluded by the government filter.** The code drops firms whose SIC code starts with 9 to remove public administration (SIC 91–97). Compustat uses SIC 9997 for conglomerates, so Honeywell, a large industrial firm with substantial financial assets, is also excluded. This looks like an unintended side effect of the filter.
- **PayPal is excluded as a financial firm** because its NAICS code starts with 52. This is consistent with the stated sample design.

**Diagnostic B: firms lost in the exposure merge.** Six firms do not match the exposure file: Altria, Philip Morris International, Coca-Cola Europacific Partners, Molson Coors, Constellation Brands (NAICS 312, beverages and tobacco), and Nike (NAICS 316, leather products). The likely reason is that the exposure file has no entries for these two industries, which would affect the authors' sample the same way. *To verify:* `covid_exp["naics"].isin(["312", "316"]).any()` should return `False`.

**Diagnostic C: firms lost when building the balanced panel.**

```python
# For firms dropped from the balanced panel, list which of the 10 time points they miss,
# together with their fiscal year-end month, to tell structural losses
# (non-December fiscal years, mergers, late listings) from data problems
all_points = sorted(cdata["datacqtr"].astype(str).unique())
lost = cdata_all[cdata_all["nobs"] != 10]
missing = lost.groupby("gvkey").agg(
    conm=("conm", "first"),
    fy_end_month=("datadate", lambda d: d.dt.month.mode()[0]),
    have=("datacqtr", lambda q: set(q.astype(str))),
)
missing["missing"] = missing["have"].apply(lambda h: [p for p in all_points if p not in h])
print(missing[["conm", "fy_end_month", "missing"]].to_string())
```

All 29 firms fall into groups with structural explanations:

| Group | Firms | Missing points |
|---|---|---|
| Fiscal year ends in January or March | Best Buy, Target, Macy's, Home Depot, Kroger, Lowe's, Walmart, Salesforce, VMware, DXC Technology, McKesson | 2019Q4 only |
| Fiscal year ends in April or May | Conagra, General Mills, FedEx, Oracle, Medtronic | Four points |
| Merged or acquired around 2020 | Raytheon, Noble Energy, Caesars Entertainment, Viacom, Old Copper (J.C. Penney) | 2020–2021 |
| Did not exist or was not public in 2015–2016 | Baker Hughes (post-2017 entity), Dell Technologies | Early points |
| No 2020–2021 quarterly data | Burlington Northern Santa Fe, Johnson Controls, Weyerhaeuser, Aptiv, Liberty Global | 2020–2021 |
| Unexplained | Cheniere Energy | 2015Q4, 2016Q4 |

For firms with a January fiscal year end, the FY2019 annual report is dated January 2020 and is assigned to 2020Q1, while the quarterly FIS data start in fiscal 2020. The 2019Q4 point can therefore never be filled. The authors' code comment says the fiscal-year threshold was moved "for Feb, due covid", but the balanced-panel requirement still excludes every large retailer with a January fiscal year end.

**The missing 120th firm.** Since every loss above follows from the code's logic, the gap of one firm most likely comes from industry codes that differ between Compustat vintages. Candidates: Honeywell, PayPal, and Cheniere Energy. *To verify:* check whether these firms appear in the paper's firm list (Appendix Table A.3) and with which SIC code.

### 4.3 Leave-one-out check on corporate bonds

```python
# Leave-one-out check for the corporate bond regression (Table 3, column 5):
# re-estimate the coefficient dropping one firm at a time and list the firms
# whose removal moves it the most, to see whether a single firm drives the sign
rows = []
for f in reg_data["gvkey"].unique():
    m = smf.ols("corp_at ~ post:covid_exp + C(datacqtr) + C(gvkey)", data=reg_data[reg_data["gvkey"] != f]).fit()
    rows.append((f, m.params["post:covid_exp"]))

loo = pd.DataFrame(rows, columns=["gvkey", "coef_without"]).sort_values("coef_without")
loo = loo.merge(cdata.groupby("gvkey")["conm"].first().reset_index(), on="gvkey")
print(loo.head(5))
print(loo.tail(5))
```

```
    gvkey  coef_without                 conm
0  020779     -0.001216    CISCO SYSTEMS INC
1  001690     -0.000841            APPLE INC
2  161844     -0.000825 LAS VEGAS SANDS CORP
3  028930     -0.000825    MARRIOTT INTL INC
4  014418     -0.000825 MGM RESORTS INTERNATIONAL
...
117  024800      0.000397          QUALCOMM INC
118  119314      0.001919 BOOKING HOLDINGS INC
```

Dropping Booking Holdings alone moves the coefficient from −0.001 to +0.0019, close to the paper's 0.003. Dropping any other firm except Qualcomm leaves it negative. Booking, an online travel platform, was among the firms most exposed to the shock and held a large corporate bond portfolio. The column 5 result is therefore fragile: one firm determines its sign, in the replication and possibly in the paper.

### 4.4 Notes on the original code and paper

1. **Table note and code disagree on the sample window.** The table note says Post covers 2020Q1–2021Q4 and that the sample runs to 2021Q4. The code restricts the regression to 2018Q4–2020Q4, and 720 = 120 × 6 matches the code's six time points, so the published results follow the code.
2. **Firm counts differ between comment and table.** The code comment on Figure 5 refers to 141 firms; the regression sample has 120. The published numbers may come from a slightly different data version.
3. **Units.** Outcome ratios are not multiplied by 100, although the column headers read "(%)". Coefficients are therefore shares of total assets.
4. **Standard errors** are not clustered.
5. **Timing rule.** Calendar 2021Q1 is mapped to 2020Q4 and the latest observation is kept, so for December fiscal year firms the 2020Q4 point uses March 2021 data.
6. **Sample selection** (Section 4.2): the SIC 9 filter removes conglomerates, and the balanced panel removes retailers with January fiscal years and firms that merged around 2020.

### 4.5 Changes made to the original pipeline

None of these change the logic of the analysis.

| Change | Reason |
|---|---|
| Names table de-duplicated to one row per firm | Web Query has no `comp.names` (Problem 2) |
| CRSP Version 2 date columns renamed | Different CRSP file version |
| Quarterly cleaning keeps only variables used downstream; earnings volatility (`sale_vol`) skipped | Not used in Section 4.3 |
| Minimal `fdata` (sample filters and asset categories only) | Bond, industry, and income merges not needed for `qdata` |
| OLS with firm and quarter dummies (`statsmodels`) instead of `lfe::felm` | Same coefficients and standard errors; verified on simulated data against the authors' R code |

---

## 5. Summary and next steps

**Summary.** The Python pipeline reproduces Figure 5 and the main Table 3 results: COVID-exposed firms built up cash, not securities. Every step of sample construction was reconciled against the authors' code, and the remaining one-firm gap is traced to likely differences in Compustat industry codes. The weaker securities results come from fragile, insignificant coefficients; column 5 changes sign when a single firm (Booking Holdings) is removed.

**Next steps.**
1. Check Appendix Table A.3 for Honeywell, PayPal, and Cheniere Energy to identify the 120th firm.
2. Confirm that NAICS 312 and 316 are absent from the exposure file.
3. Run the leave-one-out check for marketable securities (column 3), where the paper reports 0.006** and the replication 0.002.
4. Robustness: cluster standard errors by firm; use December 2020 instead of March 2021 for the 2020Q4 point; report outcomes in percent.
5. Replicate the remaining sections: Table 1 and the portfolio composition figures (Section 3).
6. Extension: the 2022–2023 rate-hike cycle. The paper's sample ends in 2021. Using SEC EDGAR filings, measure unrealized losses on these firms' bond portfolios and test whether firms with higher interest-rate exposure at the end of 2021 rebalanced toward cash, using the same design as Table 3.
