# nostradamus
Generalized time series model for multivariate and multiple regression of sparsely sampled time series

## Time series modelling
Modeling time series data involves analyzing data points collected or recorded sequentially over time to uncover patterns, trends, and seasonal behaviors. The goal is often to forecast future values based on historical data, identify anomalies, or understand the underlying dynamics of the system. Techniques range from statistical methods like ARIMA and exponential smoothing to machine learning approaches like recurrent neural networks (RNNs) and transformers. Effective time series modeling accounts for unique characteristics such as autocorrelation, seasonality, and irregular sampling intervals.

This package is aimed at developing effective time series models via regression techniques. We are inspired by the Facebook PROPHET model (Taylor, S. J., & Letham, B. (2018). Forecasting at Scale. The American Statistician, 72(1), 37–45. https://doi.org/10.1080/00031305.2017.1380080) and "Fitting Very Flexible Models: Linear Regression with Large Numbers of Parameters" (https://ui.adsabs.harvard.edu/abs/2021PASP..133i3001H/abstract).

## The model
We developed this model and framework to be used primarily for multi-colour photometric time series data. The Facebook PROPHET model did an excellent job at introducing scalable time series models with multiple components to handle periodic (seasonal) and various linear (drift) components, plus outlier (holiday) effects. This model aims to allow for the flexible modelling of multiple time series that are sampled asynchronously and that have an arbitrary amount of linear and periodic terms. 

## The algorithm

We assume that the periodic (seasonal) trend is the dominant source of variability in these cases. As such, we are primarily interested in modelling this feature of the data. The first step that we take is to identify the period / frequency of the dominant signal - while there are several methods that may be applied to the (e.g. Non-parametric kernel regression, conditional entropy, analysis of variance), we stick to the Lomb-Scargle periodogram. In effect, the algorithm is as follows:

    - Identify frequency / period of highest amplitude (& check subharmonics).
    - Define a high resolution grid in frequency within +/- 1/T, where T is the time-base of the data set.
    - Fit a regression model (with a linearized Fourier basis) with K harmonic terms to each point in the frequency grid, and select the best model.
    - Use residuals to determine points in time (i.e. break-points) where the mean varies by more than 30% of the standard deviation.
    - Do hyper-parameter optimization by shifting break-points and "width" parameter for feature weighting.

To avoid a non-linear optimization problem, we opt to instead 


The linear terms are initialized with "break-points" that are determined by