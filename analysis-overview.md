# Tuffaha 2018 rebuild: step-by-step guide (R)

Nov 20, 2026 · @Pao

## GOALS
1. To rebuild Tuffaha 2018 in R with Australian inputs
   Completed after step 6
   
3. Swap in Canadian values and compare to available literature
   Completed after step 8
   
5. Swap in Mexican values (approximations when not available)
   Completed after step 9
   
Every number comes from [parameters/master_parameters.csv](parameters/master_parameters.csv).
  
## Prior to building the model
Decisions should be made on the following fronts: 

- [ ] Half-cycle correction: none (as Tuffaha appears to do) or mid-cycle. Mid-cycle is the more common choice, but replication is easier without it. You could match Tuffaha first, then switch
- [ ] Surgery: keep Tuffaha's base case (every carrier has surgery) and also run their 80% surgery / 20% surveillance scenario?
- [ ] Second-degree relatives: include them as in Tuffaha, or first-degree only for a simpler first version
- [ ] Canada: keep Tuffaha's surgery uptake and swap only costs and discounting, or use Hurry's 44% and 21%
- [ ] Mexico: number of arms (3, or 4 with testing all women with breast cancer) and which payer
- [ ] Disutility for risk-reducing surgery: none (Tuffaha) or applied (Hurry)
- [ ] Where the model lives: its own repo or inside the pipeline repo

## Documentation & Reproducibility

- **Parameters:** Every number lives in one parameter file
- **Master function**: Calls on parameters file and runs the base case, one-way analysis and probabilistic analysis

## Step 1 

1. Export the Master sheet's values into `data/params_aus.csv` with these columns: `name, value, low, high, distribution, source, country, status`. Start with Australia only.
2. Write `R/load_params.R`: reads the CSV and returns a named list.
3. Write `R/run_model.R`: `run_model(params)` returns a list with `cost`, `qaly`, `ly`, `bc_events`, `oc_events` for each strategy.
4. Add `tests/testthat/` from the start. The first tests are the structural checks in Step 3.

Suggested layout:

```
cascade-cea/
  data/params_aus.csv
  R/load_params.R
  R/family.R        # Step 2
  R/markov.R        # Step 3
  R/outcomes.R      # Steps 4-5
  R/sensitivity.R   # Step 7
  tests/testthat/
  README.md         # question, decisions, limitations, label
```

Use base R matrices for the core. They are the easiest to explain in an interview. heemod or dampack can come later for plots.

## Step 2: Family arithmetic (the decision tree front end)

The decision tree does not follow people over time. It works out how many relatives are tested, how many are carriers, and at what age they enter the Markov part. Those counts become weights.

Per proband who tests positive, using Tuffaha's family structure:

| Relative | Number | Chance carrier | Expected carriers | Enter Markov? | Start age |
| --- | --- | --- | --- | --- | --- |
| Sisters | 0.8 | 0.5 | 0.4 | Yes | 40 |
| Brothers | 0.8 | 0.5 | 0.4 | No (test cost only) | 40 |
| Daughters | 1.0 | 0.5 | 0.5 | Yes | 10 (tested at 20, act at 30) |
| Sons | 1.0 | 0.5 | 0.5 | No | 10 |
| Nieces and nephews (children of the 0.8 siblings who test positive) | 1.6 (0.8 F, 0.8 M) | 0.5 | 0.4 F | Nieces only | 10, as "female children" |

The paper confirms this structure. Figure 1a weights the female Markov cohorts as proband 1, female siblings 0.8, female children 1.8 (one daughter plus 0.8 nieces). The 25% for second-degree relatives is not a separate input: it is 50% of siblings being carriers times 50% of their children inheriting.

Two things to state in your methods:

- Men are tested and cost money, but only women enter the Markov part. Men matter because they can pass the variant to their children.
- In the no-testing arm the same relatives exist and some are carriers. They just never learn it, so they follow carrier risk with no surgery. The benefit of cascade testing comes from carriers who then choose surgery. Non-carriers have the same outcomes in both arms, apart from the test cost.

