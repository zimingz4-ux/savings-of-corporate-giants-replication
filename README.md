# The Savings of Corporate Giants: Python Replication

Python replication of Darmouni and Mota (2024, *Review of Financial Studies*), "The Savings of Corporate Giants." The authors' R pipeline is rebuilt in Python using Compustat, CRSP, and the authors' hand-collected 10-K portfolio data.

## Status

| Section | Output | Status |
|---|---|---|
| 4.3 COVID-19 liquidity shock | Figure 5, Table 3 | Done |
| 3.1–3.2 Portfolio composition | Table 1, Figures 1, 3, 4 | In progress |
| Extension: 2022–2023 rate hikes | | Planned |

## Key results

- **Figure 5 replicates.** Large firms built up cash in early 2020 while bond holdings stayed flat.
- **The main Table 3 result replicates.** A one standard deviation higher COVID-19 exposure is associated with cash-like holdings about 1.6% of total assets higher (paper: 1.3%), significant at 1%.
- **Securities effects are weaker than published**, and the corporate bond estimate changes sign when one firm (Booking Holdings) is dropped.


## How to run

1. Install packages: `pip install pandas pyarrow statsmodels matplotlib`
2. Get the data (not included, see below) and set the paths at the top of each notebook.
3. Run `covid_data_prepare.ipynb`, then `covid_shock.ipynb`.

**Data.** Compustat and CRSP require a WRDS subscription and cannot be shared. The hand-collected data are available from the [authors' website](https://www.corporategiants.net/). Download details are in [REPLICATION_NOTES.md](REPLICATION_NOTES.md).

## Repository structure

```
├── covid_data_prepare.ipynb    builds the quarterly sample (qdata)
├── covid_shock.ipynb           Figure 5 and Table 3
├── REPLICATION_NOTES.md        full log: data issues, diagnostics, comparison with the paper
├── figures/
└── README.md
```

## Reference

Darmouni, Olivier, and Lira Mota. 2024. "The Savings of Corporate Giants." *Review of Financial Studies* 37 (10). Paper on [SSRN](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3543802).
