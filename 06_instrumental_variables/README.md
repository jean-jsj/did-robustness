## First Stage
For binary 〖Treatment〗_it, report pseudo-R² (McFadden’s R²) and examine whether the value is well within acceptable limits. According to McFadden (1979), pseudo-R² value between 0.2 and 0.4 indicate excellent fit, and simulation studies (Domencich and McFadden 1975) suggest this range is comparable to an OLS R² between 0.7 and 0.9.

## Exclusion Restriction Checks
Report two placebo test results. Examine whether the correlation is small and statistically insignificant in both cases:
* Regress outcome on IV in the never-treated group. Exclude the treated group to isolate the impact of the IV without the confounding effect of treatment. 
* Regress outcome on IV during pre-treatment periods. This is to ensure that there were no pre-existing trends between the IV and the outcome. 

## Second Stage
Estimate a series of DiD models that differently address potential selection bias. Examine whether the included covariates are statistically insignificant:
* Model 1: include the predicted probability of treatment from a first-stage probit regression as a covariate.
* Model 2: implement a Heckman-style correction using the inverse Mills ratio (IMR), under the assumption of joint normality between the error terms.
* Model 3: adopt the semiparametric correction method by Helpman, Melitz, and Rubinstein (2008), relaxing the joint normality assumption by including polynomial terms of the predicted probability and the IMR.

### Stata Code
```stata
/******************************************************************************
Variable Definition

id: user id
tg: treatment_i
post: indicator for post-treatment weeks
inter: treatment_it 
dv: dependent variable
iv: instrumental variable
week: week
******************************************************************************/

*--------------------------------------------------------------
* First Stage
*--------------------------------------------------------------
probit inter iv [include covariates], cluster(id)
predict phat, xb // predicted probability of treatment
gen mills = normalden(phat) / normal(phat) // inverse mills ratio (IMR)
gen mills_1 = (mills+phat)
gen mills_2 = (mills+phat)^2
gen mills_3 = (mills+phat)^3

*--------------------------------------------------------------
* Exclusion Restriction Checks
*--------------------------------------------------------------
reghdfe dv iv if tg==0, absorb(week id) cluster(id) // Regress outcome on IV in the never-treated group
reghdfe dv iv if post==0, absorb(week id) cluster(id) // Regress outcome on IV during pre-treatment periods

*--------------------------------------------------------------
* Second Stage
*--------------------------------------------------------------
reghdfe dv inter, absorb(week id) cluster(id) // baseline 
bs: reghdfe dv inter phat, absorb(week id) cluster(id) // model 1, standard errors bootstrapped
bs: reghdfe dv inter mills, absorb(week id) cluster(id) // model 2, standard errors bootstrapped
bs: reghdfe dv inter mills mills_1 mills_2 mills_3, absorb(week id) cluster(id) // model 3, standard errors bootstrapped
```