# Asthma Hospitalization and ER Visit Analysis for 2019 Data

## Overview

Asthma is a chronic respiratory track condition that is classified as ambulatory case sensitive. 
This designation indicates that teh vast majority of asthma-related hospitalizations and ER visits are preventable through timely and effective outpatient care.
Despite the availability of effective treatments, preventable hospitalizations persist, serving as a proxy for failures in outpatient management. The severity of the condition is exacerbated by upstream determinants such as poverty, environmental quality, and healthcare accessibility, transforming a manageable chronic condition into an acute medical emergency.
This research project serves to investigate the connections between socioeconomic status, environment and housing, access to care and how these factors play into asthma-related ER and hospitalization rates.

## Data Used

Information from several government agencies and non-profit organizations have been used during this project:
 - [American Community Survey (ACS)](https://www.census.gov/programs-surveys/acs.html)'s 5 year estimate private and public health insurance coverage and estimate population by age, gender and dependency per county 
(codes ACSST5Y2019.S2703, ACSST5Y2019.S2704 and ACSST5Y2019.S0101)
 - [Environmental Protection Agency (EPA)](https://www.epa.gov/)'s national-scale air toxic assesments (051646 on the [EPH](https://ephtracking.cdc.gov/) system)
 - [Centers for Disease Control and Prevention (CDC)](https://www.cdc.gov/index.html)'s age-adjusted ER visits and hospitalizations for asthma per 10,000 population (051823 and 053054 respectively on the EPH system)
 - [HRSA Data Warehouse](data.HRSA.gov)'s active medical doctor and respiratory therapist estimations per state, from AMA Physician Professional Data 2019
 - [County Health Rankings & Roadmaps](https://www.countyhealthrankings.org/reports/2019-county-health-rankings-key-findings-report)' 2019 County Health Rankings dataset's 'Ranked Measure Data' and 'Additional Measure Data' tabs
 - [Agency for Toxic Substances and Disease Registry (ATSDR)](https://www.atsdr.cdc.gov/place-health/index.html)'s CDC Social Vulnerability Index for 2018

Additional datasets were cleaned in the initial steps but left out during the analysis.

## Findings Summary

 - While diesel particulate matter concentration has a non-negligible correlation with asthma-related hospitalization (r≈0.40) and ER visits (r≈0.32), it is not even among the top 20 predictors for either according to a LASSO model.
Considering the documented impact air pollution has on respiratory health, this implies that other factors like demographics act as a comprehensive proxy for the cumulative environmental burden in the multivariate model, given that '% African American' and '% Hispanic' are among top predictors for both ER visits and hospitalizations
 - A lack of vehicle availability (r≈0.31) is the 4th strongest predictor for ER visits but not one of the top 20 predictors for hospitalization, signifying that the impact of a lack of car is not explainable by its association with poverty alone, and that a lack of transportation is a driver of ER usage independent of financial situation.
 - The number of single-parent households and uninsured population were collinear (r≈0.95), and the model selected the '# Single-Parent Households' feature as the number 1 predictor for both ER visits and hospitalizations. Given that our target is a rate (per 10,000) but the prediction is driven by absolute counts of single-parent households while essentially ignoring uninsured populations, we can consider population density as a very strong driver as single parents are typically much more dense in urban areas while uninsured people are more likely to be in rural areas.
 - Both ER visits and hospitalizations for asthma have high coefficients for HIV, diabetes, preventable hospital stays and injury deaths. This supports the hypothesis that the same structural failures impacting both ambulatory and emergency treatment of asthma are also impacting other medical issues, and these syndemic conditions should not be considered in isolation of one another.

## Methodology

### Data Cleaning

The different datasets used in this project follow different formatting schemes, so the first step was to clean the data and standardize the format. All the datasets were initially manually parsed and renamed for clarity, then loaded into a dictionary of Polars dataframes in Python for the processing. All the state and county name columns were normalized into 'State' and 'County' respectively, with the capitalization set to title case, and any trailing phrases like ' County' were removed from county names for unification purposes. To avoid issues with duplicate county names like 'Orange County', another column was created to host the combined 'County (State)' name. Several of these datasets utilized code phrases as the headers for the columns, and these headers had to be re-assigned using the tables included either along the datasets themselves or the dictionary tables stored in the supplementary documentation by extracting them via pdfplumber. The processed dataframes were then saved into a separate folder as separate files, with the dataframe storing asthma-related hospitalization and ER visits combined into a target dataframe.

### Statistical Analysis

