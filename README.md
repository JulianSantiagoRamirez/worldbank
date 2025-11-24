# Theta Estimations Combine kstock.dofile Explanation

This ReadMe.dmd file describes line by line the do file gave to me by Kee.

## 1. Load Data and basic cleaning


Creating temporary files and loading the main dataset.

```stata
tempfile temp1 temp2
use Ready2DVAR_combinekstock.dta
```

Then, removes the proccessing trade dummy variable interacted with imports. Rename import scope, and capital stock variables. Finally, ensure that the panel is ordered

```stata
drop procimp
rename scope_imp_hs8 impvariety
rename kstock90 kstock
rename kstock95 kkstock95
sort firmid year
```
## 2. Normalize capital and k-stock variables

Handdle missing values and apply log transform avoiding zero values 

```stata
inspect capital kstock
replace kstock=0 if kstock==.

replace capital=ln(capital+1)
replace kstock=ln(kstock+1)
```

## 3. Industry identifiers

Converts industry names to numeric codes, and create the industry x year combinations

```stata
tab group
egen ind=group(group)
egen indyr=group(ind year)
```

## 4. Fix proc inconsistencies

If proc is incorrectly coded for a year this codes corrects it.

```stata
sort firmid year
by firmid: replace proc=proc[_n-1] if switch == 1 & proc[_n-1]==proc[_n+1] & proc~=proc[_n-1] & year-year[_n-1]==1
```

## 5. Fix abnormal jumps in imports/exports

Compute growth rates and drop outliers.

```stata
foreach var of varlist imp exp{
gen ln`var'=ln(`var')
by firmid: gen dln`var'=ln`var'-ln`var'[_n-1] if year-year[_n-1]==1
}
```

## 6. Winsorize imports and exports 

Within yearxindustry winsorize imports and exports. Dropping small values

```stata
winsor2 imp, by(year group ) cut(2.5 97.5) trim
rename imp imporiginal
rename imp_tr imp

winsor2 exp, by(year group ) cut(2.5 97.5) trim
rename exp imporiginal
rename exp_tr imp

replace imp=. if imp<2000
replace exp=. if exp<2000
```

## 7. Additional corrections

Eliminate firms with inconsistent import/export ratios

```stata
gen check=imp/exp
winsor2 check, by(year group ) cut(1 99) trim
replace imp=. if check_tr==.
```


## 8. Polynomial Terms

Create squares and cubes for the capital variables and import weighted by specific exchange rate

```stata
foreach var of varlist capital imp_xr kstock{
gen `var'2=`var'*`var'
gen `var'3=`var'*`var'*`var'
}
```
Interaction between capital variables and imp_xr

```stata
foreach var of varlist capital imp_xr{
gen `var'kstock=`var'*kstock
gen `var'2kstock=`var'2*kstock
gen `var'kstock2=`var'*kstock2
}

gen capitalimp_xrkstock=capital*imp_xr*kstock
```

## 9. Baseline Regressions

-  Baseline Regression
-  Separate regressions for proc=0 vs proc=1
-  Adding controls

```stata
reghdfe exp imp, absorb(firmid year)
....
```

## 10. Theta estimation by industry

Run industry-specific regressions and compute theta = β_imp + β_procimp.

```stata
sum ind
local max=_result(6)
local i=1
while `i'<=`max'{
reghdfe exp imp procimp capital* imp_xr* kstock* if ind==`i', ...
lincom _b[imp]+_b[procimp]-1
local i=`i'+1
}
```

## 11. OP estimation

Create firmxproc panel

```stata
egen fid=group(firmid proc)
gen dvarop=.
gen thetaop=.
```
Loop over industries. Compute OP-based theta 
```stata
reghdfe exp imp capital* imp_xr* kstock* ...
replace thetaop=(_b[imp])
replace dvarop=(exporiginal-thetaop*imporiginal)/exporiginal
replace dvarop=. if dvarop<0
```

## 12. ACF estimation

Perform ACF estimation, separating by proc and loops over industries.
```stata
gen inv=capital
acfest exp , free(imp) state(kstock imp_xr) proxy(inv) i(fid) t(year) invest nbs(50) robust
```

## 13. Collapse to industry-year averages (weighted)

Weighted averages by industry-year.


```stata
egen expindacf=sum(exporiginal) if dvaracf~=., by(indyr proc)
gen expshacf=exporiginal/expindacf
gen wdvaracf=expshacf*dvaracf
egen avedvaracf=sum(wdvaracf), by(indyr proc)
replace dvaracf=avedvaracf if dvaracf==.
```


# Key Variables from the dataset

- *firmid*:Firm identifier. Critical for panel structure.
- *year*:	Year of observation.
- *exp*:	Exports (value).
- *imp*:	Imports (value).
- *capital*:	Firm-level capital import.
- *kstock90 / kstock95*:	Capital sotck depreciate rate 10%/5%.
- *imp_xr*:	Imports adjusted by exchange rate.
- *proc*:	Processing trade dummy.
- *switch*:	Indicator of firms change regime across year.
- *group*:	Industry group string.
- *procimp*:	Interaction: proc * imp (created later).
- *impvariety*:	Formerly scope_imp_hs8 = import variety count.
- *ind*:	Numeric industry identifier (after egen).
- *indyr*:	Industry-year group (used for clustering).





