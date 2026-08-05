# Soccer Transfer-Value Model

> Models what Europe's Big Five transfer market *pays* for a player against what that
> player actually *does* — and ranks the gap, to surface players the market underrates.

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-150458?style=flat&logo=pandas&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-F7931E?style=flat&logo=scikit-learn&logoColor=white)
![statsmodels](https://img.shields.io/badge/statsmodels-0b6b58?style=flat)
![seaborn](https://img.shields.io/badge/seaborn-4c72b0?style=flat)

📄 **[Full write-up — hypothesis testing, leakage guarding, and the undervalued-player ranking](https://hassamasghar.com/projects/soccer-transfer-value-model)**

---

## At a glance

| | |
|---|---|
| **Context** | MMA 860 — Management of Data, Queen's Smith School of Business |
| **My role** | Team lead — data assembly, hypothesis design, predictive modelling |
| **Problem type** | Regression + formal hypothesis testing |
| **Scale** | 4,056 player-seasons × 203 features, merged from 3 sources |
| **Headline result** | Ranked shortlist of undervalued players, each with a *reason* attached |

---

## The question

A forward who scores fifteen goals might be valued at €50M. A centre-back winning four
interceptions a game at an equally elite level for his position might be valued at €25M.

Some of that gap is rational — goals win games, scarcity is real. Some of it is
narrative: visibility, league prestige, age optics, agent noise. **If you can model the
rational part, the residual is a shopping list.**

## Approach

**Three sources, because no single one holds both sides of the question**

| Source | Contributes | Why it was necessary |
|---|---|---|
| FBref | Per-90 performance — shooting, passing, possession, defensive actions | The behavioural ground truth |
| Transfermarkt | Market valuations by season | The target variable |
| League financials | Revenue per league-season | Controls for richer leagues inflating every price inside them |

**Temporal split, not random.** 2021–22 and 2022–23 train, 2023–24 held out. Player
valuations are heavily autocorrelated year to year — a random split lets the model see a
player's later valuation while training on their earlier one, producing a beautiful score
and a useless model.

**Leakage guarding.** Explicitly dropped `market_value_eur` (raw target),
`highest_market_value_in_eur` (circular — encodes future valuations), `valuation_date`
and `contract_expiration` (post-prediction-point information).

**Log target.** Transfer values are heavily right-skewed; a handful of superstars sit
orders of magnitude above the median and would dominate the loss. Modelling
`log_market_value` makes residuals roughly homoscedastic and turns errors into something
closer to *percentage* error — which is how clubs actually reason about valuations.

**Multicollinearity.** 203 football statistics are enormously redundant
(`passes_completed` / `passes` / `passes_pct` are three views of one thing). VIF was run
**on training data only** — running it on the full set would leak test-season structure
into a preprocessing decision — and pruned iteratively.

**Four hypotheses tested before any prediction**, because a model that predicts well but
can't say *why* the market misprices anyone is not an investment thesis:

1. **League premium** — do Premier League players command more than comparable players elsewhere?
2. **Positional valuation gap** — are forwards valued above defenders after controls?
3. **Age–performance interaction** — does the market discount age for identical output?
4. **Contract length effect** — do longer remaining contracts raise valuation?

## Results

Top undervalued players in the held-out 2023–24 season:

| Player | Position | Actual | Model | Gap |
|---|---|---|---|---|
| Harry Kane | Centre-Forward | €100M | €257M | **+€158M** |
| Florian Wirtz | Att. Midfield | €130M | €263M | **+€133M** |
| Florian Lejeune | Centre-Back | €3M | €127M | **+€124M** |
| Jamal Musiala | Att. Midfield | €120M | €217M | **+€97M** |
| Martin Ødegaard | Att. Midfield | €110M | €205M | **+€95M** |

Lejeune is the extreme case — a centre-back whose defensive output the market appears to
almost entirely ignore. `model_results/` also contains the ranking run
position-by-position, so the shortlist isn't dominated by a single role.

## A bug worth keeping in the write-up

An earlier version of this ranking returned a defender with a predicted value of
**€1.55 × 10²⁹** — generously, more money than exists.

The model predicts in log space, and predictions were compared to actuals without
correctly back-transforming, so a modest error in logs became an astronomical error in
euros — the "most undervalued player" ranking was really sorting by *whose back-transform
blew up worst*.

The lesson isn't "check your maths." It's that **a result absurd on its face is a gift**,
because it announces itself. The dangerous version of this bug is the one that produces a
plausible-looking number.

## Repo contents

| File | What's in it |
|---|---|
| `MMA860_Soccer_Project.ipynb` | Full pipeline — merge, cleaning, feature engineering, VIF reduction, four hypothesis tests, valuation model |
| `model_results/top_5_overall_undervalued.csv` | Headline ranking — largest model-vs-market gaps |
| `model_results/top_5_position_undervalued2.csv` | Same ranking within each position |

## Running it

```bash
pip install pandas numpy scikit-learn statsmodels seaborn matplotlib rapidfuzz openpyxl
jupyter notebook MMA860_Soccer_Project.ipynb
```

`BASE_PATH` defaults to the repo directory. Note the raw FBref, Transfermarkt and League
Financials source files are **not** included here (size) — the notebook documents the
merge keys and cleaning steps needed to rebuild them, and `model_results/` contains the
final output so the findings are reproducible without them.

## Limitations

- **Market value ≠ transfer fee.** Transfermarkt valuations are crowd-informed estimates,
  not observed transactions — the model finds players the *market's own consensus*
  underrates.
- **Injuries aren't in the feature set**, and they explain part of why some players look cheap.
- **Three seasons is a short panel** for claims about structural market bias.

---

*Course project for MMA 860, Queen's Smith School of Business. I led the project — data
assembly, hypothesis design and the valuation model.*
