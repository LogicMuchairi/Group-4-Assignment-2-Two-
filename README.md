================================================================================
ASSIGNMENT 2
================================================================================

1. EXECUTIVE SUMMARY & ASSIGNMENT OBJECTIVES
--------------------------------------------------------------------------------
This documentation serves as the master specification and execution manual for 
Assignment 2. The objective of this project is to implement an end-to-end 
demographic and actuarial modeling pipeline using empirical population census 
data, individual policyholder exposure records, baseline mortality schedules, and 
macroeconomic/demographic projection scenarios.

Core Analytical Objectives:
  1. Data Auditing & Exposure Cleaning: Process multi-state policy logs, audit 
     overlapping/duplicate exposure windows, apply terminal state censoring, 
     and reconcile missing demographic markers across districts.
  2. Complete & Abridged Life Tables: Construct empirical life tables using the 
     Chiang method, evaluating age-specific mortality rates ($m_x$), survival 
     probabilities ($p_x$), and life expectancy ($e_x$) with Greenwood variance 
     estimation.
  3. Multi-State Transition Rate Estimation: Estimate continuous-time transition 
     intensity matrices ($\hat{\mu}_{ij}$) and discrete-time transition 
     probability matrices ($P_{ij}(t)$) for active, disabled, lapsed, and 
     deceased states.
  4. Cohort-Component Population Projections: Build a 30-year projection model 
     (2020–2050 in 5-year intervals) applying Leslie Matrix mechanics, Total 
     Fertility Rate (TFR) targets, mortality adjustments ($e_0$), and net 
     migration matrices.


2. REPOSITORY STRUCTURE & ENVIRONMENT REQUIREMENTS
--------------------------------------------------------------------------------
Directory Hierarchy:

  ASSIGNMENT_2/
  │
  ├── README.txt                     <- Primary technical manual (this document)
  │
  ├── data/                          <- Input CSV data files
  │   ├── age_sex.csv                <- District population, births & deaths
  │   ├── districts.csv              <- District metadata & socioeconomic index
  │   ├── insured_transitions.csv    <- Policyholder multi-state tracking logs
  │   ├── life_table_inputs.csv      <- Baseline age-specific mortality schedules
  │   └── projection_scenarios.csv   <- Scenario assumptions (TFR, e0, Migration)
  │
  ├── src/                           <- Computational Python / R scripts
  │   ├── 01_data_cleaning.py        <- Data auditing, validation & cleaning
  │   ├── 02_life_tables.py          <- Life table derivation engine
  │   ├── 03_multistate_model.py     <- Intensity rate & transition estimation
  │   └── 04_projections.py          <- Cohort-component projection engine
  │
  └── output/                        <- Generated outputs and audit reports
      ├── tables/                    <- Derived life tables & transition matrices
      ├── figures/                   <- Population pyramids & survival curves
      └── audit_log.txt              <- Data cleaning anomaly log

Recommended Environment & Dependencies:
  - Language: Python 3.9+ or R 4.2+
  - Python Core Packages: pandas (>=1.5.0), numpy (>=1.23.0), scipy (>=1.9.0), 
                          matplotlib (>=3.6.0), seaborn (>=0.12.0)
  - R Core Packages: tidyverse, mstate, demography, LifeTables


3. EXHAUSTIVE DATA DICTIONARIES & SCHEMAS
--------------------------------------------------------------------------------

3.1 Demographic Census Dataset (`age_sex.csv`)
  - Description: Contains district-level population counts, live births, and 
    deaths stratified by age bracket and biological sex.
  
  Columns:
  +-------------------+---------+----------------------------------------------+
  | Column Name       | Type    | Description & Constraints                    |
  +-------------------+---------+----------------------------------------------+
  | district_id       | String  | Unique district code (e.g., 'DST_001').      |
  | age_group         | String  | Standard 5-year age group ('0-4', '5-9', ...,|
  |                   |         | '80-84', '85+').                             |
  | sex               | String  | Sex category: 'M' (Male), 'F' (Female),      |
  |                   |         | or 'Total'.                                  |
  | population        | Integer | Mid-year resident population ($P_x$).        |
  | births            | Integer | Registered annual live births (Female 15-49).|
  | deaths            | Integer | Observed annual deaths ($D_x$).              |
  +-------------------+---------+----------------------------------------------+

