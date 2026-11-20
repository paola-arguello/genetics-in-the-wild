# D1: Which published model to adapt?

Nov 19, 2026 · @Pao

Decision 1 (D1) is to select which published cascade testing model to adapt as the base for a learning model in R. The recommendations are made by Claude (Opus 5.5):

|  | Tuffaha et al. 2018 | Teppala et al. 2024 |
| --- | --- | --- |
| Journal | Genet Med 20(9):985-994 | Value Health 27(11):1515-1527 |
| Proband | Women with breast cancer, at least 10% pretest probability | Men with metastatic prostate cancer |
| Relatives modelled | First- and second-degree | First-degree only |
| Model type | Decision tree + cohort Markov | Semi-Markov multi-state |
| Proband treatment in model | None beyond standard care (to confirm) | Likely olaparib (to confirm) |
| Horizon | Lifetime, to age 90 | Lifetime |
| Perspective | Health payer (Australia) | Payer (Australia) |
| Discount rate | 5% (confirmed) | ? (the companion olaparib paper used 5%) |
| Proband only vs no testing | AU$18,900/QALY | AU$265,942/QALY |
| Adding cascade testing | AU$9,500/QALY vs proband only | AU$16,392/QALY (comparator to confirm) |
| Full text access | PDF in hand (Nov 20) | Not checked |

## My decision

Both candidate models answer the same kind of question: does testing a *proband's* relatives add enough health for its cost?&#32;

### Tuffaha et al. 2018&#32;

Tuffaha, H. W., Mitchell, A., Ward, R. L., Connelly, L., Butler, J. R. G., Norris, S., & Scuffham, P. A. (2018). Cost-effectiveness analysis of germ-line BRCA testing in women with breast cancer and cascade testing in family members of mutation carriers. Genetics in Medicine, 20(9), 985–994. <https://doi.org/10.1038/gim.2017.231>

‌

### Paper's background&#32;

| Population | Family history (ovarian or breast) | Likelihood of having a BRCA1/2 mutation |
| --- | --- | --- |
| Women diagnosed with breast cancer  before 40 years | no | 10%&#32;&#32; |
|  | yes | 20% |

### General Assumptions:&#32;

1. Probands are women with unilateral breast cancer and a predicted mutation probability above 10%.
   1. Find which risk tool gave the 10% in the Methods.
2. Relatives of proband's with positive result get offered targeted testing for that variant
   1. *Each woman with breast cancer has 1.6 siblings (0.8 female, 0.8 male) and 2 children at risk.*
3. The model has two parts:&#32;
   1. decision tree to offer testing to relatives
   2. Markov model that follows people overtime&#32;
      1. cycle length is 1 year
      2. lifetime horizon to age 90
4. Possible states modeled:&#32;
   1. alive w/o cancer
   2. BRCA-positive
   3. has breast cancer&#32;
   4. has *metastatic* ovarian cancer
   5. death

\*People alive 5 years in a cancer state count as cured

