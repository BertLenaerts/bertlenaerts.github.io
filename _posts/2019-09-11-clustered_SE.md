---
layout: post
title: Clustered standard errors - plm vs felm
---

In a previous blog I mentioned the possibility to cluster on more than one level or to cluster on supra-group levels. Whereas Stata has long been known to provide clustering options on many of its commands, R is quickly catching up, especially in panel models. In fact, automated two-way clustering (as well as two-fixed effects) are only possible in R! In this blog, I will compare two R commands (*plm* and *felm*) that allow flexible clustering options for fixed effects models. Code (R) and data (randomly generated) to reproduce the results are provided on GitHub.

The *plm* package with its identically named workhorse function is perhaps the most well-known panel command in R. It allows for two-way clustering using the *effect="twoway"* option. To adjust the standard errors using clustering, one needs to use the *vcovHC* (single clustering) or *vcovDC* (double clustering) commands. One drawback is the restriction to cluster on either the group or time level (or both). Higher-level clustering is (currently) not supported by *plm*. Fortunately, adapting the existing code to allow for higher-level clustering is relatively easy (see vcovCL.R on GitHub).  

The *felm* command (from the *fle* package) does allows for double (or even multi-way) clustering. One difficulty in comparing both commands lies in their degree of freedom correction: *plm* offers different corrections based on MacKinnon and White (1985), *felm* only offers a Stata-like df correction ([link](https://www.stata.com/support/faqs/statistics/robust-standard-errors/)). 

Let's start with a dataset that allows for time, group and higher-level clustering:

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
    
Note the difference in sytax between *plm* and *felm* in specifying the exogenous, fixed and cluster variables. Since we have two-way fixed effects, the number of covariates (often denoted k) is equal to 15 and not 16 since the intercept (which can be left out if all fixed effects are added) can only be left out once. The *HC0* option indicates no df correction is carried out, while *stata=T* performs the same correction as *felm*. 

    # TIME CLUSTER
    coeftest(plm3, vcov=(40/(41-14+1-2))*(5/4)*vcovHC(plm3, type="HC0", cluster="time"))
    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$year, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | year, d))

    # HIGHER-LEVEL CLUSTER
    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$gid, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | gid, d))
    
 For higher-level clustering, the original code from *vcovHC* needs to be adapted (see vcovCL.R). 

    # TWOWAY CLUSTERING
    coeftest(plm3, vcov=( (40/(41-14+1-2))*(9/8)*vcovCL(x=plm3, cluster=d$id)+
                             (40/(41+1-14-2))*(5/4)* vcovCL(x=plm3, cluster=d$year)-
                             (40/(41-14+1-2))*(41/40)*vcovHC(plm3, type="HC0", method = "white1") ) )
    coeftest(plm3, vcov=vcovTC(x=plm3, d$year, d$id, stata=T))
    summary(felm(y ~ u | id+year | (x ~ e) | (id+year), d))
    
Two-way clustering is carried out by first clustering on both the group and time level individually and summing the variance-covariane matrices, and second by substracting a correction term. The *vcovDC* command uses as a correction term the usual (White) heteroskedasticity-robust variance matrix (Thompson, 2011). The correction implemented by *felm* is suggested by Cameron et al. (2011). That is, the group and time cluster are intersected to generate a third cluster variable. If the group-time combinations are unique, this approach is equivalent to White's heteroskedasticity-robust correction. Since our dataset *d* contains only unique id-year observations, we can check this:

    vcovCL(x=plm3, cluster=d$idyear)
    vcovHC(plm3, type="HC0", method = "white1")

The *vcovDC* command implements the Thompson (2011) approach since it does not allow non-unique group-time combinations (opposite to *felm*). Two disadvantages of the *vcocDC* command are that it (i) does not allow manual Stata-like df correction, and (ii) can generate negative variances. Similar to the *vcovHC* command, *vcovDC* allows MacKinnon and White (1985) df corrections but not a Stata-like correction. Since two-way clustering consists of the sum of three terms, term-specific Stata-like df corrections are needed and *vcovDC* does not facilite this. Cameron et al. (2011) suggest a Stata-like df of freemdom correction where the number of unique clusters formed by the intersection of the group and time level are used for the correction term. Second, the Thompson (2011) approach can generate negative variances if the White variances are larger than the sum of the group- and time-wise clustered variances. This is the case for our sample if we do not correct for the degrees of freedom:

    coeftest(plm3, vcov=vcovDC(plm3, type="HC0"))
    coeftest(plm3, vcov=vcovTC(x=plm3, d$year, d$id, stata=F))
 
**References**

Cameron, A.C., Gelbach, J.B., Miller, D.L., 2011. Robust inference with multiway clustering. J. Bus. Econ. Stat. 29, 238–249.  
MacKinnon, J.G., White, H., 1985. Some heteroskedasticity-consistent covariance matrix estimators with improved finite sample properties. J. Econom. 29, 305–325.  
Thompson, S.B., 2011. Simple formulas for standard errors that cluster by both firm and time. J. Financ. Econ. 99, 1–10.  
