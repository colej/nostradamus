# nostradamus
Generalized time series model for multivariate and multiple regression of sparsely sampled time series

## Time series modelling
Modeling time series data involves analyzing data points collected or recorded sequentially over time to uncover patterns, trends, and seasonal behaviors. The goal is often to forecast future values based on historical data, identify anomalies, or understand the underlying dynamics of the system. Techniques range from statistical methods like ARIMA and exponential smoothing to machine learning approaches like recurrent neural networks (RNNs) and transformers. Effective time series modeling accounts for unique characteristics such as autocorrelation, seasonality, and irregular sampling intervals.

This package is aimed at developing effective time series models via regression techniques. We are inspired by the Facebook PROPHET model (Taylor, S. J., & Letham, B. (2018). Forecasting at Scale. The American Statistician, 72(1), 37–45. https://doi.org/10.1080/00031305.2017.1380080) and "Fitting Very Flexible Models: Linear Regression with Large Numbers of Parameters" (https://ui.adsabs.harvard.edu/abs/2021PASP..133i3001H/abstract).

## The model
We developed this model and framework to be used primarily for multi-colour photometric time series data. The Facebook PROPHET model did an excellent job at introducing scalable time series models with multiple components to handle periodic (seasonal) and various linear (drift) components, plus outlier (holiday) effects. This model aims to allow for the flexible modelling of multiple time series that are sampled asynchronously and that have an arbitrary amount of linear and periodic terms. 

## Feature embedding 
To model the periodic (seasonal trends), we perform a feature embedding and express our model in Fourier terms. This allows us to linearize the model and use an feature-weighted least squares regression to determine the coefficients of our model. This assumes that the period/frequency is known and is used as an input feature to the design matrix, e.g. the data is transformed into a feature embedding such that time t_i becomes cos(2\pi f t_i) sin(2\pi f t_i), and the amplitude and phase are calculated from the coefficients. This feature embedding effectively increases the dimensionality of our problem by 2K, where K is the number of harmonics considered in our model. 

In order to be both general and data driven in the number of harmonics considered for a given model, we allow for the number of harmonics K to be arbitrarily large (default K=150) and we add a feature weighting following Hogg \& Villar, (2021), with a single tunable parameter S, which controls the "width" of the harmonic (@Hogg, please correct me here). We select a weight that preferentially penalizes higher frequency terms and effectively limits the number of harmonics that significantly contribute to the model. In this way, we let the data select the number of harmonic terms that are required to model the periodic trends.  


## The algorithm

We assume that the periodic (seasonal) trend is the dominant source of variability in these cases. As such, we are primarily interested in modelling this feature of the data. The first step that we take is to identify the period / frequency of the dominant signal - while there are several methods that may be applied to the (e.g. Non-parametric kernel regression, conditional entropy, analysis of variance), we stick to the Lomb-Scargle periodogram. In effect, the algorithm is as follows:

    - Identify frequency / period of highest amplitude (& check subharmonics).
    - Define a high resolution grid in frequency within +/- 1/T, where T is the time-base of the data set.
    - Fit a regression model (with a linearized Fourier basis) with K harmonic terms to each point in the frequency grid, and select the best model.
    - Use residuals to determine points in time (i.e. break-points) where the mean varies by more than 30% of the standard deviation.
    - Do hyper-parameter optimization by shifting break-points and "width" parameter for feature weighting.

To avoid a non-linear optimization problem, we opt to instead define a small, finely sampled grid around the dominant frequency that we extract via the Lomb-Scargle periodogram. We sample 50 points between f_peak +/- f_resolution, where f_fresolution = 1/time_base is set by the time-base of the time series. We select the frequency that mimizes the the MSE of the residuals (plus the regularization) as our optimal frequency, and move to the next stage. 

## Linear trends and break-points

The linear terms are initialized with "break-points" that are determined by locations where the mean of the residuals (after subtraction of the seasonal component) change by more than 30% of the standard deviation of the residuals. To minimize the number of break-points, we smooth the residuals in time-space with a moving window that is 20% of the time-base wide. This allows us to reduce the number of jumps in the data significantly before assessing locations for breakpoints. 

## Univariate vs. Multivariate time series

This modelling procedure can be used for either a univariate time series, or for univariate time series. In the astrophysics example that I developed this for, this means that we can use this procedure to model either single filter photometric time series, or multi-filter photometric time series. In the multi-variate case, we enforce that the periodic (seasonal) trend is the same for all data streams, but allow for each data stream to different linear trends. While we could model all of the data streams in a single matrix, where each data stream would essentially constitute a block matrix along the diagonal, the matrix calculation scales as N^3. Instead, we can just treat each data stream independently because 3*N_litte^3 is much faster than N_big^3. Since the regression will always minimize the residuals (and since we assume that each data steam is independent - apart from the underlying periodicity), this is effectively the same as a global minimization including all data streams. 

## Error estimation
Unlike the case of diversification in portfolio management, there is no free lunch in linear models - meaning, since we're adding a feature embedding and weighting, we can't use simple standard errors to obtain confidence intervals for our regression coefficients. Instead, we have to use bootstrapping and/or jack-knifing. We provide routines for both, so see which is better for your particular usecase. 

I /really really really/ __REALLY__ did not want to use MCMC to estimate the regression coefficients, but in a future update, I might cave and add it anyways.