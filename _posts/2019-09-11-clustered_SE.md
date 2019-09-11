---
layout: post
title: Clustered standard errors - plm vs felm
---

In a previous blog I mentioned the possibility to cluster on more than one level or to cluster on supra-group levels. Whereas Stata has long been known to provide clustering options on many of its commands, R is quickly catching up, especially in panel models. In fact, automated two-way clustering (as well as two-fixed effects) are only possible in R! In this blog, I will compare two R commands (*plm* and *felm*) that allow flexible clustering options for fixed effects models. Code (R) and data (randomly generated) to reproduce the results are provided on GitHub.

The *plm* package with its identically named workhorse function is perhaps the most well-known panel command in R. It allows for two-way clustering using the *effect="twoway"* option. To adjust the standard errors using clustering, one needs to use the *vcovHC* (single clustering) or *vcovDC* (double clustering) commands. One drawback is the restriction to cluster on either the group or time level (or both). Higher-level clustering is (currently) not supported by *plm*. Fortunately, adapting the existing code to allow for higher-level clustering is relatively easy (see vcovCL.R on GitHub).  

The *felm* command (from the *fle* package) does allows for double (or even multi-way) clustering. One difficulty in comparing both commands lies in their degree of freedom correction: *plm* offers different corrections based on MacKinnon and White (1985), *felm* only offers a Stata-like df correction ([link](https://www.stata.com/support/faqs/statistics/robust-standard-errors/)). 

Let' start with a dataset that allows for time, group and higher-level clustering:

    d = read_csv("d.csv") %>%
        mutate(idyear = as.numeric(paste0(id, year)))
        
This example data contains 41 observations, 9 group levels (id), 5 time levels (year), 3 cluster levels (gid), 2 independent variables (x and u), of which x is endogenous, and 1 instrument (e). 

Using this example, we can easily compare *plm* and *felm* if we correct for the degrees of freedom manually:

    library(plm)
    library(lfe) 
    library(lmtest)
    library(dplyr)
    library(readr)
    source("vcovCL.R")
    
    ## TWOWAY FIXED EFFECTS

    plm3=plm(y~x +u | u+e,
             data=d,
             model="within", 
             effect="twoway", 
             index=c("id", "year"))

    # GROUP CLUSTER
    coeftest(plm3, vcov=(40/(41-14+1-2))*(9/8)*vcovHC(plm3, type="HC0", cluster="group"))
    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$id, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | id, d))
    
Note the difference in sytax between *plm* and *felm* in specifying the exogenous, fixed and cluster variables. Since we have two-way fixed effects, the number of covariates (often denoted k) is equal to 15 and not 16 since the intercept (which can be left out if all fixed effects are added) can only be left out once. The HC0 option indicates no df correction is carried out, while stata=T performs the same correction as *felm*. 

    # TIME CLUSTER
    coeftest(plm3, vcov=(40/(41-14+1-2))*(5/4)*vcovHC(plm3, type="HC0", cluster="time"))
    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$year, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | year, d))

    # NO CLUSTER (no HC too)
    summary(plm3)
    summary(felm(y ~ u | id+year | (x ~ e) | 0, d))

    # HC COREECTION
    coeftest(plm3, vcov=(40/(41-14-2))*vcovG(plm3, type = "HC0", l = 0, inner = "white"))
    coeftest(plm3, vcov=(40/(41-14-2))*vcovHC(plm3, type="HC0", method = "white1"))

    # HIGHER-LEVEL CLUSTER
    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$gid, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | gid, d))
    
 For higher-level clustering, the original code from vcovHC needs to be adapted (see vcovCL.R). 

    # TWOWAY CLUSTERING
    coeftest(plm3, vcov=vcovTC(x=plm3, d$year, d$id, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | (id+year), d))
    
Although *plm* has a double clustering command (*vcovDC*), it does not allow for a Stata-like df correction. The correction implemented by *felm* uses correction suggested by Cameron et al. (2011). That is, the group and time cluster are intersected to generate
a third cluster variable; if the id-year combos are unique then this is equivalent
to White's heteroskedasticity-robust correction
since felm allows non-unique year-id combos (and also allows for multi-way clustering),
the third variable method is implemented, whereas plm uses the White method
the df correction for the substraction term takes the number of clusters for the
third variable for felm whereas plm uses the standard White HC df correction
