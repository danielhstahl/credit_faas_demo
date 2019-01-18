| [Linux][lin-link] | [Codecov][cov-link] |
| :---------------: | :-----------------: |
| ![lin-badge]      | ![cov-badge]        |

[lin-badge]: https://travis-ci.com/phillyfan1138/credit_faas_demo.svg "Travis build status"
[lin-link]:  https://travis-ci.com/phillyfan1138/credit_faas_demo "Travis build status"
[cov-badge]: https://codecov.io/gh/phillyfan1138/credit_faas_demo/branch/master/graph/badge.svg
[cov-link]:  https://codecov.io/gh/phillyfan1138/credit_faas_demo

## Demo for Credit FaaS

This project showcases how to use Fang Oosterlee's algorithm for efficiently computing portfolio statistics at a granular (loan) level.  

## Documentation

Model documentation and theory is available in the [Credit Risk Extensions](https://github.com/phillyfan1138/CreditRiskExtensions/blob/master/StahlMultiVariatePaper.pdf) repository.

## Features

The model allows for:
* Correlated PD
* Stochastic LGD (but uncorrelated with anything else)
* Granularity: no need for the assumption of "sufficiently diversified" 
* Efficiency: near real-time computation

## How to build
First, download this repo:
`git clone https://github.com/phillyfan1138/credit_faas_demo`

Change directory into the folder:
`cd credit_faas_demo`

This repo contains test files that are quite large.  I use [git-lfs](https://git-lfs.github.com/) to handle these files.  If you already have git-lfs installed, the files should be downloaded automatically.  If you cloned the repo without git-lfs installed, after installation run:
`git-lfs fetch`

Regardless of whether you download the test files, you can build the binaries with:
`cargo build --release`

## How to run
To get the expected shortfall and value at risk for granular (1 million) loans, run the following:

`./target/release/main $(cat ./data/parameters.json)  ./data/loans.json`

With optional density export:

`./target/release/main $(cat ./data/parameters.json)  ./data/loans.json ./docs/loan_density_full.json`

## Recommended implementation
In a real production setting, there will typically be a finite set of segments that describe loans.  For example, a loan may have one of 10 risk ratings and one of 10 facility grades.  Loans may also be grouped by rough exposure amount (eg, roughly 10 separate exposures).  This leads to 1000 different combinations.  Instead of simulating over every single loan, the model could simulate over each group, with each group multiplied by the number of loans in each group.  If there are 30 loans with risk rating 4, facility grade 6, and in exposure segment 5, then the exponent of the characteristic function would be 30*p(e^{uil}-1) where p is the probability of default associated with risk rating 4 and l is the combined dollar loss for a loan in segment 5 and facility grade 6.  

This will dramatically decrease the computation time.

To run the demo for this recommended implementation, 

`./target/release/main $(cat ./data/parameters.json)  ./data/loans_grouped.json`

With optional density export:

`./target/release/main $(cat ./data/parameters.json)  ./data/loans_grouped.json ./docs/loan_density_aggr.json`


## Comparison of granular and recommended implementations

Note that these plots were generated using two different simulations and are not intended to represent the same loan portfolio.  The differences when applied to a real loan portfolio should be minimal.

![](docs/density_compare.jpg?raw=true)

## Comparison with Risk Metrics

[Risk Metrics](https://www.msci.com/documents/10199/93396227-d449-4229-9143-24a94dab122f) is a competing method of estimating the loss distribution of a portfolio of loans.  For a simple single factor model, the following plot shows the mapping between the systemic correlation parameter (rho) for Risk Metrics and the standard deviation of the systemic variable for Credit Risk Plus (standard_deviation):

![](docs/vol_corr_compare.jpg?raw=true)

Note that the volatility of the underlying systemic variable for Credit Risk Plus has to be relatively large to map to the same correlation for Risk Metrics.  Comparisons between the two methodologies have been studied extensively in the literature (eg, [Gordy (1998)](https://www.federalreserve.gov/pubs/feds/1998/199847/199847pap.pdf)).  The essential differences include:

* Correlation between defaults are more dependent on the probability of default in the Risk Metrics model.  This implies that there is no method to consistently convert Risk Metrics correlations to Credit Risk Plus correlations for a heterogenous portfolio.  
* Risk Metrics allows for granular correlations of assets with the industry.  In Credit Risk Plus, these correlations are controlled by the "weights" vector but would not directly match the correlations of the asset with the industry.
* Risk Metrics does not allow for direct correlation of an asset with other assets in other industries.  The correlation is only indirectly correlated via the industry correlation.  In Credit Risk Plus, these correlations can be directly implemented via the "weights" vector.  

## Converting Risk Metrics correlation and R squared matrix to Credit Risk Plus

### Convert industry correlation
Risk Metrics requires a correlation matrix for different industries.  Within each industry, an "R-squared" measures the amount of correlation between (fully diversified) obligors in the industry.  One way to convert the industry correlation matrix is to take the entries of the correlation matrix and convert these to weights for the other industry random variables for a given obligor.

As an example, consider the correlation from the Risk Metrics technical document:

[1.0, .16, .08; .16, 1.0, .34; .08, .34, 1.0]

The weight vector for an obligor in  industry 1 would be [.806, .129, .065].  The weight vector for an obligor in  industry 2 would be [.107, .666, .227].  The weight vector for an obligor in  industry 3 would be [.056, .240, .704].  

### Convert R-squares
One way to map the R-squared vector is to convert the R-squared to a variance for a gamma random variable for each industry.  This can be done by mapping the default correlation implied by Risk Metrics to the default correlation implied by Credit Risk Plus.  As an example, see 
[this line in the model code](https://github.com/phillyfan1138/credit_faas_demo/blob/master/src/bin/main.rs#L68).  Note that this mapping requires a probability of default as an input.  For an entire industry, it makes sense to use the average probability of default for the obligors in the industry to accomplish this conversion.  

## Completed

* Parsimonious integration with correlations from Risk Metrics
* Include functions for computing risk contributions 
    * For an entire portfolio 
    * For an incremental loan.

Goals (decreasing order of importance):

* Ingest loan level parameters (balance, pd, lgd, ...etc) to create a loss characteristic function 
* High memory efficiency (don't store much data in ram)

Success Criteria

* 1,000,000 loans in under 5 seconds on  i5-5250U CPU @ 1.60GHz × 4 (done after aggregations)
* Reproducible (same output given same input) (done)
* Transparent (easy to replicate) (done)
