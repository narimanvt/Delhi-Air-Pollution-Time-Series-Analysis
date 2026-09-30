# Delhi Air Pollution – Time Series Analysis

Time series modelling of hourly **PM2.5** concentrations in Delhi, using stationarity analysis and AR / ARMA models.

## Purpose

Delhi regularly records some of the highest particulate pollution levels in the world. The goal of this project is to **model the hourly PM2.5 series** with classical time series methods and to find out which model structure describes its dynamics best.

Along the way, the project covers the standard Box–Jenkins workflow: cleaning an irregular real-world series, removing trend, checking stationarity, identifying the model order from ACF/PACF, estimating parameters, and testing the fit with residual diagnostics.

## Data

| Item | Detail |
|---|---|
| Source file | `DL040.csv` (hourly air-quality monitoring data for Delhi) |
| Records | 20,842 hourly rows |
| Full period | November 2020 – February 2025 |
| Variables | 22 columns: pollutants (PM2.5, PM10, NO, NO2, NOx, NH3, SO2, CO, …), meteorology (AT, RH, WS, BP, …) and organic compounds (Toluene, Xylene, Eth-Benzene) |
| Target variable | PM2.5 (µg/m³) |

Only PM2.5 was modelled. About 30% of the hourly values are missing (14,695 non-null out of 20,842), and the record has large gaps after mid-2022. The analysis therefore uses the continuous window **13 Nov 2020 – 31 Jul 2022**.

## Tools

- **Python** (Google Colab notebook)
- **pandas, NumPy** – data handling, resampling, interpolation
- **statsmodels** – ADF test, ACF/PACF, `AutoReg`, `ARIMA`, Ljung–Box test
- **SciPy** – numerical optimisation (`minimize`)
- **Plotly, Matplotlib** – interactive and static plots

## Methods and workflow

1. **Load and clean** – rename columns, parse timestamps, keep only `datetime` and `pm25`.
2. **Resample to a regular hourly grid** – this exposes 6,147 missing hours.
3. **Visual inspection** – the interactive plot of the full series shows discontinuities in the data, so the analysis is restricted to data up to 31 Jul 2022.
4. **Trend detection** – manual sample autocorrelation at lags 1–5 (0.85, 0.80, 0.74, 0.69, 0.65). It decays slowly, which indicates a trend / non-stationarity.
5. **Handle gaps** – linear interpolation for gaps of up to 24 hours, forward-fill for longer ones.
6. **Transform and difference** – `log(1 + PM2.5)` to stabilise variance, then first-order differencing.
7. **Stationarity check** – Augmented Dickey–Fuller test on the differenced series: statistic ≈ −26.66, p ≈ 0.0, so the series is stationary.
8. **ACF / PACF** – bar charts with 95% confidence bounds. Lag-1 autocorrelation is about −0.23, with a slowly decaying pattern.
9. **AR(2) model** – fitted with `AutoReg` (conditional MLE). AIC ≈ 6914.9.
10. **Residual diagnostics** – the Ljung–Box test rejects white-noise residuals from lag 3 onward (p ≈ 1e-8 and smaller), so AR(2) is not adequate.
11. **ARMA order comparison** – candidate orders compared by AIC:

    | Order (p,q) | AIC |
    |---|---|
    | (2,1) | 6304.88 |
    | (2,2) | 6286.91 |
    | (3,2) | 6166.42 |
    | (2,3) | 6249.83 |
    | (3,3) | 6065.57 |

12. **ARMA(2,2) fit** (MLE), chosen as a compromise between fit and parsimony.
13. **Alternative estimation methods** – parameters re-estimated by least-squares (MSE) minimisation and by method of moments, then compared with MLE.

## Results

**AR(2)** (on the differenced log series):

```
X_t = -0.2684 X_{t-1} - 0.1681 X_{t-2} + ε_t
```

**ARMA(2,2)** by MLE (AIC 6286.9, σ² ≈ 0.0897):

```
X_t = 0.8851 X_{t-1} - 0.0642 X_{t-2} - 1.2139 ε_{t-1} + 0.2512 ε_{t-2} + ε_t
```

All four coefficients are statistically significant (p < 0.05).

**Method of moments** gave a noticeably different, simpler approximation:

```
X_t = 0.2684 X_{t-1} - 0.1681 X_{t-2} + ε_t - 0.1838 ε_{t-1} + 0.0532 ε_{t-2}
```

## Conclusion

- The raw hourly PM2.5 series is strongly persistent and non-stationary. After a log transform and first differencing it becomes stationary (ADF p ≈ 0).
- A pure **AR(2)** model is not sufficient: its residuals remain autocorrelated (Ljung–Box rejects at lag ≥ 3).
- Adding moving-average terms helps substantially. **ARMA(2,2)** lowers the AIC from ≈ 6915 to ≈ 6287, and all its coefficients are significant.
- Larger models such as ARMA(3,3) reach a lower AIC (≈ 6066), so ARMA(2,2) was chosen as a balance between accuracy and simplicity rather than as the AIC minimum.
- Estimates from different methods do not all agree: the MSE optimisation returned the MLE starting values with an objective value of 1e10, meaning the routine hit its failure fallback and did not really optimise. The method-of-moments estimates are only an approximation and differ from MLE. MLE is the estimate to rely on.

### Limitations and possible next steps

- Only data up to July 2022 is used because of large gaps afterwards.
- Interpolation and forward-filling of long gaps can smooth the series artificially.
- The analysis does not model daily/seasonal cycles. A SARIMA model or seasonal differencing (24 h, weekly, yearly) is a natural extension.
- The other pollutants and meteorological variables (wind speed, temperature, humidity) could be added as exogenous regressors (ARIMAX).
- Forecast accuracy on a held-out test set was not evaluated.

## Repository contents

- `Dehli_Airpollution_Time_series_analysis.ipynb` – the full analysis notebook
- `README.md` – this file

## How to run

1. Open the notebook in Google Colab or Jupyter.
2. Provide the `DL040.csv` file and update `file_path` in the notebook (the original used a Google Drive path; remove the `drive.mount` cell if running locally).
3. Install dependencies: `pip install pandas numpy statsmodels scipy plotly matplotlib`
4. Run all cells in order.
