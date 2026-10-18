# PFT Interpreter

**Use it here: https://henriquepuls.github.io/pft-interpreter/**

Single-page interpreter for spirometry, bronchodilator response, lung volumes and diffusing capacity following the ATS/ERS 2022 interpretation standard (Stanojevic 2022). The page is `index.html`: inline CSS and JavaScript, no build step, no external scripts, no network access at run time. It is hosted on GitHub Pages from the `main` branch and also published as a private Claude artifact; the file opens with a doctype and viewport meta so it renders on its own, and the artifact viewer wraps it in its own skeleton.

## Reference values (added 2026-10-18)

When age, sex and height are entered in the Demographics panel, the page computes predicted value, lower limit of normal (LLN), percent predicted and z-score for nine parameters and fills the empty percent-predicted and z-score fields as measured values are typed. A field the user typed is kept until the Recompute button is pressed, which overwrites every computable field. Clearing the age or height removes the values the engine wrote and leaves user-typed values alone.

| Parameters | Equation set | Age range | Covariates |
|---|---|---|---|
| FEV1, FVC, FEV1/FVC | GLI Global 2022 spirometry (Bowerman 2023), the default | 3 to 95 y | age, sex, height; race-neutral, the ancestry selector is disabled |
| FEV1, FVC, FEV1/FVC | GLI-2012 spirometry (Quanjer 2012), selectable | 3 to 95 y | age, sex, height, ancestry (White, African American, North East Asian, South East Asian, other or mixed) |
| TLC, RV, RV/TLC | GLI-2021 static lung volumes (Hall 2021) | 5 to 80 y | age, sex, height (European-ancestry reference data, no ancestry term) |
| DLCO, VA, KCO | GLI-2017 TLCO (Stanojevic 2017, with the 2020 author correction) | 5 to 90 y | age, sex, height; traditional units (mL/min/mmHg) |

Outside an equation set's age range that row is reported as unavailable and its fields are left blank. The "Spirometry equations" selector in the Demographics panel switches between GLI Global 2022 (default, following the 2023 ATS statement on race-neutral reference equations) and GLI-2012. The equation table names the set in use, and the Impression line of the report names it whenever spirometry reference values were computed on the page (it is not named when the lab's values were typed in, as in the presets).

### Method

All three sets use the LMS method with age-dependent spline terms read from the published lookup tables (0.25-year steps, linear interpolation between steps, the convention used by the GLI calculator and by the rspiro and pft R packages):

- M = exp(a0 + a1·ln(height) + a2·ln(age) + ancestry terms + Mspline)
- S = exp(p0 + p1·ln(age) + ancestry terms + Sspline)
- L = q0 + q1·ln(age) + Lspline (GLI-2021 and GLI-2017 use a constant L per parameter)
- z = ((measured / M)^L − 1) / (L · S), percent predicted = 100 · measured / M
- LLN = M · (1 − 1.645 · L · S)^(1/L), ULN = M · (1 + 1.645 · L · S)^(1/L)

GLI-2021 uses the covariate scale given in Hall 2021 Table 3: TLC uses ln(age) and ln(height) for M and age for S; RV and RV/TLC use untransformed age and height for M and age for S. Predicted RV/TLC is expressed in percent. The FEV1/FVC LLN is written to the ratio LLN field.

### Hemoglobin, carboxyhemoglobin and altitude (GLI-2017 DLCO and KCO)

The GLI-2017 equations describe DLCO at a standard hemoglobin and at sea level (the source data were standardised to a barometric pressure of 760 mmHg). Following the ATS/ERS 2017 technical standard (Graham 2017), adjustments are applied to the predicted value, not to the measured value:

- Hemoglobin, when entered and the lab has not already corrected the DLCO: predicted DLCO and KCO are multiplied by 1.7·Hb / (10.22 + Hb) for males aged 15 and over (standard Hb 14.6 g/dL) and by 1.7·Hb / (9.38 + Hb) for females and children under 15 (standard Hb 13.4 g/dL), Graham 2017 equations 31 and 32.
- Carboxyhemoglobin, when entered and above 2 %: predicted DLCO and KCO are multiplied by (102 − COHb %) / 100, Graham 2017 equation 37, which the standard describes as reducing the predicted DLCO by 1 % for each percentage point of COHb above 2 % (the equation and its text were read from the ERS article page on 2026-10-18). At or below 2 %, the usual endogenous level, no adjustment is made. Without demographics the same factor is applied to the lab's DLCO % predicted (divided by the factor), mirroring the hemoglobin fallback, with a finding line stating it; the two paths exclude each other so the factor is never applied twice. The COHb "anaemia effect" and back-pressure correction of equation 38 (measured DLCO × (1 + FACOb/560)) is not implemented.
- Test-site altitude, when entered: predicted DLCO and KCO are divided by 1 + 0.0035·(PAO2 − 100), Graham 2017 equation 34 (0.35 % per mmHg of alveolar oxygen tension). PAO2 at the site is derived from the standard-atmosphere barometric pressure, PIO2 = 0.21·(PB − 47) and the alveolar gas equation with PACO2 40 mmHg and R 0.8, and the factor is normalised so that 0 m gives exactly 1. At 1600 m this raises predicted DLCO by about 11 %.

