## Procedure
* Randomly assign a “placebo” treatment to periods before the actual treatment event
* Estimate the treatment effect for each of N draws
* Plot the kernel density of these N placebo ATTs, where the x-axis represents the beta estimate values, while the y-axis shows the corresponding probability densities

### Stata Code
```stata
/******************************************************************************
Variable Definition

lead_lag: number of weeks before or after treatment event 
id: user id
tg: treatment_i
inter: treatment_it 
dv: dependent variable
week: week
******************************************************************************/

set seed 12 // set random seed for reproducibility
local pre_period -12 // Earliest lead_lag value used for placebo assignment
local reps = 1000 // set number of placebo iterations

use "data.dta", clear
reghdfe dv inter, absorb(id week) cluster(id) // run main regression to estimate actual ATE
scalar orig_beta = _b[inter]
keep if lead_lag < 0 // keep only pre-treatment observations for the placebo test

gen placebo_beta = .
forvalues i = 1/`reps' {
	gen placebo_random = .
	replace placebo_random = runiformint(`pre_period',0) // randomly assign a "placebo" treatment event
    gen placebo_post = (lead_lag >= placebo_random)
    gen placebo_inter = placebo_post * tg
    reghdfe dv placebo_inter, absorb(id week) cluster(id)
    replace placebo_beta = _b[placebo_inter] in `i'
    drop placebo_random placebo_post placebo_inter
}

sum placebo_beta, detail
gen p_value = sum(placebo_beta >= orig_beta) / `reps'
local orig_beta_value = orig_beta  // save original beta as a local macro for plotting
histogram placebo_beta, normal percent title("Outcome") ytitle("") xtitle("Placebo Beta") ///
    xline(`orig_beta_value', lcolor(red)) xlabel(, angle(45)) // plot distribution of placebo estimates

graph export "placebo_distribution.jpg", replace
```

## What to look for
* Distribution of placebo treatment effects tightly centered around zero indicates no systematic pre-treatment effect
