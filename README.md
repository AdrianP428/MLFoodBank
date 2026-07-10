**Food Bank Demand Prediction Model**

**Western Massachusetts | Machine Learning Project**

A machine learning model that predicts monthly food assistance demand across six regions in Western Massachusetts. The goal is to help The Food Bank of Western Massachusetts plan food inventory and volunteer staffing ahead of time instead of reacting after the fact.


**Overview**

The Food Bank of Western Massachusetts serves four counties (Berkshire, Franklin, Hampden, and Hampshire) and tracks demand separately for Springfield and Holyoke. Monthly demand shifts based on weather, economic conditions, and seasonal patterns, but figuring out how much it will shift is hard to do reliably from intuition alone.

This project builds a Random Forest regression model that predicts the number of people assisted per month per region and outputs a High / Normal / Low demand signal. That signal is integrated directly into the food bank's existing web bulletin board so staff can see it without any extra tools.


**Results**

MetricThis ModelNaive BaselineMean Absolute Error~1,669 people/month~6,037 people/monthR2 Score0.89-0.31

The model explains 89% of month-to-month variation in demand and is 3.6x more accurate than just predicting the historical average every month. The baseline R2 of -0.31 means the naive guess was actually worse than useless on the test set, which confirms the model is picking up real patterns.

Key finding: Previous month's demand is by far the strongest predictor of current demand. This suggests food bank need is momentum-driven rather than highly reactive to short-term weather or economic changes. In practice, the model works best as a trend-tracker and for flagging months where conditions look unusual.


**Data Sources**

SourceWhat it providesHow it's collectedOpen-Meteo Historical APIDaily temperature and precipitation per regionAutomated API callFRED - Federal Reserve Economic DataMonthly county unemployment ratesAutomated API callBLS Local Area Unemployment StatisticsMonthly city-level unemployment for Springfield and HolyokeAutomated API callFood Bank of Western MassachusettsMonthly people assisted per regionManual collection from public dashboard

Weather was pulled for one representative city per county: Pittsfield (Berkshire), Greenfield (Franklin), Springfield (Hampden, Springfield, and Holyoke), and Northampton (Hampshire). Springfield and Holyoke share the same weather source because they are about 8 miles apart in the same Connecticut River Valley climate zone, so their conditions are effectively identical.


**Methodology**

Data collection and merging

Six separate data streams (one per region) were pulled from multiple public sources, converted to monthly resolution, tagged by region, and merged on shared region/year/month keys into one unified training table.

Key decisions and why

One combined model instead of six separate ones. With only about 4 years of monthly data per region, splitting into six models would leave each one with too few training examples to learn reliably. Region is included as a feature instead, letting one model learn regional differences while training on the full pooled dataset.

Time-based train/test split instead of random. The model trains on 2021 to 2023 and tests on 2024 onwards. A random split would let the model accidentally see future data during training, which inflates apparent accuracy without reflecting how it would actually perform on new months.

