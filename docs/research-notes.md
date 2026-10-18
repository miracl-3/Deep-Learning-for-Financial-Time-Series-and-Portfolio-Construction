# Research and Evaluation Notes

These notes accompany the [project overview](../README.md) and the [original notebook](../230070_project_notebook.ipynb). They describe the coursework study completed in May 2026. Values were transcribed from stored notebook outputs; experiments were not rerun for the documentation revision of 18 October 2026.

## Evidence and scope

The notebook is the primary source for the methods and numerical results. The [original PDF report](../230070-project-report.pdf) preserves the course submission, while the [report draft](../deployment/docs/final_report_draft.md) provides an earlier narrative. The qualifications below govern interpretation of the saved experiments, including where the original narrative uses stronger language.

| Notebook section | Subject | Relevant evidence |
| --- | --- | --- |
| Reusable Functionality | Preprocessing, chronological splits, scaling, recurrent models, cross-validation, and baselines | Implemented utilities and metric definitions |
| Task 1 | Nasdaq price forecasts | Selected `GT` example; three forecasting horizons; persistence comparison |
| Task 2 | Vietnamese price forecasts | Selected `VCB` example; three forecasting horizons; persistence comparison |
| Task 3 | Buy/sell classification | Validation-selected thresholds and saved test classifications |
| Task 4 | Portfolio allocation | Forty-stock return model, scoring, two allocations, a mean-variance example, and a historical illustration |
| Task 5 | Local serving | Artifact export, FastAPI, Streamlit, and a proposed orchestration design |

## Data and experimental design

The repository contains historical CSVs for Nasdaq equities and Vietnamese stocks. Vietnamese supplemental files include dividend history, financial ratios, and industry analysis; their presence does not imply that every experiment uses them. The price models use historical price/volume data and derived technical features.

The existing documentation does not establish upstream providers, download dates, licenses, or a complete corporate-action policy. Those details should be verified before attributing a provider or redistributing the datasets elsewhere.

For the saved price-forecast experiments:

- The input window contains 30 observations. Nasdaq inputs contain 10 features and target adjusted close; Vietnamese inputs contain 9 features and target close.
- The chronological split uses 70% training, 15% validation, and the remaining observations for testing. Stored prepared-row counts are 3,676 / 788 / 788 for GT and 2,375 / 509 / 510 for VCB.
- Feature and target MinMax scalers are fitted on the training block. Prior observations provide input context at validation and test boundaries.
- Three-fold time-series cross-validation operates on the initial 85% block. The final GRU experiments use validation-loss early stopping and learning-rate reduction, with training shuffling disabled.
- The forecasting horizons are one subsequent observation, the third subsequent observation, and a vector of the next three observations. These are row-based horizons; they require a verified trading calendar before being treated as precise trading-day horizons.

Ticker eligibility and ranking are calculated using full available histories. That selection process is not point-in-time. A chronological train/test partition therefore does not remove every source of selection bias.

The saved GT preview also contains dates falling on weekends. This warrants auditing timestamp parsing and row ordering against the original source before interpreting the sequence as a clean exchange-session history. The documentation preserves the reported values and does not certify the underlying market calendar.

## Forecast evaluation

### Saved GRU summaries

| Market / ticker | Forecast | MAE | RMSE | sMAPE (%) | R² | Directional accuracy (%) |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| Nasdaq / GT | Next observation | 2.3361 | 3.0346 | 13.1027 | 0.8358 | 42.31 |
| Nasdaq / GT | Third observation | 2.4790 | 3.2099 | 13.8992 | 0.8166 | 50.19 |
| Nasdaq / GT | Next three observations | 2.4779 | 3.1904 | 13.8397 | 0.8185 | 55.11 |
| Vietnam / VCB | Next observation | 4,328.9255 | 4,979.1999 | 5.5286 | 0.2291 | 44.20 |
| Vietnam / VCB | Third observation | 4,688.4456 | 5,474.1722 | 6.0109 | 0.0690 | 45.36 |
| Vietnam / VCB | Next three observations | 7,199.6722 | 7,960.2206 | 9.4489 | -0.9911 | 45.31 |

