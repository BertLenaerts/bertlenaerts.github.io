---
layout: post
title: Clustered standard errors - R vs Stata
---

**Updated version: January 5th, 2021**

*A previous version of this blog was written on September 11th, 2019 using R version 3.6. In more recent versions, however, the R package lfe no longer works properly and has been removed from the CRAN repository on December 4th, 2020 ([link](https://cran.r-project.org/web/packages/lfe/index.html)).*

In a previous blog I mentioned the possibility to cluster on more than one level or to cluster on supra-group levels. Whereas Stata has long been known to provide clustering options on many of its commands, R is quickly catching up, especially for panel models. In fact, automated two-way clustering (as well as two-way fixed effects) are only possible in R. In this blog, I will compare two R commands (*plm* and *felm*) and the equivalent commands in Stata that allow flexible clustering options for fixed effects models. Code (R and Stata) and an example dataset to reproduce the results are provided [here](https://github.com/BertLenaerts/blogfiles).

The *plm* package with its identically named command is perhaps the most well-known panel command in R. It allows for two-way fixed-effects using the *effect="twoway"* option. To adjust the standard errors using clustering, one needs to use the *vcovHC* (single clustering) or *vcovDC* (double clustering) commands. One drawback is the restriction to cluster on either the group or time level (or both). Higher-level clustering is (currently) not supported by *plm*. Fortunately, adapting the existing code to allow for higher-level clustering is relatively easy (see vcov_plm.R on GitHub for the commands *vcovHR*, *vcovCL* and *vcovTC*).  

The *felm* command (from the *fle* package) does allows for double (or even multi-way) clustering. One difficulty in comparing both commands lies in their degree of freedom (df) correction: *plm* offers different corrections based on MacKinnon and White (1985), whereas *felm* only offers a Stata-like df correction ([link](https://www.stata.com/support/faqs/statistics/robust-standard-errors/)). Since *plm* does not allow a Stata-like df correction, I rewrote the command *vcovHC* into three separate commands that do: *vcovHR*, *vcovCL* and *vcovTC*.

Let's start with a dataset that allows for time, group and higher-level clustering. To load the dataset into R:

    d = read_csv("d.csv") %>%
        mutate(idyear = as.numeric(paste0(id, year)))
    
The equivalent commands in Stata are:

    import delimited "d.csv", clear 
    qui tab id, gen(k)
    xtset id year

This example data contains 41 observations, 9 group levels (id), 5 time levels (year), 3 cluster levels (gid), and 2 independent variables (x and u). In this example, the group panels (id) are nested within the clusters (gid).

The uncorrected standard errors for a two-way fixed-effects model in R are:

    plm3=plm(y~x + u,
             data=d,
             model="within", 
             effect="twoway", 
             index=c("id", "year"))

    summary(plm3)
    summary(felm(y ~ u + x| id+year | 0 | 0, d))
    
and in Stata:

    reg y u x i.year k*
    xtreg y u x i.year, fe
    
Note (i) the difference in syntax between *plm* and *felm* in specifying the exogenous, fixed and cluster variables, and (ii) that in Stata two-way fixed effects are not automated (i.e. either the time- or group-fixed have to be added manually as dummies).

The heteroskedasticity-consistent (White's) standard errors in R can be obtained by:

    coeftest(plm3, vcov=(41/(41-14+1-2))*vcovHC(plm3, type="HC0", method = "white1"))
    coeftest(plm3, vcovHR(plm3))
    summary(felm(y ~ u + x| id+year | 0 | 0, df), robust = TRUE)

and in Stata by:

    reg y u x i.year k*, robust

The command *vcovHR* is essentially a wrapper of the *vcovHC* command using a Stata-like df correction. In Stata, the *robust* option only delivers HC standard erros in non-panel models. In panel models, it delivers clustered standard errors instead.

Clustering can be done at different levels (group, time, higher-level), both at a single or mutiple levels simultaneously. In R, clustering at the group level can be done as follows:

    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$id))
    summary(felm(y ~ u + x| id+year | 0 | id, d))
    
and in Stata by:

    reg y u x k* i.year, vce(cluster id)
    xtreg y u x i.year, fe cluster(id) dfadj

Since we have two-way fixed effects, the number of covariates (often denoted k) here is equal to 15 and not 16 since the intercept (which can be left out if all fixed effects are added) can only be left out once. Note that the *dfadj* option is needed to ensure the appropriate df correction is carried out.

Analogously, clustering can be done at the time level:

    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$year))
    summary(felm(y ~ u + x| id+year | 0 | year, d))

    reg y u x k* i.year, vce(cluster year)
    * no xtreg example as the time panels are not nested within clusters in this example
    
or at a higher level:

    coeftest(plm3, vcov=vcovCL(x=plm3, cluster=d$gid))
    summary(felm(y ~ u + x| id+year | 0 | gid, d))

    reg y u x k* i.year, vce(cluster gid)
    xtreg y u x i.year, fe cluster(gid) dfadj
    
R also offers the option of two-way clustering (unlike Stata):

    coeftest(plm3, vcov=vcovDC(plm3, type="HC0"))
    coeftest(plm3, vcov=vcovTC(x=plm3, d$year, d$id))
    summary(felm(y ~ u + x| id+year | 0 | (id+year), d))
    
Two-way clustering is carried out by first clustering on both the group and time level individually and summing the variance-covariane matrices, and second by substracting a correction term. The *vcovDC* command (part of *plm*) uses as a correction term the usual (White) heteroskedasticity-robust variance matrix (Thompson, 2011). The correction implemented by *felm* is suggested by Cameron et al. (2011). That is, the group and time variable are intersected to generate a third cluster variable. If the group-time combinations are unique, this approach is equivalent to White's heteroskedasticity-robust correction. 

Lastly, the Thompson (2011) approach can generate negative variances if the White variances are larger than the sum of the group- and time-wise clustered variances. This is the case for our sample with two-way fixed effects and explains why *coeftest(vcovTC)* and *felm* do not report the same result.

**References**

Cameron, A.C., Gelbach, J.B., Miller, D.L., 2011. Robust inference with multiway clustering. J. Bus. Econ. Stat. 29, 238–249.  
MacKinnon, J.G., White, H., 1985. Some heteroskedasticity-consistent covariance matrix estimators with improved finite sample properties. J. Econom. 29, 305–325.  
Thompson, S.B., 2011. Simple formulas for standard errors that cluster by both firm and time. J. Financ. Econ. 99, 1–10.  