3.2 Geographic & Socioeconomic Metadata (`districts.csv`)
  - Description: Regional characteristics used for spatial comparison.
  
  Columns:
  +-------------------+---------+----------------------------------------------+
  | Column Name       | Type    | Description & Constraints                    |
  +-------------------+---------+----------------------------------------------+
  | district_id       | String  | Foreign Key matching `age_sex.csv`.          |
  | district_name     | String  | Full district name.                          |
  | region            | String  | Macro zone ('North', 'South', 'Central', etc)|
  | urban_rural       | String  | Classification: 'Urban', 'Semi-Urban', 'Rural|
  | socioeconomic_idx | Float   | Socioeconomic score (0.0 = lowest, 100.0=max)|
  +-------------------+---------+----------------------------------------------+

3.3 Insured State Transitions Dataset (`insured_transitions.csv`)
  - Description: Multi-state tracking log for insured policyholders.
  
  Columns:
  +-------------------+---------+----------------------------------------------+
  | Column Name       | Type    | Description & Constraints                    |
  +-------------------+---------+----------------------------------------------+
  | policy_id         | String  | Unique policy identifier.                    |
  | age               | Integer | Attained integer age at start of interval.   |
  | initial_state     | String  | Origin state: 'Active', 'Disabled', 'Lapsed'.|
  | final_state       | String  | Destination state: 'Active', 'Disabled',     |
  |                   |         | 'Lapsed', 'Death', 'Surrender'.              |
  | exposure_time     | Float   | Exact fraction of year exposed ($0 < E \le 1$)|
  | transition_date   | String  | ISO-8601 date string ('YYYY-MM-DD').         |
  +-------------------+---------+----------------------------------------------+

3.4 Life Table Inputs Dataset (`life_table_inputs.csv`)
  - Description: Age-specific central mortality rates and separation factors.
  
  Columns:
  +-------------------+---------+----------------------------------------------+
  | Column Name       | Type    | Description & Constraints                    |
  +-------------------+---------+----------------------------------------------+
  | age               | Integer | Exact age $x$ at interval start.             |
  | n                 | Integer | Interval length in years ($n=1$ or $n=5$).    |
  | mx                | Float   | Observed central mortality rate ($m_x$).     |
  | ax                | Float   | Average fraction of year lived by deceased.  |
  +-------------------+---------+----------------------------------------------+

