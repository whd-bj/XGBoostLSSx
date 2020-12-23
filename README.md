# XGBoostLSS - An extension of XGBoost to probabilistic forecasting
We propose a new framework of XGBoost that predicts the entire conditional distribution of a univariate response variable. In particular, **XGBoostLSS** models all moments of a parametric distribution, i.e., mean, location, scale and shape (LSS), instead of the conditional mean only. Choosing from a wide range of continuous, discrete and mixed discrete-continuous distribution, modeling and predicting the entire conditional distribution greatly enhances the flexibility of XGBoost, as it allows to gain additional insight into the data generating process, as well as to create probabilistic forecasts from which prediction intervals and quantiles of interest can be derived. In its current implementation, **XGBoostLSS** is available in *R* and extensions to *Julia* and *Python* are in progress. 

## Motivation
Many regression / supervised models focus on the conditional mean only, implicitly treating higher moments of the conditional distribution as fixed nuisance parameters. This assumption, however, of constant higher moments not changing as functions of covariates is a stark one and is only valid in situations where the user is privileged with dealing with data generated by a symmetric Gaussian distribution with constant variance. In real world situations however, the data generating process is usually less well behaved, exhibiting characteristics such as heteroscedasticity, varying degrees of skewness and/or kurtosis. If the user sticks to his/her assumption of not modeling all characteristics of the data, inference as well as uncertainty assessments, such as confidence and predictions intervals, are at best invalid. In this context, the introduction of *Generalised Additive Models for Location Scale and Shape (GAMLSS)* has stimulated a lot of research and culminated in a new branch of statistics that focuses on modeling the entire conditional distribution as functions of covariates. Looking at the literature of more recent computer science and machine learning, however, the main focus has been on prediction accuracy and estimation speed. In fact, even though many recent machine learning approaches, like random forest or gradient boosting-type algorithms, outperform many statistical approaches when it comes to prediction accuracy, the output/forecast of these models provides information about the conditional mean only. Hence, these models are rather reluctant to reveal other characteristics of the (predicted) distribution and fall short in applications where a probabilistic forecasts is required, e.g., for assessing estimation/prediction uncertainty in form of confidence/prediction intervals.

In the era of increasingly awareness that the output of black box prediction models needs to be turned into explainable insights, our approach extends recent machine learning models to account for all distributional properties of the data. In particular, we focus on XGBoost, which has gained much popularity and attention over the last years and has arguably become among the most widely used tools in practical data science. We term our model **XGBoostLSS**, as it combines the accuracy and speed of XGBoost with the flexibility of GAMLSS that allow for the estimation and prediction of the full conditional distribution. While retaining the speed of estimation and accuracy of XGBoost, **XGBoostLSS** allows the user to chose from a wide range of continuous, discrete and mixed discrete-continuous distributions to better adapt to the data at hand, as well as to provide predictive distributions, from which prediction intervals and quantiles can be derived. Furthermore, all XGBoost additions, such as partial dependent plots or the recently added SHAP (SHapley Additive exPlanations) approach that allows to explain the output of any machine learning model, are still applicable, with the additional advantage that they can be applied to all distributional parameter. As such, **XGBoostLSS** is intended to further close the gap between machine learning/computer science and statistics, so that models focusing mainly on prediction can also be used to describe and explain the underlying data generating process of the response of interest.

In the following, we provide a short walk-through of the functionality of **XGBoostLSS**. 

## Simulation

We simulate a data set w

```r
s = "Python syntax highlighting"
print s
```


## References
März, Alexander (2019). *XGBoostLSS - An extension of XGBoost to probabilistic forecasting*. Under Review, pp. 1–26.
