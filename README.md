# XGBoostLSS - An extension of XGBoost to probabilistic forecasting
We propose a new framework of XGBoost that predicts the entire conditional distribution of a univariate response variable. In particular, **XGBoostLSS** models all moments of a parametric distribution, i.e., mean, location, scale and shape (LSS), instead of the conditional mean only. Choosing from a wide range of continuous, discrete and mixed discrete-continuous distribution, modeling and predicting the entire conditional distribution greatly enhances the flexibility of XGBoost, as it allows to gain additional insight into the data generating process, as well as to create probabilistic forecasts from which prediction intervals and quantiles of interest can be derived. 

![Optional Text](../master/plots/XGBoostLSS.png)

## Introduction
Many regression / supervised models focus on the conditional mean only, implicitly treating higher moments of the conditional distribution as fixed nuisance parameters. This assumption, however, of constant higher moments not changing as functions of covariates is a stark one and is only valid in situations where the user is privileged with dealing with data generated by a symmetric Gaussian distribution with constant variance. In real world situations however, the data generating process is usually less well behaved, exhibiting characteristics such as heteroscedasticity, varying degrees of skewness and/or kurtosis. If the user sticks to his/her assumption of not modeling all characteristics of the data, inference as well as uncertainty assessments, such as confidence and predictions intervals, are at best invalid. In this context, the introduction of [Generalised Additive Models for Location Scale and Shape (GAMLSS)](https://www.gamlss.com/) has stimulated a lot of research and culminated in a new branch of statistics that focuses on modeling the entire conditional distribution as functions of covariates. Looking at the literature of more recent computer science and machine learning, however, the main focus has been on prediction accuracy and estimation speed. In fact, even though many recent machine learning approaches, like random forest or gradient boosting-type algorithms, outperform many statistical approaches when it comes to prediction accuracy, the output/forecast of these models provides information about the conditional mean only. Hence, these models are rather reluctant to reveal other characteristics of the (predicted) distribution and fall short in applications where a probabilistic forecasts is required, e.g., for assessing estimation/prediction uncertainty in form of confidence/prediction intervals.

In the era of increasingly awareness that the output of black box prediction models need to be turned into explainable insights, our approach extends recent machine learning models to account for all distributional properties of the data. In particular, we focus on [XGBoost](https://github.com/dmlc/xgboost), which has gained much popularity and attention over the last years and has arguably become among the most widely used tools in practical data science. We term our model **XGBoostLSS**, as it combines the accuracy and speed of XGBoost with the flexibility of GAMLSS that allow for the estimation and prediction of the full conditional distribution. While retaining the speed of estimation and accuracy of XGBoost, **XGBoostLSS** allows the user to chose from a wide range of continuous, discrete and mixed discrete-continuous distributions to better adapt to the data at hand, as well as to provide predictive distributions, from which prediction intervals and quantiles can be derived. Furthermore, all XGBoost additions, such as partial dependent plots or the recently added [SHAP (SHapley Additive exPlanations)](https://github.com/slundberg/shap) approach that allows to explain the output of any machine learning model, are still applicable, with the additional advantage that they can be applied to all distributional parameter. As such, **XGBoostLSS** is intended to further close the gap between machine learning/computer science and statistics, so that models focusing mainly on prediction can also be used to describe and explain the underlying data generating process of the response of interest.

In the following, we provide a short walk-through of the functionality of **XGBoostLSS**. 

## Examples

### Simulation

We start with a simulated a data set that exhibits heteroscedasticity where the interest lies in predicting the 5% and 95% quantiles. The data is simulated as follows: 

```r
# Simulate
set.seed(123)
quant_sel <- c(0.05, 0.95)
n <- 10000
p <- 10
x <- runif(n)
y <- rnorm(n, mean = 10, sd = 1 + 4*(0.3 < x & x < 0.5) + 2*(x > 0.7))
X <- matrix(runif(n * p), ncol = p) 
sim_data <- data.frame(y,x,X) 
dep_var <- "y"
covariates <- c("x", paste0("X", seq(1:10))) # x is informative only, X1, ..., X10 are noise variables

# Split into train and test data
train_sample <- sample(1:n, floor(0.7*n))
train <- sim_data[train_sample, ]
test <- sim_data[-train_sample, ]
```

The dots in red show points that lie outside the 5% and 95% quantiles, which are indicated by the black dashed lines.

![Optional Text](../master/plots/sim_data.png)

Let's fit an **XGBoostLSS** to the data. In general, the syntax is similar to the original XGBoost implementation. However, the user has to make a distributional assumption by specifying a family in the function call. As the data has been generated by a normal distribution, we use the Normal as a function input.

```r
# Data for XGBoostLSS
dtrain <- xgb.DMatrix(data = data.matrix(train[, covariates]),
                      label = train[, dep_var])
dtest <- xgb.DMatrix(data = data.matrix(test[, covariates]))
                     
# Fit model
xgblss_model <- xgblss.train(data = dtrain,
                             family = "NO",
                             n_init_hyper = 50,
                             time_budget = 10)
```

The user has also the option to provide a list of hyperparameters to optimize, as well as its boundary values. In this example however, we use Bayesian Optimization as implemented in the [mlrMBO](https://github.com/mlr-org/mlrMBO) R-package, using a randomly generated set of initial hyperparameters that are used for training the surrogate Kriging model to find an optimized set of parameter. Currently, the default set-up in **XGBoostLSS** optimizes *eta, gamma, max_depth, min_child_weight, subsample* and *colsample_bytree* as hyperparameter. The *time_budget* parameter indicates the running time budget in minutes and is used as a stopping criteria for the Bayesian Optimization.

Once the model is trained, we can predict all parameter of the distribution.            
   
                    
```r
# Predict
xgblss_pred <- predict(xgblss_model,
		       newdata = dtest,
		       parameter = "all")
```

As **XGBoostLSS** allows to model the entire conditional distribution, we can draw random samples from the predicted distribution, which allows us to create prediction intervals and quantiles of interest. The below image shows the predictions of **XGBoostLSS** for the 5% and 95% quantile in blue. 

![Optional Text](../master/plots/xgboostlss_mbo_sim.png)

Comparing the coverage of the intervals with the nominal level of 90% shows that **XGBoostLSS** does not only correctly model the heteroscedasticity in the data, but it also provides an accurate forecast for the 5% and 95% quantiles. The great flexibility of **XGBoostLSS** also comes from its ability to provide attribute importance, as well as partial dependence plots for all of the distributional parameters. In the following we only investigate the effect on the conditional variance. The following plots are generated using wrappers around the [interpretable machine learning (iml)](https://github.com/christophM/iml) R package.

```r
# Shapley value
plot(xgblss_model,
     parameter = "sigma",
     type = "shapley")
```

![Optional Text](../master/plots/xgboostlss_shapley.png)

The plot of the Shapley value shows that **XGBoostLSS** has identified the only informative predictor *x* and does not consider any of the noise variables X1, ..., X10 as an important feature. 


```r
# Partial Dependence Plots
plot(xgblss_model,
     parameter = "sigma",
     type = "pdp")
```
Looking at partial dependence plots of the effect of x on Var(y|x) shows that it also correctly identifies the amount of heteroscedasticity in the data.

![Optional Text](../master/plots/xgboostlss_parteffect.png)

###  Data for the Rent Index 2003 in Munich, Germany

![Optional Text](../master/plots/munich_rent_district.png)

In this example we show the usage of **XGBoostLSS** using a sample of 2,053 apartments from the data collected for the preparation of the Munich rent index 2003.

- *rent*
  - Net rent in EUR (numeric).

- *rentsqm*
  - Net rent per square meter in EUR (numeric).

- *area*
  - Floor area in square meters (numeric).

- *rooms*
  - Number of rooms (numeric).

- *yearc*
  - Year of construction (numeric).

- *bathextra*
  - Factor: High quality equipment in the bathroom?

- *bathtile*
  - Factor: Bathroom tiled?

- *cheating*
  - Factor: Central heating available?

- *district*
  - Urban district where the apartment is located. Factor with 25 levels: "All-Umenz" (Allach - Untermenzing), "Alt-Le" (Altstadt - Lehel), "Au-Haid" (Au - Haidhausen), "Au-Lo-La" (Aubing - Lochhausen - Langwied), "BamLaim" (Berg am Laim), "Bogenh" (Bogenhausen), "Feld-Has" (Feldmoching - Hasenbergl), "Had" (Hadern), "Laim" (Laim), "Lud-Isar"(Ludwigsvorstadt - Isarvorstadt), "Maxvor" (Maxvorstadt), "Mil-AmH" (Milbertshofen - Am Hart), "Moos" (Moosach), "Neuh-Nymp" (Neuhausen - Nymphenburg), "Obgies" (Obergiesing), "Pas-Obmenz" (Pasing - Obermenzing), "Ram-Per" (Ramersdorf - Perlach), "SchwWest" (Schwabing West), "Schwab-Frei" (Schwabing - Freimann), "Schwanth" (Schwanthalerhoehe), "Send" (Sendling), "Send-West" (Sendling - Westpark), "Th-Ob-Fo-Fu-So" (Thalkirchen - Obersendling - Forstenried - Fuerstenried - Solln), "Trud-Riem" (Trudering - Riem) and "Ugies-Har" (Untergiesing - Harlaching).

- *location*
  - Quality of location. Ordered factor with levels "normal", "good" and "top".

- *upkitchen*
  - Factor: Upscale equipment in kitchen?

- *wwater*
  - Factor: Hot water supply available?

```r
# Load data
data("munichrent03", package = "LinRegInteractive")

munichrent03 <- munichrent03 %>% 
  select(-rent) %>% 
  mutate_if(is.integer, as.numeric)
  
# Dummy Coding
munichrent03_dummy <- munichrent03 %>% 
  mlr::createDummyFeatures(target = dep_var)

# Train and Test Data 
set.seed(123)
train_split <- sample(1:nrow(munichrent03), floor(0.7*nrow(munichrent03)))
train <- munichrent03_dummy[train_split,]
test <- munichrent03_dummy[-train_split,]

# Select dependent variable and covariates
covariates <- munichrent03_dummy %>% 
  select(-dep_var) %>% 
  colnames()  
dep_var <- "rentsqm"

# Data for XGBoostLSS
dtrain <- xgb.DMatrix(data = data.matrix(train[, covariates]),
                      label = train[, dep_var])
dtest <- xgb.DMatrix(data = data.matrix(test[, covariates]))
```

The first decision one has to make is about choosing an appropriate distribution for the response. As there are many potential candidates, we use an automated approach based on the generalised Akaike information criterion. Due to its great flexibility, **XGBoostLSS** is built around the R package [gamlss](https://cran.r-project.org/web/packages/gamlss/index.html). 

```r
opt_dist <- fitDist(y = train[, dep_var],
                    type = "realplus",
                    extra = "NO")              
                 
      dist    GAIC
1      GB2 6588.29
2       NO 6601.17
3       GG 6602.02
4     BCCG 6602.26
5      WEI 6602.37
6   exGAUS 6603.17
7      BCT 6603.35
8    BCPEo 6604.26
9       GA 6707.85
10     GIG 6709.85
11   LOGNO 6839.56
12      IG 6871.12
13  IGAMMA 7046.50
14     EXP 9018.04
15 PARETO2 9020.04
16      GP 9020.05

```

Even though the generalized Beta type 2 provides the best approximation to the data, we use the more parsimonious Normal distribution, as it has only two distributional parameter, compared to 4 of the generalized Beta type 2. In general, though, **XGBoostLSS** is flexible to allow the user to choose and fit all distributions available in the [gamlss](https://cran.r-project.org/web/packages/gamlss/index.html) package. The good fit of the Normal distribution is also confirmed by the the density plot, where the train data is presented as a histogram, while the Normal fit is shown in red.

![Optional Text](../master/plots/fitted_dist.png)

Now that we have specified the distribution, let's fit an **XGBoostLSS** to the data. Again, we use Bayesian Optimization for finding an optimal set of hyperparameter, while restricting the overall runtime to 5 minutes.   

```r
# Fit model
xgblss_model <- xgblss.train(data = dtrain,
                             family = "NO",
                             n_init_hyper = 50,
                             time_budget = 5)
```
Looking at the estimated effects indicates that newer flats are on average more expensive, with the variance first decreasing and increasing again for flats built around 1980 and later. Also, as expected, rents per square meter decrease with an increasing size of the appartment.  

```r
# Partial Dependence Plots
plot(xgblss_model,
     parameter = "all",
     type = "pdp")
```

![Optional Text](../master/plots/munich_rent_estimated_effects.png)

The diagnostics for XGBoostLSS are based on the residuals of the fitted model, where we use the  normalised quantile residuals for continuous response variables and randomised normalised quantile residuals for discrete response variable (if the response of the test data is also available, quantile residuals are also available for the test data set). Quantile residuals are based on the idea of inverting the estimated distribution function for each observation to obtain exactly standard normal residuals.

![Optional Text](../master/plots/xgboostlss_quant_res.png)

Despite some slight underfitting in the tails of the distribution, XGBoostLSS provides a well calibrated forecast and the good approximation of our model to the data is confirmed. XGBoostLSS also allows to investigate the estimated effects for all distributional parameter. Looking at the top 10 Shapley values for both the conditional mean and variance indicates that both *yearc* and *area* are considered as being important variables.
             
```r
# Shapley value
plot(xgblss_model,
     newdata = test,
     parameter = "all",
     type = "shapley",
     top_n = 10)
```
![Optional Text](../master/plots/munich_rent_shapley.png)

Besides the global attribute importance, the user might also be interested in local attribute importances for each single prediction individually. This allows to answer questions like '*How did the feature values of a single data point affect its prediction*?' For illustration purposes, we select the first predicted rent of the test data set.

```r
# Local Shapley value
plot(xgblss_model,
     newdata = test,
     parameter = "mu",
     type = "shapley",
     x_interest = 1,
     top_n = 10)
 ```
 
![Optional Text](../master/plots/xgboostlss_shapley_ind.png)
 
We can also measure how strongly features interact with one other. The range of the measure is between 0 (no interaction) and 1 (strong interaction).

```r
# Interaction plots
plot(xgblss_model,
     parameter = "mu",
     type = "interact",
     top_n = 10)
 ```
 
![Optional Text](../master/plots/xgboostlss_inter.png)
  
From all covariates, *yearc* seems to have the strongest interaction. We can also further analyse its effect and specify a feature and measure all its 2-way interactions with all other features.

```r
# 2-Way Interaction plots
plot(xgblss_model,
     parameter = "mu",
     type = "interact",
     feature = "yearc",
     top_n = 10)
 ```
 
![Optional Text](../master/plots/xgboostlss_two_way_inter.png)

As we have modelled all parameter of the Normal distribution, **XGBoostLSS** provides a probabilistic forecast, from which any quantity of interest can be derived. 

```r
# XGBoostLSS predictions
xgblss_pred <- predict(xgblss_model,
		       newdata = dtest,
		       parameter = "all")
```

The following plot shows a subset of 50 predictions only for ease of readability. The red dots show the actual out of sample rents, while the boxplots are the distributional predictions.

![Optional Text](../master/plots/munich_rent_pred_boxplot.png)

Even though the Normal distribution was identified by the AIC as an appropriate distribution, the Whiskers in the above plot show that some of the forecasted rents are smaller than 0. In real life applications, a distribution with positive support might be a more reasonable assumption and the above distributional assumption was made for illustration purposes only. Also, we can plot a subset of the forecasted densities and cumulative distributions.

![Optional Text](../master/plots/munich_rent_density.png)


### Comparison to other approaches
To evaluate the prediction accuracy of **XGBoostLSS**, we compare the forecasts of the Munich rent example to the implementations available in [gamlss](https://cran.r-project.org/web/packages/gamlss/index.html) and in [gamboostLSS](https://cran.r-project.org/package=gamboostLSS). For both implementations, we use factor coding, instead of dummy-coding as for **XGBoostLSS**.


```r
# Data for gamlss models
train_gamlss <- munichrent03[train_split,]
test_gamlss <- munichrent03[-train_split,]
gamlss_form <- as.formula(paste(dep_var, "~ ."))

# gamboostLSS
covariates_gamlss <- munichrent03 %>% 
  dplyr::select(-dep_var) %>% 
  colnames()

gamboostLSS_form <- as.formula(paste0(dep_var, "~",
                                    paste0("bbs(", covariates_gamlss[1:3], ")", collapse = "+"),
                                    "+",
                                    paste0("bols(", covariates_gamlss[4:length(covariates_gamlss)], ")", collapse = "+")))
                                    
# rentsqm ~ bbs(area) + bbs(rooms) + bbs(yearc) + bols(bathextra) + bols(bathtile) + 
# bols(cheating) + bols(district) + bols(location) + bols(upkitchen) + bols(wwater)

gamboostlss_mod <- gamboostLSS(list(mu = gamboostLSS_form,
                                    sigma = gamboostLSS_form),
                                    families = GaussianLSS(),
                                    control = boost_control(mstop = 5000,
                                                               nu = 0.1),
                                    method = "noncyclic",
                                    data = train_gamlss)

cv10f <- cv(model.weights(gamboostlss_mod), type = "kfold")
cv_model <- cvrisk(gamboostlss_mod,
                   folds = cv10f,
                   papply = myApply)
gamboostlss_mod_cv <- gamboostlss_mod[mstop(cv_model)] 

                                           
# GAMLSS
gamlss_form_mu <- as.formula(paste0(dep_var, "~",
                                    paste0("pb(", covariates_gamlss[1:3], ")", collapse = "+"),
                                    "+",
                                    paste0(covariates_gamlss[4:length(covariates_gamlss)], collapse = "+")))
				    
gamlss_form_lss <- as.formula(paste0("~",
                                     paste0("pb(", covariates_gamlss[1:3], ")", collapse = "+"),
                                     "+",
                                     paste0(covariates_gamlss[4:length(covariates_gamlss)], collapse = "+")))
                                    
# rentsqm ~ pb(area) + pb(rooms) + pb(yearc) + bathextra + bathtile + 
#    cheating + district + location + upkitchen + wwater
    
gamlss_model <- gamlss(gamlss_form_mu,
                       sigma.formula = gamlss_form_lss,
                       family = "NO",
                       data = train_gamlss)
```

We evaluate distributional forecasts using the average Continuous Ranked Probability Scoring Rules (CRPS) and the average Logarithmic Score (LOG) implemented in the [scoringRules](https://cran.r-project.org/web/packages/scoringRules/index.html) R-package, where lower scores indicate a better forecast, along with additional error measures evaluating the mean-prediction accuracy of the models.

```r
            CRPS_SCORE LOG_SCORE   MAPE    MSE   RMSE    MAE MEDIAN_AE    RAE  RMSPE  RMSLE   RRSE R_SQUARED
XGBoostLSS      1.1393    2.1570 0.2461 4.1363 2.0338 1.6088    1.3178 0.7807 0.3895 0.2479 0.7826    0.3875
gamboostLSS     1.1541    2.1920 0.2485 4.1596 2.0395 1.6276    1.3636 0.7898 0.3900 0.2492 0.7848    0.3841
GAMLSS          1.1527    2.1848 0.2478 4.1636 2.0405 1.6251    1.3537 0.7886 0.3889 0.2490 0.7852    0.3835
```

All measures, except RMSPE, show that **XGBoostLSS** provides more accurate forecasts than the other two approaches. To investigate the ability of **XGBoostLSS** to provide insights into the estimated effects on all distributional parameter, we compare its estimated effects to those estimated by gamboostLSS.

![Optional Text](../master/plots/munich_rent_gamboostlss.png)

All effects are similar to **XGBoostLSS** and therefore confirm its ability to provide insights into the data generating process.

## Expectile Regression

While GAMLSS retain the assumption of a parametric distribution for the response, it may also be useful to completely drop this assumption and to formulate models that still allow us to describe more than the mean of the response. This may in particular be the case if interest is not on identifying covariate effects on speciﬁc parameters of the response distribution but on the relation of extreme observations in the tails of the distribution on covariates. This is enabled in quantile and expectile regression. As XGBoost requieres both Gradient and Hessian to be non-zero, we illustrate the ability of **XGBoostLSS** to model and provide inference for different parts of the response distribution using expectile regression.

We estimate a model, where we replace the family argument with „Expectile“, where *tau* specifies the expectiles of interest. Note that for *tau=0.5*, a mean regression model is estimated. As in the above examples, we use Bayesian Optimization to find the best hyperparameter.

```r
# Fit model
xgblss_expect <- xgblss.train(data = dtrain,
                              family = "Expectile",
		              tau = c(0.05, 0.5, 0.95),
                              n_init_hyper = 50,
                              time_budget = 10)
``` 


Investigation of the feature importances across different expectiles allows to infere the most important covariates for each point of the response distribution so that, e.g., effects that are more important for more expensive rents can be isolated from those of low rents.

![Optional Text](../master/plots/munich_rent_expectiles_shapley.png)

Furthermore, plotting the effects across different expectile allows to uncover heterogeneity in the data, as the estimated effects, as well as their strengths are allowed to vary across the response distribution. 

![Optional Text](../master/plots/munich_rent_expectiles_effects.png)

## Summary and key features

In summary, **XGBoostLSS** has the following key features: 

- Extends XGBoost to probabilistic forecasting from which prediction intervals and quantiles of interest can be derived.
- Valid uncertainty quantification of forecasts.
- Compatible with all XGBoost implementations, i.e., R, Julia, Python, Java and Scala.
- Parallel model traininig, both CPU and GPU, as well as Spark and Dask.
- Fast Histogram Model Training.
- High prediction accuracy due to Newton boosting. 
- High interpretability of results.
- Efficient representation of sparse matrices.
- Compatibility for training large models using large datasets (> 1 Mio rows).
- Low memory usage.
- Missing value imputation.

While statistical boosting algorithms, such as gamboostLSS, are supposed to perfrom well in terms of speed for medium to moderately big sized data sets, where the focus is on inference rather than prediction accuracy, **XGBoostLSS** plays off its strengths in situations where the user faces data sets that deserve the term big data. The above mentioned features set **XGBoostLSS** apart from existing distributional modeling approaches and highlight the benefits of our approach that combines the interpretability and flexibility of GAMLSS with the speed and accuracy of XGBoost. 

## Software Implementations
In its current implementation, **XGBoostLSS** is available in *R* only, but extensions to *Julia* and *Python* are in progress.

## References
März, Alexander (2019). *"XGBoostLSS - An extension of XGBoost to probabilistic forecasting"*. pp. 1–26. *Under Review*.

You may find a summary of the paper [here](). 