Lag feature (previous month's demand). Adding last month's actual demand as a feature improved R2 from 0.38 to 0.89. Food insecurity is a persistent condition driven by structural factors that do not shift dramatically from one month to the next, so recent history turns out to be the strongest available signal.

Hampden County adjusted for overlap. The food bank's dashboard tracks Hampden County total, Springfield, and Holyoke separately. Since Springfield and Holyoke are geographically inside Hampden County, the county total includes them. Adjusted Hampden figures were calculated as Hampden total minus Springfield minus Holyoke to keep all regions non-overlapping.

October 2025 excluded. County-level unemployment data for this month is permanently missing nationwide because a federal government shutdown prevented BLS from collecting the household survey. This month was removed from training rather than estimated around a genuine data collection failure.

2020 treated with caution. Pandemic-era disruptions like stimulus payments, eviction moratoriums, and lockdowns likely changed the normal relationship between unemployment and food bank demand in ways that would mislead the model. The available demand data starts October 2020, and that period was excluded where possible.

**Feature set**

FeatureTypeSourceprev_month_peopleLagCalculated from demand dataunemployment_rateEconomicFRED / BLSmean_temperatureWeatherOpen-Meteomin_temperatureWeatherOpen-Meteoprecipitation(mm)WeatherOpen-Meteoheating_seasonDerivedMonth column (Nov through Mar = 1)summer_breakDerivedMonth column (Jun through Aug = 1)region_encodedCategoricalLabel-encoded region namemonthTemporalDate columnyearTemporalDate column

**Model**

Random Forest Regressor (scikit-learn) with constraints to reduce overfitting on a small dataset:


n_estimators=100
max_depth=5
min_samples_leaf=3
random_state=42



**Project Structure**


food_bank_model.ipynb       - Full data pipeline and model training

food_bank_model.pkl         - Saved trained model

historical_averages.csv     - Per-region monthly averages for signal calculation

generate_prediction.py      - Monthly prediction script that outputs prediction.json

FoodBankOutputCS.csv        - Food bank demand data (manually collected)

WeathBerkshire.csv          - Weather data (Berkshire)

WeathFranklin.csv           - Weather data (Franklin)

WeathHampdenHolySpring.csv  - Weather data (Hampden, Springfield, Holyoke)

WeathHampshire.csv          - Weather data (Hampshire)

UnemployBerkshire.csv       - Unemployment data (Berkshire)

UnemployFranklin.csv        - Unemployment data (Franklin)

UnemployHampden.csv         - Unemployment data (Hampden)

UnemployHampshire.csv       - Unemployment data (Hampshire)

UnemployHolySpring.csv      - Unemployment data (Springfield, Holyoke)


**How to Run**

Install dependencies:

pip install pandas scikit-learn matplotlib joblib requests numpy openpyxl

Run the full notebook:
Open food_bank_model.ipynb in VS Code with the Jupyter extension and run all cells top to bottom. This loads all data, merges it, trains the model, and saves food_bank_model.pkl and historical_averages.csv.

Generate a monthly prediction:


Open generate_prediction.py
Update the manual section at the top:

next_month and next_year
prev_month_people for each region (from the food bank dashboard)
springfield_unem and holyoke_unem (from the MA LMI dashboard)



Run python generate_prediction.py
Copy the output prediction.json to your website folder


Weather and county unemployment pull automatically from Open-Meteo and BLS. Only the food bank demand numbers and Springfield/Holyoke unemployment need manual updates each month.


**Website Integration**

Predictions show up on the food bank's existing web bulletin board as color-coded High / Normal / Low signals per region. The signal is calculated by comparing the model's predicted value against the historical average for that region in that specific month, so "High" means higher than typical for that time of year rather than higher than an overall average.

The website reads a prediction.json file that gets regenerated monthly and re-uploaded. No backend server is needed.


**Limitations**


The dataset covers October 2020 through April 2026, which is small by ML standards. Model reliability will improve as more months of data accumulate.
Demand data is collected manually from a public dashboard with no export API and cannot be automated.
Because the lag feature dominates, the model tracks ongoing trends well but is less reliable at catching sudden spikes caused by a new external event like a plant closure or natural disaster. Staff judgment remains important in those situations.
The model should be retrained every 3 to 6 months as new data accumulates. The pipeline is already set up so retraining just means re-running the notebook with updated input files.
Springfield and Holyoke unemployment come from the MA LMI dashboard, which has no public API, so those two values still require a manual lookup each month.


**Author**

Adrian Paez

High school student, Williston Northampton School, 2027

Built as an independent research project in collaboration with The Food Bank of Western Massachusetts.

https://github.com/AdrianP428

https://www.linkedin.com/in/adrian-odulio-paez/

**Built With**

Python, pandas, scikit-learn, matplotlib, joblib, NumPy, requests, Open-Meteo API, FRED API, BLS LAUS API