In R, keep this as one small function that returns a data frame with one row per relative type (`n`, `p_carrier`, `start_age`, `enters_markov`). Write a test that the expected carriers add up to the numbers in the table above.

## Step 3: Markov states and transitions in R

Build one cohort Markov model for a woman of a given start age and carrier status, with yearly cycles to age 90. Run it once per cohort type from Step 2.

States (check against Figure 1):

- BRCA positive, BRCA negative
- Post RRM, post RRBSO, post RRM plus RRBSO
- Breast cancer, ovarian cancer
- Cured, dead

That is Figure 1b. It shows one state per cancer, but costs differ in year 1 and years 2 to 5, and people are cured after 5 years. A cohort Markov model has no memory, so you need tunnel states: year 1, year 2, year 3, year 4 and year 5 after diagnosis, each moving to the next, then cured. Draw your own version of Figure 1b with the tunnels before coding.

Rules from the paper to write as code comments: ovarian cancer can follow breast cancer but not the reverse; probands keep a risk of a second breast cancer; surgery happens within a year of a positive result; daughters stay in the no-surgery state until 30; no surveillance-only or chemoprevention arm.

Risks change with age, so build one transition matrix per cycle. Convert rates and cumulative risks to yearly probabilities with:

```latex
p = 1 - e^{-r t} \qquad r = \frac{-\ln(1 - p)}{t}
```

Table 1 already gives annual risks by 10-year age band for carriers (breast, ovarian, contralateral) and for the general population, so you can use them directly as yearly probabilities. Apply surgery as relative risks from Table 1: RRM 0.110 on breast; RRBSO 0.550 on breast and 0.166 on ovarian; both 0.110 on each. These differ slightly from the percentages in the text. Use Table 1.

Keep rates and probabilities apart. A 5-year breast cancer mortality of 10% gives a rate of 0.0211 and a yearly probability of 0.0209, both about 0.021 as in Tuffaha. A 5-year ovarian mortality of 57% gives a rate of 0.169 and a yearly probability of 0.155. Tuffaha's 0.15 is the probability, rounded. Both drafts called this a mismatch because they compared it with the rate. There is no mismatch, which is a good interview example of why the difference matters.

```r
step_cohort <- function(trace_row, P) trace_row %*% P

check_P <- function(P) {
  stopifnot(all(P >= 0), all(abs(rowSums(P) - 1) < 1e-9))
  stopifnot(P["dead", "dead"] == 1)
}
```

Tests for testthat: every matrix passes `check_P`; each row of the cohort trace sums to 1; nobody comes back from dead; with zero cancer risk, survival matches the life table.

## Step 4: Costs, QALYs and discounting

Attach a cost and a utility to each state, then add them up cycle by cycle.

- One-time costs (tests, counselling, surgery, first year of cancer) are charged when someone enters the state, not every year.
- Yearly costs (mammography at AU$90, cancer care in years 2 to 5 at AU$550 for breast and AU$5,000 for ovarian) are charged every cycle someone spends in the state. Year-1 cancer costs are AU$24,510 for breast and AU$20,000 for ovarian (Table 1).
- Keep the two mastectomy costs separate: probands already had breast-conserving surgery and get a contralateral mastectomy; relatives get a bilateral one.
- QALYs per cycle are the share in each state times its utility (0.90 no cancer, 0.79 breast cancer, 0.63 ovarian cancer). No disutility for surgery in the Tuffaha version.
- Count new breast and ovarian cancers as you go. They are a useful check against the cumulative risks in Step 6.

Discount both costs and QALYs at 5% for the Australian replication:

```latex
\text{value}_t = \frac{\text{value}}{(1 + r)^{t}}
```

