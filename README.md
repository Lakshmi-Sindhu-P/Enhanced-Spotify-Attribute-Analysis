# 🎶📊 Enhanced Analysis on Spotify Attributes Data 🎶📊

**Author: Lakshmi Sindhu Pulugundla**
**Analysis date: 2025-10-04 · corrected re-analysis: 2026-10-27**

***

## 👋 Welcome!

This notebook asks which musical attributes predict how long a track gets listened
to, using the Spotify song-attributes export in `data/`. It walks through the full
path — data audit, cleaning, exploratory analysis, regression, and the robustness
checks a claim of this kind needs.

> **Status:** the notebook is self-contained and knits end to end:
> `rmarkdown::render("Enhanced-Spotify-Attribute-Analysis.rmd")`.
> It needs only base R plus `rmarkdown`; `nlme` (which ships with R) is used for one
> mixed-model check and is optional. Most numbers are interpolated from the code
> above them, but not all: the figures quoted in this README and a handful of
> contextual numbers in the prose (the 22 columns, the 1,400-track test set) are
> transcribed from the rendered output. Any number that can be re-derived should be
> re-derived rather than trusted from the prose.

The first version of this analysis is preserved unmodified in `../work/v1/`
(alongside the repository)
(notebook, rendered HTML, and its README). It reached two conclusions this version
does not support; **Section 7 of the notebook** states exactly what changed and
why.

***

## 🚦 What the notebook covers

### 1️⃣ Data and the unit of observation
- **Is a row a track?** No — every track appears exactly twice, identical in all 22
  columns, because a join on a non-unique key doubled the file. The analysis
  proceeds on 5,040 distinct tracks.
- **Duplication mechanism**: 29 ids are shared by two different titles (a track and
  its remix), and 550 rows carry no id and no audio features — the unmatched half of
  a left join.
- **What `msPlayed` measures**: cumulative time, not one play. 37.5% of tracks were
  abandoned before a full play; a few were repeated hundreds of times. The response
  is a mixture of *skipping* and *duration*, and the analysis treats it that way.

### 2️⃣ Preprocessing
- Genre standardisation with the consolidation map, rare levels collapsed **after**
  mapping, missing genre kept as its own `Unspecified` level rather than merged into
  `Other`.
- Artist names transliterated from UTF-8 before punctuation stripping (so `Beyoncé`
  and `Björk` survive), lowercased, then collapsed on distinct-track counts.
- **No imputation, no IQR capping, no min-max.** The 275 tracks with no audio
  features are excluded from the audio models and checked explicitly with an
  indicator model instead of being filled with fabricated values. Predictors are
  used in their natural units and effects are reported per standard deviation.
- Response modelled as `log10(msPlayed + 1)`; a Box-Cox profile is shown alongside
  and the alternative scales are carried through the specification curve.

### 3️⃣ Exploratory analysis
- Response shape on three scales, including `plays = msPlayed / duration`
- Associations with the response (Pearson and Spearman), collinearity among predictors
- Genre differences and **skip rates by genre** — the pattern that later becomes the
  main finding

### 4️⃣ Regression
- **Model ladder** (null → audio → genre → artist → additive → interactions) with
  nested F-tests, AIC and BIC, so every block has to earn its place
- **HC3 robust standard errors**, computed directly, alongside the classical ones
- **Diagnostics**: residual plots, Breusch-Pagan, Shapiro-Wilk, Cook's distance, VIF,
  condition number

### 5️⃣ Robustness
- **200 random 70/30 splits** instead of one, because one split of 1,400 tracks has
  an R² standard error about as large as the effect under discussion
- **Specification curve** across response scale × genre coding × artist coding
- **Random intercept for artist**, because artists contribute several tracks each
- **Selection check** on the tracks that were excluded

### 6️⃣ The decomposition
The response is split into its two behavioural margins: whether a track was skipped
(logistic model), and how much it was played given at least one full play (linear
model). This is where the audio attributes actually show up, and it changes the
interpretation of the regression coefficient.

***

## 📝 What we learned

- The five predictors together (three audio attributes, genre, artist) explain
  **~8.5%** of the variance in log playback time, and the model **does generalise**
  out of sample (mean test R² ≈ 0.05 over 200
  splits). The earlier "does not predict at all" result was an artefact of modelling
  the raw millisecond scale: the same specification averages **−0.024** there over the
  same 200 splits, and is negative in 82% of them, because a handful of enormous values dominate squared error.
- **Genre and artist are the substantive predictors** (8.3% together) and each adds
  significantly over the other. BIC, which charges for the artist dummies, prefers
  genre alone — reported rather than buried.