When the engine has adjusted the predicted value, the interpretation panel reports the adjustment and skips its older percent-predicted correction so the hemoglobin effect is never applied twice. The computed z-score and percent predicted already include the adjustment.

### Data sources

The coefficient and spline tables embedded in `index.html` (constant `GLI_DATA`, about 130 kB) were taken from the `sysdata.rda` file of the CRAN package `pft` 1.0.1 (R/sysdata.rda), which packages the published GLI supplements:

- GLI-2012: `lookuptables.xls` from the ERJ supplement to Quanjer et al. 2012 (DOI 10.1183/09031936.00080312). Coefficients are the 4-decimal values of the workbook; the identical values are in CRAN `rspiro` 0.5.
- GLI Global 2022: `gli_global_lookuptables_dec6.xlsx` from the GLI network, as packaged in pft 1.0.1 (Bowerman et al., Am J Respir Crit Care Med 2023;207:768, DOI 10.1164/rccm.202205-0963OC). All six spline tables and all coefficients are identical to those shipped in CRAN `rspiro` 0.5 (`GLIgl_lookup` and `getLMS_GLIgl.R`), checked cell by cell.
- GLI-2021: supplementary material 2 of Hall et al. 2021 (DOI 10.1183/13993003.00289-2020) for the splines; regression coefficients from Table 3 of the paper.
- GLI-2017: the corrected supplementary workbooks of Stanojevic et al. 2017 (DOI 10.1183/13993003.00010-2017; correction DOI 10.1183/13993003.50010-2017), traditional-unit sheets.

The R data files were decoded with a small reader written for this project (bzip2-compressed XDR serialisation) and converted to compact JavaScript arrays with spline values rounded to 7 decimal places. Nothing is fetched at run time.

### Validation

The engine was checked against the GLI web calculator output that ships with `pft` 1.0.1 as its test oracle (`gli_test_groundtruth`: 1000 subjects, both sexes, all five ancestry groups, ages 5 to 88.2 years, heights 50 to 230 cm). For every parameter the computed predicted value, LLN and ULN, rounded to 3 decimals, matched the oracle in all rows (maximum absolute difference 0.001, one rounding tie in RV/TLC ULN):

| Set | Rows compared | Result |
|---|---|---|
| GLI-2012 FEV1, FVC, FEV1/FVC | 1000 each | all match |
| GLI-2021 TLC, RV, RV/TLC | 887 each (113 subjects older than 80 y are outside the equations) | all match |
| GLI-2017 DLCO, VA, KCO | 941 each (the calculator oracle stops at 85 y) | all match |

Four subjects from that grid, chosen to cover both sexes, two ancestry groups and both age extremes (oracle predicted / LLN, then this page):

| Subject | FEV1 (L) | FVC (L) | FEV1/FVC | TLC (L) | RV (L) | RV/TLC (%) | DLCO | VA (L) | KCO |
|---|---|---|---|---|---|---|---|---|---|
| Male, 46.6 y, 170 cm, White. Oracle | 3.650 / 2.870 | 4.565 / 3.593 | 0.802 / 0.694 | 6.490 / 5.251 | 1.585 / 0.923 | 24.20 / 15.39 | 27.37 / 21.06 | 5.959 / 4.853 | 4.626 / 3.600 |
| Same, this page | 3.650 / 2.870 | 4.565 / 3.593 | 0.802 / 0.694 | 6.490 / 5.251 | 1.585 / 0.923 | 24.20 / 15.39 | 27.37 / 21.06 | 5.959 / 4.853 | 4.626 / 3.600 |
| Female, 36.2 y, 150 cm, African American. Oracle | 2.256 / 1.756 | 2.671 / 2.092 | 0.845 / 0.727 | 4.359 / 3.567 | 1.043 / 0.569 | 23.64 / 14.02 | 19.10 / 15.17 | 4.088 / 3.383 | 4.739 / 3.744 |
| Same, this page | 2.256 / 1.756 | 2.671 / 2.092 | 0.845 / 0.727 | 4.359 / 3.567 | 1.043 / 0.569 | 23.64 / 14.02 | 19.10 / 15.17 | 4.088 / 3.383 | 4.739 / 3.744 |
| Female, 83 y, 150 cm, African American. Oracle | 1.351 / 0.888 | 1.756 / 1.171 | 0.778 / 0.620 | n/a | n/a | n/a | 15.92 / 11.71 | 3.703 / 2.919 | 4.273 / 3.217 |
| Same, this page | 1.351 / 0.888 | 1.756 / 1.171 | 0.778 / 0.620 | outside 5 to 80 y | outside | outside | 15.92 / 11.71 | 3.703 / 2.919 | 4.273 / 3.217 |
| Male, 5 y, 110 cm, African American. Oracle | 0.951 / 0.724 | 1.029 / 0.785 | 0.930 / 0.814 | 1.549 / 1.197 | 0.484 / 0.159 | 21.63 / 7.31 | 10.14 / 6.96 | 1.650 / 1.300 | 6.254 / 4.223 |
| Same, this page | 0.951 / 0.724 | 1.029 / 0.785 | 0.930 / 0.814 | 1.549 / 1.197 | 0.484 / 0.159 | 21.63 / 7.31 | 10.14 / 6.96 | 1.650 / 1.300 | 6.254 / 4.223 |

