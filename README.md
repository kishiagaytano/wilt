# Wilt

**Ranking web pages by how far their click-through rate falls below what their search position predicts.**

A page can rank well and still lose clicks. Wilt scores every page against the click rate actually observed for pages at its search position, then ranks the shortfalls. The output is a review queue with reason codes: which pages a content editor should open first when rewriting titles and meta descriptions, given they can review a few dozen per sprint. It is decision-support — it orders scarce attention. It does not decide anything, and it makes no claim that editing a page causes clicks to return.

Built on an anonymized slice of real search-performance data from the FlyRank ML internship. Work in progress — see [Current status](#current-status).

---

## Why this problem matters

A content team has thousands of published pages and capacity to review a few dozen. The question is not *"which pages are bad"* — it is *"which pages should a person open first."* Get that ordering wrong and the cost lands in two places:

- **A false positive costs editor hours.** Someone rewrites a title that was already fine, chasing a gap that was measurement noise. This is the expensive error, because it happens at the top of the queue where the scarce attention goes.
- **A false negative costs unrealised clicks.** A page that genuinely under-captures stays unfixed. Cheaper per instance, and invisible — nobody notices what the queue left out.

The errors are asymmetric, so the work optimises for precision at the top of the list and accepts lower recall.

The obvious rule — flag anything below a fixed CTR threshold — fails on this data. Measured on the starter slice at a 1,000-impression floor, `ctr < 0.5%` at positions 1–20 flags **59.6%** of eligible pages. A queue containing most of the inventory does not order anything. It fails because expected CTR depends on position: the observed rate spans roughly **14×** from the top-3 stratum to the deep stratum. One flat cutoff is simultaneously too lenient at the top of the results and impossible to meet at the bottom.

Position alone does not solve it either. Position tier explains about **7%** of the variance in per-page CTR; roughly **93%** of the spread sits *within* tiers. That within-tier spread is what this project ranks.

---

## Data

**In use today — the anonymized starter slice** (`data/raw/content_refresh_anonymized.csv`):

| Property | Value |
|---|---|
| Grain | one row per pseudonymized content item |
| Size | 30,000 rows × 44 columns |
| Clients | 32 pseudonymized |
| Window | one trailing 90-day window, aggregated |

It contains observed search and engagement measurements, content metadata, and transparent derived buckets. It contains no client names, domains, URLs, page titles, keywords, or raw queries. Identifiers are pseudonyms used for grouping and splitting only — never as features. FlyRank's own product decision flags and scores are deliberately absent from the release, so nothing here can learn the product's existing answer.

**Not yet used — the warehouse release** (`FlyRank/internship-warehouse`, gated). ~79M daily rows across `dim_clients`, `dim_content`, `fact_content_daily_performance`, and `fact_content_query_90d`, covering **2025-01-27 → 2026-06-30**. Required for two things this project needs and the starter slice cannot provide: rebuilding the position→CTR curve at scale, and forward-window validation. *In progress — no warehouse data has been read yet.*

**Exclusions, and why:**

- **`avg_position == 0` rows are dropped (1,205).** Zero means "no position data," not "ranked first." Keeping them would treat a missing measurement as the best possible rank.
- **A minimum-impressions floor is applied.** At a 500-impression floor the median page has 5 clicks and half have 5 or fewer, so per-page CTR is dominated by small-count variation. The exact floor is a policy choice being decided in ML-04, with the trade-off written down: a higher floor buys cleaner measurement and costs coverage.
- **`ctr` and `clicks_90d` are excluded as model features.** They are the components of the target itself. `impressions_90d` and `avg_position` remain legal — they are exposure and context, not outcome.
- **`trend_pct` and `trend_direction` are unused.** They are the label source for the reference pipeline's task, not this one.

Full column reference: [`docs/data-dictionary.md`](docs/data-dictionary.md). Handling rules: [`DATA_USE.md`](DATA_USE.md).

> One scale note that causes most misreadings of this data: rate columns are ×100 percentages. `ctr = 0.76` means 0.76%, not 76%.

---

## Method

Designed from the framing in [`w01`](work/notebooks/w01_research_question.ipynb) and [`w02`](work/notebooks/w02_ml_task_framing.ipynb). **Not yet built** — status per item below.

**Task type: ranking, not classification.** No column in this data records whether a page was reviewed or fixed. To classify, I would first have to invent a binary label and then train a model to reproduce it — a circular result that measures only how well a model copies my own rule.

**Target — a derived score, stated as such.** The inputs are observed: clicks and impressions are counted, position is measured. The expectation is estimated from the same observations — the impression-weighted click rate of every page in the same position stratum. The score is the shortfall against that expectation. It is a proxy I define, not an observed outcome, and it is labelled that way wherever it appears.

**Estimator: impression-weighted, not mean-of-ratios.** Averaging per-page CTR inverts the position ordering on this data — the top-3 stratum comes out *below* page-1, which is not credible. Weighting by impressions restores an ordering consistent with position. Mean-of-ratios over-weights tiny denominators, which is the same small-count problem that makes a volume floor necessary. *Measured; see `w01`.*

**Shrinkage.** A page with 1,000 impressions and zero clicks and a page with 50,000 impressions and zero clicks in the same stratum produce the same raw gap and must not receive the same score. Each estimate is shrunk toward its stratum rate in proportion to how thin its evidence is. *Designed — in progress (ML-07).*

**Validation: client-grouped holdout, not a random split.** Pages from one client share templates, topics, and site-wide characteristics. A random row split lets the same client appear in train and test, so a model can score well by recognising the client rather than by generalising. Whole clients are held out instead, which asks the question that matters: does this work on a client never seen before?

The size of that effect is measured rather than assumed. In [`notebooks/02`](notebooks/02_your_first_readable_model.ipynb) I re-ran a top-50 comparison across five client-holdout splits: the same method swung **0.44–0.68** depending only on which clients landed in the holdout — a wider spread than the gap between the two methods being compared. Single-split results on this data are not measurements. Everything gets reported as a range across repeated grouped splits.

**Leakage checks.** *Designed — in progress (ML-04, ML-05).* The list to verify: target components excluded from features; feature window never overlapping the target window; no rebuilt product flag entering as a feature; no derived field secretly encoding the target. The warehouse adds one specific trap — `fact_content_query_90d` covers a window that overlaps recent months, so for a label defined on the final month only the `*_prev30` columns are safe.

**Metric.** Precision@50 against a *forward-observed* outcome: of the top 50 pages flagged, how many show their click rate moving toward their stratum's expected rate over the following 30 days. K=50 because that is roughly a sprint of editor capacity. Supported by two checks — top-50 overlap across splits (an unstable queue is not actionable), and a hand review of the top 20. *Requires warehouse access — in progress (ML-09).*

### Results

<!-- PLACEHOLDER — my own model results go here once ML-07 through ML-09 are complete. -->

**No model of my own has been trained yet.** This section is intentionally empty rather than filled with the reference pipeline's numbers. Any figures published here will be my own, measured under the validation design above, reported as ranges across grouped splits.

---

## Current status

**Week 3. ML-04 (data contract) is next.**

Done:

- **ML-02 — research question and lane.** Lane chosen against measured evidence rather than descriptions, with three rejected alternatives and the numbers behind each. [`w01`](work/notebooks/w01_research_question.ipynb)
- **ML-03 — task framing.** Task type, target definition, success metric, unit of analysis, and why a fixed rule fails here. [`w02`](work/notebooks/w02_ml_task_framing.ipynb)
- **Exploratory work on the starter slice.** The position→CTR curve and the estimator bias, the click-count noise profile, the structural missingness of keyword data by content type, and the five-split instability result. [`notebooks/01`](notebooks/01_first_look_and_discovery.ipynb), [`notebooks/02`](notebooks/02_your_first_readable_model.ipynb)
- **Reference pipeline run end to end** for comparison, unmodified.

In progress / not started:

| Step | Status |
|---|---|
| ML-04 — data contract, volume floor decided and defended | Next |
| ML-05 — leakage audit | Not started |
| ML-06 — signal audit | Not started |
| ML-07 — expected-CTR baseline + shrinkage | Not started |
| ML-08 — model | Not started |
| ML-09 — validation, forward-window check | Not started |
| ML-10 — action playbook, reason codes | Not started |
| ML-11 — capstone write-up and paper | Not started |
| Warehouse access and the ~79M-row workflow | Not started |

---

## Repo structure

| Path | What it is |
|---|---|
| `work/notebooks/` | My assignment notebooks. `w01`, `w02` complete; the rest are skeletons. |
| `notebooks/01`, `notebooks/02` | Guided starter notebooks, run top to bottom, with my own experiments in the open cells. |
| `notebooks/03` | The DuckDB + Hugging Face workflow for the full release. Not yet run. |
| `scripts/01–05`, `run_all.py` | The reference pipeline — prepare, baseline, train, evaluate, report. Unmodified, kept as the comparison point. |
| `data/raw/` | The anonymized starter CSV. The only dataset in this repo. |
| `outputs/` | Reference pipeline artifacts. **Not my results.** |
| `docs/` | Data dictionary, lane guide, framework notes. |
| `skills/` | Instruction library for AI assistants working in this repo. |

---

## Reproducibility

The starter CSV ships with the repo. No credentials are needed for anything currently in it.

```bash
git clone https://github.com/kishiagaytano/wilt
cd wilt
pip install -r requirements.txt
python scripts/run_all.py
```

That runs the reference pipeline on the bundled sample and writes to `outputs/`. It is the baseline this project compares against — not my method.

The notebooks run top to bottom in Colab or a local Jupyter kernel:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/kishiagaytano/wilt/blob/main/work/notebooks/w01_research_question.ipynb?flush_cache=true) ML-02 — research question

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/kishiagaytano/wilt/blob/main/work/notebooks/w02_ml_task_framing.ipynb?flush_cache=true) ML-03 — task framing

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/kishiagaytano/wilt/blob/main/notebooks/01_first_look_and_discovery.ipynb?flush_cache=true) Starter — first look and discovery

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/kishiagaytano/wilt/blob/main/notebooks/02_your_first_readable_model.ipynb?flush_cache=true) Starter — readable model and leakage

Seeds are fixed and noted in each notebook. Datasets never enter git; CI fails any commit containing one.

Work on the full warehouse release requires gated access to `FlyRank/internship-warehouse` and a read token supplied via environment variable or Colab Secrets — never committed.

---

## Acknowledgments

Built on the FlyRank ML Internship dataset — [flyrank.ai](https://flyrank.ai).

Code is MIT licensed (see [`LICENSE`](LICENSE)). The data is governed by [`DATA_USE.md`](DATA_USE.md) and is not redistributable beyond the anonymized slice included here.
