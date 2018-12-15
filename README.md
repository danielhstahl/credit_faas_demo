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

`cargo build --release`

## How to run

`./target/release/loan_cf $(cat ./data/parameters.json)  ./data/loans.json`

With option density export:

`./target/release/loan_cf $(cat ./data/parameters.json)  ./data/loans.json ./docs/loan_density_full.json`

## Recommended implementation
In a real production setting, there will typically be a finite set of segments that describe loans.  For example, a loan may have one of 10 risk ratings and one of 10 facility grades.  Loans may also be grouped by rough exposure amount (eg, roughly 10 seperate exposures).  This leads to 1000 different combinations.  Instead of simulating over every single loan, the model could simulate over each group, with each group multiplied by the number of loans in each group.  If there are 30 loans with risk rating 4, facility grade 6, and in exposure segment 5, then the exponent of the characteristic function would be 30*p(e^{uil}-1) where p is the probability of default associated with risk rating 4 and l is the combined dollar loss for a loan in segment 5 and facility grade 6.  

This will dramatically decrease the computation time.

To run the demo for this recommended implementation, 

`./target/release/loan_cf $(cat ./data/parameters.json)  ./data/loans_grouped.json`

With option density export:

`./target/release/loan_cf $(cat ./data/parameters.json)  ./data/loans_grouped.json ./docs/loan_density_aggr.json`


## Comparison of granular and recommended implementations

Note that these plots were generated using two different simulations and are not intended to represent the same loan portfolio.  The differences when applied to a real loan portfolio should be minimal.

![](docs/density_compare.jpg?raw=true)

## Roadmap

Goals (decreasing order of importance):

* Ingest loan level parameters (balance, pd, lgd, ...etc) to create a loss characteristic function (done)
* High memory efficiency (don't store much data in ram) (done)

Success Criteria

* 1,000,000 loans in under 5 seconds on  i5-5250U CPU @ 1.60GHz × 4 (done after aggregations)
* Reproducible (same output given same input) (done)
* Transparent (easy to replicate) (done)