- **The audio attributes add 0.26 percentage points** — small, statistically
  detectable (`F(3, 4697) = 4.38`, `p = 0.0044`), robust to HC3 standard errors, an
  artist-clustered Wald test and a random intercept for artist, but the out-of-sample
  gain is not clearly positive and BIC prefers the model without them.
- They are **not redundant** in the sense the first version claimed: only 18% of
  danceability, 26% of energy and 4% of tempo variation is explained by genre and
  artist, so most of their variance is information the categoricals do not carry.
  They are simply not very informative about *this* outcome.
- **Energy is the one distinguishable attribute, and its sign depends on which
  question you ask.** On the skip margin the audio block is clearly detected
  (`p = 6.8e-07`): a one-SD increase in energy multiplies the odds of falling below
  one full play by 1.22, and the top energy decile is skipped 47% of the time against
  36% below it. On the duration margin — among tracks played at least once — the
  *block* is not significant (`p = 0.24`), but energy on its own is, with the
  **opposite sign** (`+0.10`, HC3 `p = 0.045` — marginal). The pooled negative
  coefficient is not an average of the two (they are on different scales); it is
  essentially the skip channel: told whether a full play occurred, the model's energy
  coefficient drops to `+0.03` and the indicator absorbs the association.

***

## ❓ Analysis FAQ

**Q: Why not impute the missing audio features with KNN?**
A: The first version did. Two problems: the neighbour distance was computed over all
22 columns including `msPlayed` — the outcome — and the resulting imputations
collapsed onto a single donor value, so 550 rows received a fabricated point mass.
Excluding those 275 tracks and reporting an indicator-model sensitivity check is more
honest than inventing values, and the notebook states the cost of the exclusion.

**Q: Why log-transform the response?**
A: `msPlayed` is skewed 13.6 with a 44-hour maximum, and the scale mixes skips with
repeats. A log (or Box-Cox) scale stabilises the variance, makes the residuals
usable, and turns coefficients into proportional effects. It is also what makes the
model predict at all out of sample.

**Q: Doesn't deduplication change the results?**
A: It leaves every coefficient exactly where it was and halves the sample size, so
standard errors grow by √2, every t-statistic is divided by √2 and every p-value
**grows** — a study that looks significant on doubled data may not be significant on
the real sample. Every AIC, every count and every "significant at the 5% level" claim
changes with them. That is why the audit had to come first.

**Q: Why so many robustness checks?**
A: Because the headline effect is small (0.26 pp of R²). An effect that size is
exactly the kind that can be created or destroyed by a defensible analyst choice, so
the notebook shows the choices being varied rather than picking one and asserting it.

**Q: Only 8% of variance explained — is that a failure?**
A: It is the answer, with a qualification. The 8% is measured at one particular
artist-pruning threshold; unpruned, the same model reaches R² = 0.57, almost all of
it from 2,187 artist dummies fitted to 4,765 tracks — and on held-out tracks that
design scores **R² ≈ −0.6**, worse than predicting the mean, in every split tried
(Section 5.2). The extra R² is memorisation, not information. So the ceiling is set by the
bias-variance trade-off, not by the attributes being uninformative. What the data
does not contain is context: mood, device, playlist position, and who is listening.

**Q: Where can I check the intermediate steps?**
A: Every transformation prints its own before/after summary in the notebook, and the
first version's code and output are kept in `../work/v1/` so the two can be compared
line by line.

***

## 🚀 Next Steps

The remaining opportunities are about **data**, not about model tuning:

1. **Get the listening context.** Session, device, time of day, and whether a track
   was chosen deliberately or served by a playlist. Nothing in this file
   distinguishes a deliberate skip from a background track being scrolled past.
2. **Model the two margins separately by design.** The skip/duration split in
   Section 6 is the most informative thing in the analysis; a proper hurdle or
   survival formulation would make it the primary model rather than a follow-up.
3. **Recover the missing audio.** 275 tracks lost their audio features to a failed
   join. Matched properly, they would add back 5.5% of the sample — and they are not
   missing at random.
4. **Use a genre taxonomy that survives.** About half of the modelling sample
   collapses into `Other`; a coarser but complete taxonomy would let genre do more of
   the work.

None of these is a modelling upgrade. The current model is close to the ceiling this
dataset allows *at the pruning threshold that generalises*, and Section 5.2 shows how
much of the remaining headroom is just extra artist dummies.

***

**Interested? Open the RMarkdown notebook and reproduce any step — or reach out for deeper explanations or code tweaks!**