3.5 Scenario Parameters Dataset (`projection_scenarios.csv`)
  - Description: Assumption trajectories for cohort-component forecasting.
  
  Columns:
  +-------------------+---------+----------------------------------------------+
  | Column Name       | Type    | Description & Constraints                    |
  +-------------------+---------+----------------------------------------------+
  | scenario_id       | String  | Scenario label ('Baseline', 'High_Growth',   |
  |                   |         | 'Pessimistic').                              |
  | projection_year   | Integer | Target projection year (2025, 2030, ..., 2050|
  | tfr               | Float   | Target Total Fertility Rate (births/woman).  |
  | e0_male           | Float   | Target male life expectancy at birth.        |
  | e0_female         | Float   | Target female life expectancy at birth.      |
  | net_migration_rate| Float   | Annual net migrants per 1,000 residents.     |
  +-------------------+---------+----------------------------------------------+


4. PREPROCESSING, AUDITING & CLEANING PROTOCOLS
--------------------------------------------------------------------------------

4.1 Exposure Audit & Deduplication Rules
  1. Negative or Zero Exposure: Any record with `exposure_time` <= 0 must be 
     flagged and removed.
  2. Post-Terminal Activity: If a policyholder transitions to an absorbing 
     state ('Death' or 'Surrender'), all subsequent rows associated with that 
     `policy_id` must be dropped.
  3. Overlapping Intervals: Total cumulative exposure for policy $k$ in a 1-year 
     observation window cannot exceed 1.0 person-year:
     $$\sum_{r} E_{i, r}^{(k)} \le 1.0$$
  4. State Consistency: The `initial_state` of record $r$ must match the 
     `final_state` of record $r-1$ for the same policyholder.

4.2 Missing Data & Anomaly Resolution
  - Infant Separation Factor ($a_0$): If missing, calculate via Keyfitz rule:
    * If $m_0 \ge 0.107$, $a_0 = 0.35$
    * If $m_0 < 0.107$, $a_0 = 0.07 + 1.7 \cdot m_0$
  - Adult Separation Factors ($a_x$): For age groups $x \ge 1$, default to $0.5$ 
    (assuming uniform distribution of deaths across the interval).
  - Open-Ended Group ($85+$): Set $n = \infty$, $a_{85+} = \frac{1}{m_{85+}}$.


5. MATHEMATICAL & ACTUARIAL MODELING SPECIFICATIONS
--------------------------------------------------------------------------------

5.1 Complete & Abridged Life Table Equations (Chiang Method)

  1. Probability of Dying ($q_x$):
     $$q_x = \frac{n \cdot m_x}{1 + (1 - a_x) \cdot n \cdot m_x}$$

  2. Probability of Survival ($p_x$):
     $$p_x = 1 - q_x$$

  3. Cohort Survival Count ($l_x$), with Radix $l_0 = 100,000$:
     $$l_{x+n} = l_x \cdot p_x = l_x \cdot (1 - q_x)$$

  4. Cohort Death Count ($d_x$):
     $$d_x = l_x - l_{x+n} = l_x \cdot q_x$$

  5. Person-Years Lived ($L_x$):
     - Standard Interval $[x, x+n)$:
       $$L_x = n \cdot l_{x+n} + a_x \cdot n \cdot d_x$$
     - Open-Ended Terminal Interval ($85+$):
       $$L_{85+} = \frac{l_{85+}}{m_{85+}}$$

  6. Total Person-Years Remaining ($T_x$):
     $$T_x = \sum_{y=x}^{\infty} L_y$$

  7. Life Expectancy ($e_x$):
     $$e_x = \frac{T_x}{l_x}$$

  8. Standard Error of Life Expectancy (Greenwood's Formula):
     $$\text{Var}(q_x) = \frac{n^2 \cdot m_x \cdot (1 - a_x \cdot n \cdot m_x)}{D_x \cdot (1 + (1 - a_x) \cdot n \cdot m_x)^3}$$
     $$\text{Var}(e_x) = \frac{\sum_{y=x}^{85+} l_y^2 \cdot ((1 - a_y) \cdot n + e_{y+n})^2 \cdot \text{Var}(q_y)}{l_x^2}$$
     $$\text{SE}(e_x) = \sqrt{\text{Var}(e_x)}$$

5.2 Multi-State Actuarial Transition Modeling

  1. Maximum Likelihood Transition Intensity ($\hat{\mu}_{ij}$):
     For transition from active state $i$ to state $j$ ($i \neq j$):
     $$\hat{\mu}_{ij} = \frac{D_{ij}}{\sum_{k} E_i^{(k)}}$$
     where $D_{ij}$ is total observed transitions from $i$ to $j$, and 
     $\sum_k E_i^{(k)}$ is total person-years spent in state $i$.

  2. Transition Rate Matrix ($Q$):
     $$Q = \begin{pmatrix} 
     \mu_{AA} & \mu_{AD} & \mu_{AL} & \mu_{AM} \\
     \mu_{DA} & \mu_{DD} & \mu_{DL} & \mu_{DM} \\
     0 & 0 & 0 & 0 \\
     0 & 0 & 0 & 0 
     \end{pmatrix}$$
     where diagonal elements are defined as:
     $$\mu_{ii} = -\sum_{j \neq i} \mu_{ij}$$

  3. Continuous-to-Discrete State Transition Probabilities ($P(t)$):
     Given constant transition intensities over interval $t$:
     $$P(t) = \exp(Q \cdot t) = \sum_{k=0}^{\infty} \frac{(Q \cdot t)^k}{k!}$$

5.3 Cohort-Component Population Projection Engine (Leslie Matrix Model)

  1. Leslie Matrix Structure ($\mathbf{L}$):
     $$\mathbf{P}_{t+5} = \mathbf{L} \cdot \mathbf{P}_t + \mathbf{M}_{t+5}$$

     $$\mathbf{L} = \begin{pmatrix}
     F_0 & F_1 & F_2 & \dots & F_{k-1} & F_k \\
     S_0 & 0   & 0   & \dots & 0       & 0   \\
     0   & S_1 & 0   & \dots & 0       & 0   \\
     \vdots & \vdots & \ddots & \dots & \vdots & \vdots \\
     0   & 0   & 0   & \dots & S_{k-1} & 0
     \end{pmatrix}$$

  2. Survival Ratios ($S_x$):
     $$S_x = \frac{L_{x+5}}{L_x}$$
     For terminal group ($80-84$ to $85+$):
     $$S_{80+} = \frac{L_{85+}}{L_{80-84} + L_{85+}}$$

  3. Effective Age-Specific Fertility Rates ($F_x$):
     $$F_x = \frac{L_0}{2 \cdot l_0} \cdot \left( \text{ASFR}_x + \text{ASFR}_{x+5} \cdot S_x \right)$$
     where $\text{ASFR}_x$ is scaled so that:
     $$\sum_{x=15}^{49} 5 \cdot \text{ASFR}_x = \text{TFR}_{\text{scenario}}$$

  4. Net Migration Matrix ($\mathbf{M}_{t+5}$):
     $$M_{x, t+5} = P_{x, t} \times \left( \frac{\text{net\_migration\_rate}}{1000} \right) \times 5$$


6. ANALYTICAL PIPELINE & EXECUTION WORKFLOW
--------------------------------------------------------------------------------

  Step 1: Data Cleaning & Validation
    Command: `python src/01_data_cleaning.py`
    Outputs: `output/clean_age_sex.csv`, `output/clean_transitions.csv`
    Actions: Removes negative exposures, flags post-death logs, verifies FKs.

  Step 2: Life Table Derivation
    Command: `python src/02_life_tables.py`
    Outputs: `output/tables/life_tables_by_district.csv`
    Actions: Computes $q_x, l_x, d_x, L_x, T_x, e_x$, and Greenwood SEs.

  Step 3: Multi-State Transition Estimation
    Command: `python src/03_multistate_model.py`
    Outputs: `output/tables/transition_intensity_matrix.csv`
    Actions: Aggregates person-years exposure, calculates $\hat{\mu}_{ij}$ 
             and solves $P(t) = \exp(Q \cdot t)$.

  Step 4: Population Projections
    Command: `python src/04_projections.py`
    Outputs: `output/tables/projections_2020_2050.csv`, `output/figures/*.png`
    Actions: Simulates 5-year cohort steps through 2050 across all scenarios.


7. QUALITY ASSURANCE & INTEGRITY CHECKLIST
--------------------------------------------------------------------------------
  [ ] Radix Consistency: $l_0$ equals exactly 100,000 in all life tables.
  [ ] Monotonic Decrements: $l_x$ decreases monotonically with age.
  [ ] Probability Bounds: $0 \le q_x \le 1.0$ and $0 \le p_x \le 1.0$ across all ages.
  [ ] Exposure Non-Negativity: All person-years exposures $\sum E_i > 0$.
  [ ] Intensity Matrix Row Sums: Every row sum of rate matrix $Q$ equals zero ($\sum_j \mu_{ij} = 0$).
  [ ] Population Conservation: Projections correctly balance Births - Deaths + Migration.
  [ ] Foreign Key Integrity: 100% match rate between `district_id` keys across tables.


8. ASSIGNMENT GRADING RUBRIC MAPPING
--------------------------------------------------------------------------------
  +--------------------------------+----------------------------+--------------+
  | Rubric Requirement             | Implementation Script      | Target Score |
  +--------------------------------+----------------------------+--------------+
  | 1. Data Cleaning & Exposure    | `src/01_data_cleaning.py`  | 100% / Max   |
  | 2. Life Table Construction     | `src/02_life_tables.py`    | 100% / Max   |
  | 3. Multi-State Transition Rates| `src/03_multistate_model.py`| 100% / Max   |
  | 4. Cohort-Component Forecasts  | `src/04_projections.py`    | 100% / Max   |
  | 5. Technical Documentation     | `README.txt`               | 100% / Max   |
  +--------------------------------+----------------------------+--------------+

================================================================================
END OF TECHNICAL SPECIFICATION MANUAL
================================================================================