If you choose a half-cycle correction, the exponent becomes t − 0.5. Run with and without it and see which matches Tuffaha better. Record the choice.

## Step 5: Run, aggregate, ICER and net monetary benefit

1. For each strategy, run the Markov part for each cohort type (proband, sisters, daughters, nieces), split into carriers and non-carriers.
2. Weight each cohort's costs and QALYs by its expected number from Step 2, and by the 15% chance that the proband is a carrier.
3. Add testing and counselling costs for everyone tested, men included.
4. Compare strategies:

```latex
\text{ICER} = \frac{C_{new} - C_{old}}{E_{new} - E_{old}} \qquad \text{NMB} = \lambda \, \Delta E - \Delta C
```

NOTE: Tuffaha's Analysis paragraph writes the ICER with *effectiveness over cost*, but its results use *cost over effectiveness*. Use *cost over effectiveness*. λ is the threshold (AU$50,000 per QALY). 

Three comparisons to report: 
- affected only vs none
- affected plus cascade vs none
- cascade vs affected only

## Step 6: Validate against Tuffaha's published results

With Australian inputs, the model should land close to these numbers (all of them are in the paper's text). 

| Comparison | Extra cost | Extra QALYs | ICER per QALY |
| --- | --- | --- | --- |
| Affected only vs none | AU$1,880 | 0.10 | AU$18,900 |
| Affected plus cascade vs none | AU$2,535 | 0.17 | AU$14,900 (text), AU$15,011 (Table 3) |
| Cascade vs affected only | AU$665 | 0.07 | AU$9,500 |

Table 2 also gives the totals behind these, which are better targets than the differences: affected only, AU$5,955 vs AU$4,076 and 15.83 vs 15.73 QALYs; with cascade, AU$8,602 vs AU$6,067 and 22.54 vs 22.37 QALYs.

Other checks:

- Cumulative risk to 70 in carriers without testing: 53% breast, 25% ovarian. With testing: 24% breast, 19% ovarian, 28% contralateral.
- Life-years: 0.09 for affected only. The paper reports 0.70 for affected plus cascade. Table 2 marks costs and QALYs as discounted but not life-years, so compare your undiscounted life-years with it.

Contingency: 
- If results are too far off, check the following in order:
  1. the family weights
  2. the surgery split
  3. penetrance
    
Write down every difference you can't explain in a table each with an explanation.

## Step 7: One-way and probabilistic sensitivity analysis

One-way analysis comes first (Stop 2). Change one input at a time between its low and high value, record the ICER, and draw a tornado diagram. Start with the inputs Tuffaha found mattered most: discount rate, the 15% pretest probability (5% to 20% in Table 3; the PSA uses uniform 10% to 20%), start age, and uptake of testing and surgery. Add relatives' uptake of cascade testing, since it is a gap for Canada and Mexico.

Probabilistic analysis comes later (Stop 3). Draw every input from a distribution, run the model many times, and plot the cost-effectiveness plane and acceptability curve. Tuffaha used 10,000 draws: beta for probabilities and utilities, gamma for costs, log-normal for relative risks, normal for starting ages, uniform for the pretest probability and Dirichlet for the surgery split. Standard errors are in Table 1.

To get beta parameters from a mean and standard error (method of moments):

```latex
n = \frac{p(1-p)}{SE^2} - 1 \qquad \alpha = p \, n \qquad \beta = (1 - p) \, n
```

Set a random seed and save it, so the results can be reproduced. Start with 1,000 draws while testing the code.

## Step 8: Canadian version

Keep Tuffaha's structure and change the inputs, one group at a time, in a new file `data/params_can.csv`. Rerun after each group so you can see what moved the result.

1. Framing: 1.5% discount (0% and 3% as scenarios), 2016 or 2022 CAD, public health system perspective. Report at CAD 50,000 and 100,000 per QALY.
2. Costs: test CAD 220 and counselling CAD 167 from Sun 2024 (2022 CAD). Surgery and cancer costs from the sources Hurry cites. Bring everything to one price year with Statistics Canada CPI.
3. Prevention: Hurry's 44% RRBSO and 21% RRM as a scenario next to Tuffaha's split. This is likely the biggest change.
4. Family size: Statistics Canada, as Hurry did.
5. Life tables: Statistics Canada.

Hurry is a patient-level simulation with a 50-year horizon, so don't expect your Canadian ICER to match its CAD 14,942 per QALY. Use it as a sense check, not a target. Hurry also compared testing plus surgery against no testing, with no proband-only arm.

## Step 9: Mexican scenario and the ancestry thread

Treat Mexico as a scenario analysis, not a full adaptation. About half the inputs have no Mexican source. Work in this order:

1. Family structure from INEGI or ENADID. Larger families mean more relatives per proband, which helps cascade testing. Younger mothers shift the children's ages.
2. Who gets tested. Torres-Mejia 2015 found 4.3% carriers in 810 unselected patients, and only about 15% of carriers had an affected first-degree relative. A 10% risk-tool threshold has not been validated in Mexican women, so testing all women with breast cancer is the more realistic base case.
3. Costs: Gonzalez-Robledo 2015 for breast cancer care (about US$8,557 per patient-year, US$6,500 for stage I to US$12,757 for stage III). Institutional unit costs for surgery and tests. Check that IMSS-Bienestar has replaced INSABI before citing INSABI tariffs.
4. Survival and stage: Unger-Saldana 2023 for women without social security.
5. Utilities: Tuffaha's cancer utilities, labelled as borrowed. The Mexican EQ-5D-5L value set (Gutierrez-Delgado 2021) turns questionnaire answers into utilities. It does not give utilities for cancer states by itself.
6. Gaps (surgery uptake, cascade uptake, ovarian cancer costs and survival): one illustrative value each with a wide range, listed in the data gap register.

The ancestry thread goes here, as one scenario and one paragraph. Penetrance comes from cohorts of mostly European ancestry. Nobody has measured it in Mexican carriers, and the variant mix differs (recurrent BRCA1 ex9-12del). Run the model with penetrance lower and higher (for example by 20%, a range you choose and justify) and report how much the ICER moves. Say plainly that this is a sensitivity test and not an estimate.

## Step 10: Reporting

Three short tables go in the README and on the Lunaland project page:

- Assumptions table: each assumption from the paper and from you, with where it comes from. For example: men tested but not modelled, cured at 5 years, no chemoprevention, daughters act at 30, no PARP inhibitor benefit, no surgery disutility.
- Data gap register: every input marked ✘ in the master table, the value you used, its range, and why.
- Differences from Tuffaha: from Step 6, each with an explanation.

At Stop 3, go through the CHEERS 2022 checklist and say where each item is covered. Label every output "learning model adapted from Tuffaha et al. 2018, illustrative, not for clinical or policy use". Present it as a replication and an exercise in methods, not as health economics expertise or policy advice.

## Sources

Full references with DOI links are in [parameters/sources.csv](parameters/sources.csv) and [parameters/parameters_can-mex-aus.bib](parameters/parameters_can-mex-aus.bib). Read in full on Nov 20, 2026:

- [Tuffaha et al. 2018, Genetics in Medicine](https://doi.org/10.1038/gim.2017.231) (full PDF, including Tables 1 to 3)
- [Hurry et al. 2020, Int J Technol Assess Health Care](https://doi.org/10.1017/S0266462319003519) (open access)
- [Sun et al. 2024, JAMA Network Open](https://doi.org/10.1001/jamanetworkopen.2024.32725)
- [ISPOR summary of Mexico's CSG economic evaluation guide (Jan 2023)](https://ispor.org/heor-resources/more-heor-resources/pharmacoeconomic-guidelines/pe-guideline-detail/m%C3%A9xico)