5. The unit used to describe the health benefit of the testing is a combination of:
   1. the monetary cost of all genetic testing&#32;
      1. Canada genetic testing cost: [CAD $220 per BRCA1/BRCA2 test (2022 dollars, range ±30%) plus CAD $167 for genetic counselling.](https://jamanetwork.com/journals/jamanetworkopen/fullarticle/2823637)&#32;
   2. the Quality-Adjusted Life Year (QALY) for all family members tested&#32;

## Checked against the full text (Nov 20, 2026)

All five items are confirmed from the Tuffaha 2018 PDF. None are missing.

| Item | Answer | Where |
| --- | --- | --- |
| Risk tool behind the 10% | No single tool. Any of BOADICEA, BRCAPRO or the pathology-adjusted Manchester score above 10% | Methods, Model description (p. 2) |
| Source for 1.6 siblings and 2 children | Australian Institute of Family Studies: a 2.6-child first-generation household and a modern 2-child household (ref. 23, Hayes et al. 2010) | Methods (p. 2) |
| Discount rate | 5% a year for costs and QALYs | Analysis (p. 6) |
| Uptake | No uptake parameter, so testing uptake is 100% in the base case. Surgery: every carrier picks RRM alone (30%), RRBSO alone (54%) or both (16%). Table 3 tests 50% testing uptake and 80% surgery / 20% surveillance | Table 1 (p. 4), Table 3 (p. 8) |
| Table 1 | In hand. All values are in `parameters/master_parameters.csv` | pp. 4-5 |

All parameters with sources: [master_parameters.csv](parameters/master_parameters.csv). Build steps: [Tuffaha 2018 rebuild guide](<Tuffaha 2018 rebuild guide.md>).

## My contribution

### CANADA

Previous work

- Sun, L., Wei, X., Fierheller, C. T., Dawson, L., Oxley, S., Kalra, A., … Manchanda, R. (2024). Economic Evaluation of Population-Based BRCA1 and BRCA2 Testing in Canada. JAMA Network Open, 7(9), e2432725. <https://doi.org/10.1001/jamanetworkopen.2024.32725>

‌

In 2020, Canada had 393,000 Jews&#32;

- Congress, W. J. (n.d.). World Jewish Congress. Retrieved on November 9, 2026 from World Jewish Congress website: <https://www.worldjewishcongress.org/en/about/communities/ca>

### MEXICO

**Mutation yield**:&#32;

4.3% BRCA1/2 carriers among 810 *unselected* Mexican breast-cancer patients (62.8% of mutations recurrent; BRCA1 ex9–12del the most frequent), with only **\~15% of carriers having a first-degree relative with breast cancer** — an argument for testing all affected women rather than family-history-based selection, exactly the comparison Tuffaha modelled&#32;

- Torres-Mejía, G., Royer, R., Llacuachaqui, M., Akbari, M. R., Giuliano, A. R., Martínez-Matsushita, L., … Narod, S. A. (2015). Recurrent BRCA1 and BRCA2 Mutations in Mexican Women with Breast Cancer. *Cancer Epidemiology, Biomarkers & Prevention*, *24*(3), 498–505. <https://doi.org/10.1158/1055-9965.epi-13-0980>

  ‌

**Costs**

Direct medical cost of breast-cancer care in the Mexican public sector ≈ US 8,557/patient−year(current scenario), by stage US 6,500 (I) to US$12,757 (III) — usable for the cost side of the model.

- González-Robledo, M. C. (2015). Costs of breast cancer care in Mexico: analysis of two insurance coverage scenarios. *Ecancermedicalscience*, *9*. <https://doi.org/10.3332/ecancer.2015.587>

## Glossary

| Term | Meaning |
| --- | --- |
| Proband | The first person in a family to be tested, usually because they already have cancer. Also called the *index case*. |
| Cascade testing | Testing blood relatives of a carrier for the one known family variant. |
| QALY | Quality-adjusted life year: years lived × utility. |
| Cycle length | Time step of the model. |
| Time horizon | How long the model runs.&#32; "Lifetime" means until almost everyone has died. |
| ICER | Incremental cost-effectiveness ratio: extra cost ÷ extra QALYs between two strategies. |
| Cohort Markov model | Tracks the share of a group in each health state, cycle by cycle. Transitions depend only on the current state. |
| Decision tree | A branching diagram for one-off events, here the test and its result, before the Markov part starts. |
| Transition probability | Chance of moving from one state to another in one cycle. Not the same as a rate. Convert with p = 1 − exp(−rate × t). |
| First- / second-degree relative | **First**: parents, siblings, children (share about 50% of DNA).&#32; **Second**: grandparents, aunts and uncles, nieces and nephews, half-siblings (about 25%). |
| Deterministic / one-way sensitivity analysis | Change one input at a time across its plausible range and watch the ICER. Shown as a tornado diagram. |
| Probabilistic sensitivity analysis (PSA) | Draw all inputs from distributions many times at once to describe overall uncertainty. Shown as a cost-effectiveness acceptability curve. |
