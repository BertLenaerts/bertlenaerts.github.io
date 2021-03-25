
---
layout: post
title: Reviving correlation
---

Almost a year a blog post appeared on Towards Data Science suggesting an alternative to the common correlation coefficient called the Predictive Power Score (PPS). The PPS is promoted for being able to measure nonlinear relationships (asymmetrically) for both numeric and categorical variables. Without going into the technical details behind the PPS algorithm, I was wondering whether an extension to the classic correlation setup couldn’t produce similar (or even better) results. 
To extend the normal correlation coefficient, I relied on the related but less-known metric of multiple correlation. The multiple correlation coefficient is simply the square root of the coefficient of determination from an OLS regression on one or more covariates (features). This modest extension already allows to include categorical features (as dummies) and measure nonlinear relationships (for example, by using a polynomial functional form for a single feature). To address the problem of categorical targets (responses), I suggest to regress every single level and take the weighted average across all multiple correlation coefficients (a multinomial probit might be another option). To minimise computation time, a cardinality threshold can be defined. 
The original blog post compared the classic correlation matrix to the PPS matrix on the Titanic dataset (which can be downloaded from the Kaggle website). In this blog post, I'll compare the multiple correlation matrix to the previously proposed PPS matrix. 