DLCO and KCO are in mL/min/mmHg and mL/min/mmHg/L.

GLI Global 2022 was validated three ways, since the GLI Global web calculator was not queried:

1. An independent emulation of rspiro 0.5 (its own `GLIgl_lookup` table, the coefficients hard-coded in `getLMS_GLIgl.R`, and its interpolation) run on the same 1000-subject grid: 3000 predicted values and LLN agree with the page engine to within 4 × 10⁻⁷ L, and z-scores at 80 % predicted to within 3 × 10⁻⁷.
2. The six Bowerman 2023 supplement Table E2 by-hand cases recorded in the pft test suite all match to 5 decimals.
3. The 30-row `gli_2022_oracle.csv` fixture of pft 1.0.1 matches for predicted, LLN, z-score and % predicted to within 4 × 10⁻⁶.

Subjects for the README (predicted / LLN, page engine and rspiro emulation agree to the shown precision):

| Subject | FEV1 (L) | FVC (L) | FEV1/FVC |
|---|---|---|---|
| Male, 25 y, 170 cm (Bowerman Table E2) | 3.893 / 3.046 | | |
| Female, 45 y, 170 cm (Bowerman Table E2) | | 3.759 / 2.837 | |
| Female, 70 y, 170 cm (Bowerman Table E2) | | | 0.782 / 0.654 |
| Male, 46.6 y, 170 cm | 3.418 / 2.591 | 4.203 / 3.211 | 0.808 / 0.699 |
| Female, 36.2 y, 150 cm | 2.420 / 1.844 | 2.829 / 2.171 | 0.853 / 0.742 |
| Male, 5 y, 110 cm (lower age limit) | 1.078 / 0.838 | 1.179 / 0.920 | 0.922 / 0.808 |
| Female, 95 y, 160 cm (upper age limit) | 1.645 / 1.070 | 2.223 / 1.479 | 0.768 / 0.622 | The published worked examples are also reproduced: Hall 2021 supplement FRC example (male, 30 y, 178 cm) and the Stanojevic 2017 corrected TLCO value for the same subject (11.06 mmol/min/kPa) match the pft package tests that pin them, since the same tables and equation forms are used.

Page-level checks run with node (script extracted from `index.html` and executed under a minimal DOM shim): the seven example presets produce the same headline and report text before and after the change; entering the oracle predicted values as measurements yields z = 0.00 and 100 % predicted; a user-typed z-score survives a change of the measured value until Recompute; hemoglobin 10 g/dL in a man lowers predicted DLCO by the factor 0.841; ages outside a set's range leave that set's fields blank. Second round: switching the spirometry selector between GLI Global 2022 and GLI-2012 changes the computed FEV1 % predicted (107 % against GLI Global, 100 % against GLI-2012 for the GLI-2012 oracle value) and disables or enables the ancestry selector; COHb 7 % with demographics gives DLCO and KCO 105 % predicted (factor 0.95) and the adjustment finding; COHb 1.5 % leaves them at 100 %; without demographics COHb 7 % converts a lab value of 80 % predicted to 84 % with the fallback finding; the Impression line names the spirometry set only when the page computed it.

### Limitations

- The ancestry selector affects GLI-2012 spirometry only. GLI-2021 lung volumes were derived from European-ancestry data and GLI-2017 TLCO from White subjects; the page applies them to everyone and says so in the equation note.
- The altitude factor applies the oxygen-tension equation of the 2017 standard to a derived site PAO2. It is an approximation for laboratories above sea level and is off unless an altitude is entered.
- The COHb adjustment uses equation 37 of the 2017 standard (predicted value); the measured-side back-pressure correction (equation 38) is not implemented.
- GLI Global 2022 was not checked against the GLI web calculator itself, only against rspiro, the Bowerman worked examples and the pft fixture.

## Development notes

- Keep the page self-contained and under 16 MB (currently about 190 kB).
- Check syntax after edits: extract the script between the `<script>` tags and run `node --check` on it.
- Writing style for page text: plain sentences, no em dashes.

## How this was made

This tool was built with Claude Code using Anthropic's Claude Fable 5.1 model, under the direction of Henrique Puls, who specified the clinical logic and reviewed the output.
