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






