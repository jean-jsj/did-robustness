![Event‐Study-Plots](07_heterogeneity_robust_estimators/event_study_plot.jpg)

In settings with staggered treatment timing—where different units receive treatment at different times—the standard TWFE (two-way fixed effects) estimator produces unbiased estimates only if treatment effects are constant across time and units. A key problem is that TWFE can compare treated units to those already treated, treating them as if they were untreated. When treatment effects vary, these “forbidden comparisons” introduce bias (de Chaisemartin and D’Haultfœuille 2020, Goodman-Bacon 2021).

To address this, researchers often rely on alternative estimators that avoid these problematic comparisons. These methods differ in how they construct control groups and how they weight cohort-time comparisons:
* Callaway and Sant’Anna (2021)
* De Chaisemartin and d’Haultfoeuille (2020)
* Sun and Abraham (2021)
* Borusyak et al. (2024)

## Procedure
### Stata Code
```stata
/******************************************************************************
Variable Definition

treat_timing: week of treatment event
tg: treatment_i
p_lag_*, p_lead_*: time-relative dummy variables (lags and leads)
******************************************************************************/

local max_lag 9 // Number of weeks after treatment
local max_lead 10 // Number of weeks before treatment

gen cs_treat = treat_timing // Callaway & Sant'Anna (2021): set to 0 for never-treated units
replace cs_treat = 0 if tg==0
gen es_treat = treat_timing // Sun & Abraham (2021): set to missing for never-treated units
replace es_treat = . if tg==0
gen never_treat = (es_treat==.) // indicator for never-treated units

*--------------------------------------------------------------
* Callaway & Sant'Anna (2021) 
*--------------------------------------------------------------
csdid dv, ivar(id) time(week) gvar(cs_treat)
csdid_estat event, estore(cs)

*--------------------------------------------------------------
* de Chaisemartin & D'Haultfoeuille (2020) 
*--------------------------------------------------------------
did_multiplegt_dyn dv id week inter, effects(`max_lag') placebo(`max_lead') cluster(id)
matrix dcdh_b = e(estimates)
matrix dcdh_v = e(variances)

*--------------------------------------------------------------
* Sun & Abraham (2021)
*--------------------------------------------------------------
eventstudyinteract dv p_lag_* p_lead_*, vce(cluster id) absorb(id week) cohort(es_treat) control_cohort(never_treat)
matrix sa_b = e(b_iw)
matrix sa_v = e(V_iw)

*--------------------------------------------------------------
* Borusyak, Jaravel, & Spiess (2024)
*--------------------------------------------------------------
did_imputation dv id week es_treat, fe(id week) horizons(0/`max_lag') pretrend(`max_lead')
estimates store bjs

event_plot bjs dcdh_b#dcdh_v cs sa_b#sa_v, ///
	stub_lag(tau# Effect_# Tp# p_lag_#) stub_lead(pre# Placebo_# Tm# p_lead_#) plottype(scatter) ciplottype(rcap) ///
	together perturb(-0.3(0.1)0.3) trimlead(`max_lead') trimlag(`max_lag') noautolegend graph_opt(title("DV", size(medlarge)) ///
	xtitle("Weeks since Treatment") ytitle("DID Estimate and 95% C.I.") xlabel(-`max_lead'(1)`max_lag') ylabel(-0.6(0.2)0.8) ///
	legend(order(1 "Borusyak et al." 3 "de Chaisemartin-D'Haultfoeuille" 5 "Callaway-Sant'Anna" 7 "Sun-Abraham") rows(3) position(6) region(lc(black))) ///
	xline(-1, lcolor(gs8) lpattern(dash)) yline(0, lcolor(gs8)) graphregion(color(white)) bgcolor(white) ylabel(, angle(horizontal))) ///
	lag_opt1(msymbol(O) color(cranberry)) lag_ci_opt1(color(cranberry)) ///
	lag_opt2(msymbol(Dh) color(gray)) lag_ci_opt2(color(gray)) ///
	lag_opt3(msymbol(Th) color(gold)) lag_ci_opt3(color(gold)) ///
	lag_opt4(msymbol(Sh) color(navy)) lag_ci_opt4(color(navy)) 

graph export "event_study_plots.jpg", replace
