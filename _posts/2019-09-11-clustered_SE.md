---
layout: post
title: Clustered standard errors - plm vs felm
---

In a previous blog I mentioned the possibility to cluster on more than one level or to cluster on supra-group levels. Whereas Stata has long been known to provide clustering options on many of its commands, R is quickly catching up, especially in panel models. In fact, automated two-way clustering (as well as two-fixed effects) are only possible in R! In this blog, I will compare two R commands (*plm* and *felm*) that allow flexible clustering options for fixed effects models. Code (R) and data (randomly generated) to reproduce the results are provided on GitHub.

The *plm* package with its identically named workhorse function is perhaps the most well-known panel command in R. It allows for two-way clustering using the *effect="twoway"* option. To adjust the standard errors using clustering, one needs to use the *vcovHC* (single clustering) or *vcovDC* (double clustering) commands. One drawback is the restriction to cluster on either the group or time level (or both). Higher-level clustering is (currently) not supported by *plm*. Fortunately, adapting the existing code to allow for higher-level clustering is relatively easy (see vcovCL.R on GitHub).  

The *felm* command (from the *fle* package) does allows for double (or even multi-way) clustering. One difficulty in comparing both commands lies in their degree of freedom correction: *plm* offers different corrections based on MacKinnon and White (1985), *felm* only offers a Stata-like df correction (![](https://www.stata.com/support/faqs/statistics/robust-standard-errors/)). 

    Dit is code
    En dit ook
    
ddddd