The next-observation persistence comparisons are shown in the [README](../README.md#next-observation-price-forecasts). They indicate a price-error improvement for GT and a substantial deterioration for VCB. Results from two selected tickers do not establish performance across their respective markets.

### Metric definitions and cautions

- **MAE / RMSE:** errors in the native target-price units, after inverse scaling. Cross-market magnitudes are not directly comparable.
- **sMAPE:** the mean absolute error divided by the average absolute actual/predicted magnitude, multiplied by 100.
- **R²:** a measure of price-level fit. Persistence can achieve high values in a persistent price series; a high value does not demonstrate return predictability.
- **Directional accuracy:** the notebook compares signs of successive differences in the predicted-price series with signs of successive differences in the actual-price series. This differs from comparing a forecast with the price observed at the moment of the forecast.
- **Multi-output metrics:** three-observation outputs are flattened, including overlapping forecast windows. Their directional measure mixes within-window differences and boundaries between forecast origins. Treat it as an implementation-specific summary, not a trading accuracy measure.

The persistence baseline predicts the previous actual value, with the first prediction initialized from the last validation price. The additional "Mean baseline" uses the **test-set mean**. It is an ex post diagnostic and is not an available real-time forecasting benchmark. No baseline advantage or small directional difference establishes statistical significance without an appropriate uncertainty analysis.

## Signal-classification evaluation

| Signal | Chosen threshold | Accuracy | Precision | Recall | F1 | ROC AUC | Reported PR metric | Positive-class fraction |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Buy | 0.39 | 0.4531 | 0.3099 | 0.5301 | 0.3911 | 0.4642 | 0.2995 | 0.3313 |
| Sell | 0.47 | 0.5170 | 0.3582 | 0.6235 | 0.4550 | 0.5511 | 0.3410 | 0.3234 |

The notebook labels the precision-recall summary "PR AUC"; its plotting utility uses average precision. Positive-class prevalence is included to put these scores in context. At approximately one-third positives, accuracy alone is not sufficient; both reported accuracies are below the corresponding always-negative classification accuracy. The sell ROC AUC is only modestly above 0.5, and the buy ROC AUC is below it. Neither result establishes a deployable trading signal.

## Portfolio evaluation

### Allocation mechanics

The portfolio experiment uses 40 stocks, 60-observation feature windows, and a 20-observation future-return target. It combines return forecasts with trailing momentum, volatility, downside risk, drawdown, and liquidity measures. The final two allocations each hold eight selected stocks:

- **Risk-Taking:** a score with a lighter risk penalty, followed by positive predicted-return weighting.
- **Prudent:** a score with a stronger risk penalty, followed by inverse-volatility weighting.

The separate mean-variance example maximizes predicted return minus a covariance-based risk penalty under long-only, fully invested constraints. Its saved solution concentrates essentially all weight in CLC. It is not one of the three portfolios in the historical performance table. Its predicted-return horizon and daily covariance units also require alignment before interpreting the optimization economically.

### Why the historical comparison is retrospective

The notebook predicts returns from the last available window for each ticker, with most stored prediction dates on **28 February 2023**. It calculates risk scores from the last available trailing windows and builds portfolios from those scores. It subsequently applies the resulting fixed weights to returns beginning **16 December 2019**.

Consequently, the allocation uses information that was unavailable at the beginning of the illustrated period. The chart is affected by look-ahead bias and does not measure a strategy that could have been followed using information available on each date. This qualification also applies to the comparator, which is selected from the same end-of-sample score table.

| Allocation | Total return (%) | Annualized return (%) | Sharpe | Maximum drawdown (%) | Observations |
| --- | ---: | ---: | ---: | ---: | ---: |
| Risk-Taking | 64.04 | 16.92 | 0.930 | -36.45 | 798 |
| Prudent / inverse volatility | 56.77 | 15.26 | 0.936 | -32.29 | 798 |
| Equal-weight selected-stock comparator | 60.14 | 16.03 | 0.964 | -33.94 | 798 |

These are the original reported outputs, rather than corrected estimates. The calculation:

- applies the same weights to each day's returns, implying constant-weight exposure rather than weights drifting as in an unrebalanced buy-and-hold portfolio;
- annualizes using 252 observations and assumes zero risk-free return in the Sharpe ratio;
- excludes transaction costs, slippage, and rebalancing expenses;
- fills unavailable aligned ticker returns with zero;
- selects the first eight eligible entries from the sorted score table for its equal-weight comparator, rather than using a broad market index or all 40 stocks.

The lower reported drawdown of the prudent allocation is descriptive within this illustration. It does not establish a future risk reduction or investment advantage.

### What a future validation would require

A future study would need to form every universe, feature set, forecast, risk estimate, and allocation from information available at each historical formation date. Overlapping 20-observation labels would require appropriate boundary treatment, and cross-ticker splits would need common calendar cutoffs. A specified rebalancing rule, aligned return/covariance horizons, transaction costs, missing-price treatment, and independent benchmarks would make the evaluation economically interpretable.

Those are proposed research improvements. They have not been implemented as part of this documentation update.

## Reproduction notes

1. Use the environment and installation commands in the [README](../README.md#getting-started). Dependencies are unpinned, and the notebook's stored kernel metadata does not reconstruct the complete original training environment.
2. Configure the existing notebook paths before execution. `PROJECT_DIR` should identify the checkout; `RAW_DATA_DIR` should identify its `data/` directory. The notebook's saved Google Drive / `data/raw/` assumptions differ from the committed layout.
3. Check resolved dataset locations, date parsing, and chosen tickers before starting a full run. The default ticker selectors can change their choices if the input universe changes.
4. Distinguish reviewing saved outputs from reproducing them. A fresh run can produce different results because of dependency versions, device behavior, and stochastic training, even with the configured seed of 42.
5. Running the local API/dashboard uses supplied model and scaler files. It does not validate their forecast quality or reproduce training. The main artifact directory is `deployment/artifacts/`; the API also contains a fallback copy under `deployment/api/artifacts/`.

The API artifacts describe a 30-by-10 GT input and a 30-by-9 VCB input. Their feature order is recorded in each artifact's `metadata.json`. Treat these as ticker-specific examples.

## Figure provenance

The two research figures linked from the README were extracted directly from PNG outputs already embedded in the notebook. No model was run and no numerical series was regenerated:

| Figure | Saved output source |
| --- | --- |
| `figures/nasdaq-next-day-forecast.png` | Cell beginning `# Next day forecast sample`, actual-versus-predicted output |
| `figures/portfolio-retrospective-comparison.png` | Cell beginning `# Call the backtesting to see the performance and its cumulative returns`, comparison plot |

Original plot titles are preserved, including the portfolio plot's "Test Period" wording. The README caption supplies the timing qualification needed to interpret that figure.
