# Protocol: Bioreactor Modeling in gPROMS

*Prepared 2026-10-24*

## Overview

This protocol builds a fed-batch bioreactor model in gPROMS, simulates it, fits its kinetic parameters to lab data, and optimizes the feed. gPROMS is an equation-oriented modeling platform from Siemens (formerly Process Systems Enterprise); this protocol uses the general-purpose gPROMS language, as in gPROMS ModelBuilder or [gPROMS Process](https://www.siemens.com/en-us/products/gproms/process/).

**Example system:** an aerobic microbial fermentation with biomass (X), substrate (S) and product (P), Monod growth, Luedeking–Piret product formation, and a glucose feed that raises the volume.

**Workflow:**

1. Create a project and define variable types.
2. Write the MODEL: parameters, variables and equations.
3. Write the PROCESS: parameter values, inputs, initial conditions and schedule.
4. Run the dynamic simulation and check the results.
5. Estimate kinetic parameters from experimental data.
6. Optimize the feed profile to maximize product.

Menu names and some keywords differ slightly between gPROMS versions. Check them against the documentation for your installed version.

## Requirements

- **Software:** gPROMS ModelBuilder or gPROMS Process with a valid licence. Parameter estimation and optimization may need extra licence features.
- **Background:** mass balances, Monod kinetics, ordinary differential equations.
- **Data for Step 6:** time-course measurements of X, S and P (g/L) from at least 2 batch or fed-batch runs, with feed rate, feed concentration and starting volume recorded.
- **Starting guesses for parameters:**

| Parameter | Symbol | Typical starting value | Unit |
| --- | --- | --- | --- |
| Maximum growth rate | μmax | 0.4 | 1/h |
| Half-saturation constant | Ks | 0.5 | g/L |
| Biomass yield on substrate | Yxs | 0.5 | g/g |
| Growth-associated product coefficient | α | 0.2 | g/g |
| Non-growth product coefficient | β | 0.01 | g/(g·h) |
| Maintenance coefficient | ms | 0.02 | g/(g·h) |

These are generic textbook values for a bacterial fermentation. Replace them with values for your organism.

## Steps 1–2: Project and variable types

### Step 1: Create the project

1. Open gPROMS and choose **File → New** to create a project, for example `FedBatch_Bioreactor.gPJ`.
2. In the project tree you will add entities under these folders: **Variable Types**, **Models**, **Processes**, and later **Experiments** and **Parameter Estimations**.
3. Save often. Each entity is checked for syntax when you save or build.

### Step 2: Define variable types

A variable type sets the default value, lower and upper bounds and unit for a class of variables. Bounds stop the solver from wandering into impossible values such as negative concentrations.

Right-click **Variable Types** → **New Entity**, then add one row per type:

| Name | Default | Lower bound | Upper bound | Unit |
| --- | --- | --- | --- | --- |
| Concentration | 1 | 0 | 500 | g/L |
| Volume | 1 | 0 | 100 | L |
| Flowrate | 0 | 0 | 10 | L/h |
| SpecificRate | 0.1 | -10 | 10 | 1/h |
| Mass | 1 | 0 | 1E4 | g |

## Step 3: Write the bioreactor MODEL

The model is 4 mass balances plus 1 kinetic expression. F is the feed rate and Sf the feed substrate concentration.

$$\frac{dV}{dt} = F$$

$$\frac{dX}{dt} = \mu X - \frac{F}{V} X$$

$$\frac{dS}{dt} = \frac{F}{V}(S_f - S) - \left(\frac{\mu}{Y_{xs}} + m_s\right) X$$

$$\frac{dP}{dt} = (\alpha \mu + \beta) X - \frac{F}{V} P$$

$$\mu = \mu_{max} \frac{S}{K_s + S}$$

Right-click **Models** → **New Entity**, name it `FedBatchReactor`, and enter:

```
# Fed-batch bioreactor: Monod growth, Luedeking-Piret product
PARAMETER
  mu_max  AS REAL     # maximum specific growth rate, 1/h
  Ks      AS REAL     # half-saturation constant, g/L
  Yxs     AS REAL     # biomass yield on substrate, g/g
  alpha   AS REAL     # growth-associated product term, g/g
  beta    AS REAL     # non-growth product term, g/(g h)
  ms      AS REAL     # maintenance coefficient, g/(g h)
  Sf      AS REAL     # feed substrate concentration, g/L

VARIABLE
  X   AS Concentration   # biomass
  S   AS Concentration   # substrate
  P   AS Concentration   # product
  V   AS Volume          # broth volume
  F   AS Flowrate        # feed rate (input)
  mu  AS SpecificRate    # specific growth rate
  D   AS SpecificRate    # dilution rate F/V
  Ptot AS Mass           # total product in reactor

EQUATION
  # Kinetics
  mu = mu_max * S / (Ks + S) ;
  D  = F / V ;

  # Mass balances
  $V = F ;
  $X = mu * X - D * X ;
  $S = D * (Sf - S) - (mu / Yxs + ms) * X ;
  $P = (alpha * mu + beta) * X - D * P ;

  # Output of interest
  Ptot = P * V ;
```

**Degrees of freedom:** 8 variables, 7 equations. F is the one input you must set in the PROCESS. gPROMS reports the count when you build; it must be zero before a simulation will run.

## Steps 4–5: PROCESS entity and simulation

### Step 4: Write the PROCESS

The PROCESS sets parameter values, fixes the input, gives initial conditions and runs a schedule. Here the reactor runs as a batch for 8 h, then feeds at 0.05 L/h for 16 h.

Right-click **Processes** → **New Entity**, name it `Sim_FedBatch`, and enter:

```
UNIT
  R101 AS FedBatchReactor

SET
  WITHIN R101 DO
    mu_max := 0.4 ;
    Ks     := 0.5 ;
    Yxs    := 0.5 ;
    alpha  := 0.2 ;
    beta   := 0.01 ;
    ms     := 0.02 ;
    Sf     := 400 ;     # g/L glucose in feed
  END

ASSIGN
  WITHIN R101 DO
    F := 0 ;            # batch phase, no feed
  END

INITIAL
  WITHIN R101 DO
    X = 0.1 ;           # inoculum, g/L
    S = 20 ;            # initial glucose, g/L
    P = 0 ;
    V = 2 ;             # starting volume, L
  END

SOLUTIONPARAMETERS
  ReportingInterval := 0.25 ;   # h

SCHEDULE
  SEQUENCE
    CONTINUE FOR 8                 # batch phase, h
    RESET R101.F := 0.05 ; END     # start feeding, L/h
    CONTINUE FOR 16                # fed-batch phase, h
  END
```

### Step 5: Run and check

1. Select `Sim_FedBatch` and click **Simulate** (the run button).
2. Read the execution output. Fix any errors before looking at results:
    1. Syntax errors point to the entity and line.
    2. A degrees-of-freedom error means an input is missing from ASSIGN or assigned twice.
    3. A failed initialization usually means bad initial values or bounds.
3. Open the trajectories in the results viewer (or export to Excel or CSV) and plot X, S, P and V against time.
4. Sanity checks:
    - S never goes below 0 and X rises while S falls.
    - V rises by exactly 0.05 × 16 = 0.8 L, from 2.0 to 2.8 L.
    - A carbon or mass balance closes within your expected error.
5. Save the results set with a clear name, for example `Base_case_F0.05`.

## Step 6: Parameter estimation

Fit μmax, Ks, Yxs, α, β and ms to your measured X, S and P. gPROMS uses maximum-likelihood estimation, so it also reports confidence intervals and correlations.

1. **Prepare the PROCESS for estimation.** Copy `Sim_FedBatch` to `PE_Base`. Keep the parameters to be estimated in SET with their starting guesses; gPROMS overrides them during fitting.
2. **Create one Performed Experiment per lab run** (right-click **Experiments** → **New Entity**). For each run enter:
    1. Initial conditions: X, S, P and V at time 0.
    2. Controls: the feed rate F over time, as piecewise-constant steps matching what you actually fed.
    3. Measurements: time, variable (`R101.X`, `R101.S`, `R101.P`) and measured value. Paste these from Excel.
3. **Create the Parameter Estimation entity** (right-click **Parameter Estimations** → **New Entity**) and set:
    1. The PROCESS to use: `PE_Base`.
    2. Parameters to estimate, each with starting guess, lower and upper bound (for example μmax 0.4, bounds 0.05–1.5 1/h).
    3. Experiments to include: all Performed Experiments.
    4. Measurement variance model for each sensor, for example constant variance (absolute error) for S and relative variance for X.
4. **Run** the estimation and read the report.
5. **Judge the fit:**
    - Plot model curves against data for every experiment.
    - Check each parameter's 95 % confidence interval and t-value. A t-value below the reference t means the data cannot pin that parameter down.
    - Look at the correlation matrix. Values above about 0.95 (μmax with Ks is common) mean the pair is not separately identifiable.
    - Check the lack-of-fit (weighted residual) test against its χ² reference value.
6. **If the fit is poor:** fix poorly identifiable parameters at literature values, add a richer experiment (for example a pulse of substrate), or extend the model (for example substrate inhibition).
7. **Validate** on a run that was not used for fitting.
8. Copy the final estimates back into the SET section of the simulation PROCESS.

## Step 7: Dynamic optimization of the feed profile

Find the feed rate over time that gives the most product at the end of the run, within the reactor's volume limit.

1. Copy the fitted PROCESS to `Opt_Base`.
2. Create an **Optimisation** entity (right-click **Optimisations** → **New Entity**) linked to `Opt_Base`.
3. **Objective:** maximize `R101.Ptot` at the final time.
4. **Control variable:** `R101.F`, piecewise-constant over, for example, 8 intervals of 3 h. Bounds 0–0.15 L/h.
5. **Time horizon:** fixed at 24 h, or let gPROMS treat the final time as a decision variable with bounds such as 16–36 h.
6. **Constraints:**
    - End-point: `R101.V` ≤ 4 L (working volume).
    - Path: `R101.S` ≤ 5 g/L, to avoid overflow metabolism or inhibition.
7. **Run** the optimization. Start from the base-case feed as the initial guess.
8. **Check the result:** plot the optimal F profile and the resulting X, S, P and V. Confirm all constraints are met and the profile is physically sensible (no rapid on–off switching you cannot run).
9. **Test robustness:** re-run with μmax and Yxs at the edges of their confidence intervals and check that product stays acceptable.
10. Write the optimal profile into a new simulation PROCESS (`Sim_Optimal`) using RESET steps, so the recipe can be re-run and shared.

## Troubleshooting and good practice

| Problem | Likely cause | Fix |
| --- | --- | --- |
| "Degrees of freedom not zero" | Input missing from ASSIGN, or a variable assigned and also given by an equation | Count variables and equations; assign exactly the free inputs (here F) |
| Initialization fails | Initial values far from the solution, or a division by zero (V = 0) | Set realistic INITIAL values and variable-type defaults; keep V > 0 |
| Variable hits a bound | Bounds too tight, or a real model error (S going negative) | Check the balance signs first, then widen bounds if justified |
| Integration stalls when S runs out | Monod term becomes stiff near S = 0 | Use a small positive lower bound, or a smooth switch, and tighten solver tolerances |
| Parameter estimates at their bounds | Poor identifiability or poor starting guesses | Fix some parameters, change guesses, add informative experiments |
| Optimal feed is bang-bang and unrunnable | Too many control intervals or no penalty on changes | Use fewer intervals or add a rate-of-change constraint |

**Good practice:**

- Keep units consistent (g, L, h) and write them in comments next to every parameter and variable.
- Build in stages: batch model first, then add feeding, then product, checking results each time.
- Keep separate PROCESS entities for simulation, estimation and optimization so a change to one does not break the others.
- Version the project file and record parameter sets with the date and data used.
- Document the final model, data sources and assumptions so others can reproduce the results.

*The gPROMS code here is written from general knowledge of the language; verify syntax and menu names against your version's documentation.*
