# Asthma Hospitalization and ER Visit Analysis for 2019 Data

## Overview

Asthma is a chronic respiratory track condition that is classified as ambulatory case sensitive. 
This designation indicates that the vast majority of asthma-related hospitalizations and ER visits are preventable through timely and effective outpatient care.
Despite the availability of effective treatments, preventable hospitalizations persist, serving as a proxy for failures in outpatient management. The severity of the condition is exacerbated by upstream determinants such as poverty, environmental quality, and healthcare accessibility, transforming a manageable chronic condition into an acute medical emergency.
This research project serves to investigate the connections between socioeconomic status, environment and housing, access to care and how these factors play into asthma-related ER and hospitalization rates.

## Data Used

Information from several government agencies and non-profit organizations have been used during this project:
 - [American Community Survey (ACS)](https://www.census.gov/programs-surveys/acs.html)'s 5 year estimate private and public health insurance coverage and estimate population by age, gender and dependency per county 
(codes ACSST5Y2019.S2703, ACSST5Y2019.S2704 and ACSST5Y2019.S0101)
 - [Environmental Protection Agency (EPA)](https://www.epa.gov/)'s national-scale air toxic assesments (051646 on the [EPH](https://ephtracking.cdc.gov/DataExplorer/?c=11&i=81&m=-1) system)
 - [Centers for Disease Control and Prevention (CDC)](https://www.cdc.gov/index.html)'s age-adjusted ER visits and hospitalizations for asthma per 10,000 population (051823 and 053054 respectively on the EPH system)
 - [HRSA Data Warehouse](https://data.hrsa.gov/topics/health-workforce/nchwa/ahrf)'s active medical doctor and respiratory therapist estimations per state, from AMA Physician Professional Data 2019
 - [County Health Rankings & Roadmaps](https://www.countyhealthrankings.org/reports/2019-county-health-rankings-key-findings-report)' 2019 County Health Rankings dataset's 'Ranked Measure Data' and 'Additional Measure Data' tabs
 - [Agency for Toxic Substances and Disease Registry (ATSDR)](https://www.atsdr.cdc.gov/place-health/index.html)'s CDC Social Vulnerability Index for 2018
 - [United States Census Bureau](https://www.census.gov/geographies/reference-files/time-series/geo/gazetteer-files.html)'s census tracts for latitude and longitude information
Additional datasets were cleaned in the initial steps but left out during the analysis.

## Findings Summary

## Methodology

### Data Cleaning and Merging

The different datasets used in this project follow different formatting schemes, so the first step was to clean the data and standardize the format. All the datasets' names were initially manually parsed and renamed for clarity, then loaded into a dictionary of Polars dataframes in Python for the processing. The used datasets as they were downloaded and the names given to them are as follows:
 - 2025_Gaz_counties_national -> 'lat_long', to serve as the geographical position data for the imputing of missing measurements later on
 - 5 Year estimate population by age, gender and dependency per county (ACSST5Y2019.S0101) -> '5yr_demographic', for detailed population estimates
 - 5 Year estimate public health insurance coverage (ACSST5Y2019.S2704) ->'5yr_public_insurance', for public insurance coverage data
 - 5 Year estimate private health insurance coverage (ACSST5Y2019.S2703) -> '5yr_private_insurance', for private insurance coverage data
(the 5-year estimates were used for the above 3 measures to improve reliability and coverage for small/urban counties)
 - Pollutant per county (_051646) -> 'pollution', to track the micrograms of air pollutants per cubic meter
 - Asthma hospitalizations (_053054) -> 'asthma_hospitalization'
 - Emergency asthma visits (_051823) -> 'emergency_asthma'
The emergency asthma visit and asthma hospitalization dataframes were combined into one before saving
 - 'Medical doctors per state' and 'Respiratory therapists per state' -> 'md_per_state' and 'resp_therapist_per_state'
 - 2019 County Health Rankings Data - v3 -> two separate dataframes; 'county_health_measures' to store directly health-related data like infant mortality rates and HIV prevalence, and 'county_additional_measures' to store other information like homicides and disconnected youth. These dataframes went through an additional step of cleaning afterwards since they included overlapping information
 - svi 2018 -> 'svi_2018'

All the state and county name columns were normalized into 'State' and 'County' respectively, with the capitalization set to title case, and any trailing phrases like ' County', ' Parish' and ' Borough' were standardized into ' County' for unification purposes. A general-purpose function was defined to catch the 'duplicate' county entries for Virgninian independent cities to ensure that they were appropriately suffixed with ' City' when referring to independent cities and ' County' otherwise, to ensure that upon merging the datasets the available information was appropriately assigned to the correct census units. To avoid issues with duplicate county names like 'Orange County', another column was created to host the combined 'County (State)' name. Several of these datasets utilized code phrases as the headers for the columns, and these headers had to be re-assigned using the tables included either along the datasets themselves or the dictionary tables stored in the supplementary documentation by extracting them via pdfplumber. The processed dataframes were then saved into a separate folder as separate files, with the dataframe storing asthma-related hospitalization and ER visits combined into a target dataframe.

### Data Pre-processing

Before the statistical analysis, missing entries needed to be handled. As this project included datasets from numerous sources, the intersection of counties with no missing features was very limited. All the cleaned dataframes from the previous step were loaded and joined onto the dataframe containing asthma-related data, dropping counties that don't have any information for either of the asthma-related metrics. The end result was 1614 counties (or equivalent units) out of 3240 that had measurements for either ER visits or hospitalizations, 1512 counties with ER visit rate data, 1096 counties with hospitalization data, and 1037 counties with data for both at once. For the independent datasets, a simple collinearity test was performed to drop features with correlation coefficients over 0.98, resulting in a total of 47 features dropped. Most notably, 17 out of 30 features were removed from the '5yr_demographic' dataframe, and 27 out of 120 features were dropped from the 'svi_2018' dataframe.

Afterwards, all the datasets were combined into a single dataframe to host the remaining 352 features. After this merging, features that were missing from at least 30% of the counties were removed, leaving 281 features. The asthma-related hospitalization and ER visit were then excluded, and the remaining features were imputed using the 5 nearest neighbors (KNN) for measures that are likely to be impacted by state-level laws, and multiple imputation by chained equations (MICE) for other features. To ensure location data still influenced the MICE model, the state information was encoded into dummies, and the latitude/longitude information from the USCB's gazetteer files was used. To identify the 'hard constraint' features to fill in with KNN imputing, a function was defined to select columns containing the following keywords:

 - Health policy-related features:  'medicaid', 'insurance', 'enrollment', 'medicare'
 - Fiscal policy-related features:  'tax', 'spending', 'funding'
 - Education-related features:      'graduation", 'proficiency', 'score', 'college', 'education'
 - Legal/Reporting standards:       'crime', 'arrest'


In the next step, the existing features were separated into groups based on the presence of certain subwords within their headers, with the groups higher on the list taking precedence in the case a column includes subwords from multiple groups. This was done for better readability and did not directly impact the analysis.

 - 'Asthma':         'asthma'
 - 'Pollution':      'pollutant', 'pollution'
 - 'Care Provider':  'respiratory therapist', 'm.d.', 'physician', 'hospital'
 - 'Insurance':      'insur', 'coverage', 'medicaid', 'private', 'medicare']
 - 'Other Health':   'physical', 'smok', 'disability', 'disabl', 'medical', 'mental', 
                     'activity', 'hiv', 'diabet', 'food', 'nutrition', 'injury', 
                     'mortality', 'obese', 'life expectancy', 'birthweight'
 - 'Socioeconomic':  'income', 'poverty', 'unemployment', 'education', 'socioeconomic', 
                     'minority', 'demographic', 'crime', 'college', 'education', 'social', 
                     'alone', 'single'
 - 'Housing':        'rent', 'value', 'owner', 'vacant', 'crowd', 'hous'

### Statistical Analysis

The 'Asthma' category's features only included the hospitalization and ER visit rates, and their correlation coefficients to the entries in the other categories were then calculated. Using the scikit-learn and statsmodel Python packages, the LASSO model was used to select statistically relevant features from the other categories, leaving us with 84 features for ER visit rates and 33 features for hospitalization rates. For these features, the variance inflation factors and p-values were calculated to spot significant relationships. The results that have a p-value less than 0.05 and a variance inflation factor less than 10 in the order of highest absolute OLS coefficient are as follows, with the full tables available under the project files.

### Age-adjusted ER Visit Rate for Asthma per 10,000 People (Adjusted R^2: 0.6119)
| Feature                                                                                                                                       |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:----------------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Demographics => % African American                                                                                                            |     5.90676  |    6.44776 | 2.63354e-10 | 9.8834  |    0.535384   |
| (Household Composition/Disability) Percentage of persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [MP_AGE17]          |    -2.4505   |   -3.33509 | 8.04519e-10 | 2.7991  |   -0.189799   |
| Median household income => Household Income                                                                                                   |    -2.02355  |   -3.12503 | 0.000780164 | 8.2956  |   -0.139338   |
| (Minority Status/Language) Sum of series for Minority Status/Languag e theme 2018 DESCRIPTION [SPL_THEME3]                                    |     1.90788  |    2.76183 | 0.000305527 | 5.60696 |    0.37892    |
| State_Minnesota                                                                                                                               |    -2.38082  |   -2.71481 | 1.07588e-08 | 2.14499 |   -0.19243    |
| State_Louisiana                                                                                                                               |    -1.96681  |   -2.61201 | 5.23214e-06 | 3.14218 |    0.096494   |
| Adult smoking => % Smokers                                                                                                                    |    -0.754752 |   -2.54452 | 0.00413137  | 7.55587 |    0.102209   |
| (Housing Type/Transportation) Percentage of households with no vehicle available estimate [EP_NOVEH]                                          |     2.60984  |    2.43749 | 0.014284    | 9.50809 |    0.446558   |
| Income inequality => Income Ratio                                                                                                             |    -1.48467  |   -2.31058 | 0.000100059 | 3.37758 |    0.331754   |
| HIV prevalence => HIV Prevalence Rate                                                                                                         |     2.27494  |    2.16356 | 0.000956432 | 4.11452 |    0.505187   |
| Life expectancy => Life Expectancy                                                                                                            |    -1.85157  |   -2.07568 | 0.0103676   | 6.29773 |   -0.190377   |
| Respiratory Therapist per 100,000 population                                                                                                  |    -1.6477   |   -2.05366 | 0.0113006   | 6.3123  |   -0.151788   |
| Diabetes prevalence => % Diabetic                                                                                                             |     1.28199  |    1.9061  | 0.0145906   | 5.85095 |    0.198298   |
| Preventable hospital stays => Preventable Hosp. Rate                                                                                          |     1.31347  |    1.86509 | 0.000115463 | 2.24094 |    0.139559   |
| (General) Adjunct variable - Percentage uninsured in the total civilian noninstitutiona lized population estimate, 2014-2018 ACS [EP_UNINSUR] |    -1.34342  |   -1.84844 | 0.00407513  | 3.9753  |    0.112138   |
| COVERAGE ALONE OR IN COMBINATION => Direct-purchase insurance alone or in combination => Under 19                                             |     0.903447 |    1.82829 | 3.81765e-05 | 1.88605 |   -0.137047   |
| State_Massachusetts                                                                                                                           |     1.39833  |    1.76276 | 0.000152525 | 2.07538 |    0.132306   |
| Demographics => % Asian                                                                                                                       |    -0.382022 |   -1.72872 | 0.0186167   | 5.18545 |    0.139659   |
| Demographics => % Rural                                                                                                                       |    -0.752655 |   -1.6622  | 0.019406    | 4.85793 |   -0.291233   |
| (Socioeconomic) Flag - the percentage of persons with no high school diploma is in the 90th percentile (1 = yes, 0 = no) [F_NOHSDP]           |    -1.2424   |   -1.63455 | 0.000224196 | 1.87985 |    0.0250259  |
| Severe housing problems => Severe Housing Cost Burden                                                                                         |     1.84187  |    1.62508 | 0.00806945  | 3.6136  |    0.436264   |
| Sexually transmitted infections => Chlamydia Rate                                                                                             |     0.679995 |    1.59526 | 0.0127393   | 3.93956 |    0.481205   |
| State_Maine                                                                                                                                   |     1.04709  |    1.5238  | 0.000164818 | 1.56686 |    0.0305938  |
| (Housing Type/Transportation) Sum of series for Housing Type/ Transportation theme [SPL_THEME4]                                               |    -0.944056 |   -1.47672 | 0.0479158   | 5.35759 |    0.32746    |
| (Household Composition/Disability) Persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [M_AGE17]                         |     0.796432 |    1.36216 | 0.000346239 | 1.38897 |    0.0366445  |
| PRIVATE INSURANCE ALONE OR IN COMBINATION => 75 years and over                                                                                |    -0.158058 |   -1.17788 | 0.0185716   | 2.40552 |   -0.242517   |
| Violent crime => Violent Crime Rate                                                                                                           |     1.85719  |    1.16772 | 0.0295568   | 2.7677  |    0.534609   |
| State_Connecticut                                                                                                                             |     0.510531 |    0.80066 | 0.0306032   | 1.31785 |    0.0497299  |
| State_New Hampshire                                                                                                                           |     0.31981  |    0.79839 | 0.0328034   | 1.34469 |    0.00417388 |



### Age-adjusted Hospitalization Rate for Asthma per 10,000 People (Adjusted R^2: 0.4524)
Age-adjusted Hospitalization Rate for Asthma per 10,000 People
| Feature                                                                                                                              |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:-------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Preventable hospital stays => Preventable Hosp. Rate                                                                                 |   0.414381   |   0.698902 | 8.18192e-14 | 1.88486 |     0.260849  |
| (Housing Type/Transportation) Percentage of households with no vehicle available estimate [EP_NOVEH]                                 |   0.405751   |   0.429827 | 6.65743e-05 | 2.54688 |     0.443441  |
| HIV prevalence => HIV Prevalence Rate                                                                                                |   0.277306   |   0.392329 | 0.000556275 | 2.83749 |     0.397855  |
| State_Minnesota                                                                                                                      |  -0.180774   |  -0.383435 | 1.75469e-05 | 1.74628 |    -0.173672  |
| Low birthweight => % LBW                                                                                                             |   0.0575854  |   0.377329 | 0.000279213 | 2.3672  |     0.237626  |
| Demographics => % Hispanic                                                                                                           |   0.116947   |   0.364335 | 0.00109852  | 2.73878 |     0.19989   |
| High school graduation => Graduation Rate                                                                                            |  -0.139905   |  -0.357852 | 5.16687e-05 | 1.71323 |    -0.260989  |
| State_Louisiana                                                                                                                      |  -0.0753505  |  -0.336887 | 1.70854e-05 | 1.3443  |     0.0217947 |
| State_Missouri                                                                                                                       |   0.0844992  |   0.327821 | 1.27995e-05 | 1.23577 |     0.0620509 |
| (Household Composition/Disability) Percentage of persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [MP_AGE17] |  -0.175764   |  -0.299598 | 0.0378399   | 4.58913 |    -0.259582  |
| State_Kansas                                                                                                                         |   0.046385   |   0.292928 | 0.000278124 | 1.42586 |    -0.0221605 |
| COVERAGE ALONE OR IN COMBINATION => Employer-based insurance alone or in combination => 65 years and over                            |   0.0569604  |   0.216616 | 0.0246332   | 2.04802 |     0.232214  |
| Longitude                                                                                                                            |   0.0904691  |   0.20663  | 0.0385557   | 2.19921 |     0.225974  |
| State_Massachusetts                                                                                                                  |   0.00226196 |   0.143528 | 0.044201    | 1.12188 |     0.114854  |


The full tables for these scores and the correlation values is available in the files.

### Conclusions
 - For every unit increase in the percentage of African American population in a county, the age-adjusted ER vist rate rises considerably (LASSO coefficient of 5.37, OLS coefficient of 6.44, p-value of 8.39x10^(-14)). While the variane inflation factor being 7.91 indicates that this particular measure has high collinearity with a multitude of features (most of which are most likely to do with socioeconomic factors), the extremely low p-value implies the existence of strong systemic disparities that African American communities face that expose them to increased risk, such as environmental racism.
 - The percentage of households with no vehicle available has a LASSO coefficient of 2.03, p-value of 0.01, correlation of 0.45 and VIF of 7.93 for ER visits. A lack of car might be causing families to miss the routine management visits that serve as the ambulatory care for asthma, causing severe attacks that demand ER vists later on. Alternatively, families that lack cars might live in urban areas where hospitals are closer by, making regular controls easier to manage. Finally, it could be that a lack of vehicles is a signifier of low income, which would also deter families from routine checkups and push them to rely more on ER services.
 - Life expectancy acts as a proxy for the overall health and wealth of a community, with a LASSO coefficient of -1.23, p-value of 0.005, correlation of -0.18 and VIF of 5.29.
 - Areas with higher single-parent household counts also see higher hospitalization rates, with a LASSO score of 0.70, p-value of 7.54x10^(-11) and correlation of 0.31, though a VIF score of 36.69 implies this statistic is highly connected to other features, most likely socioeconomic factors. Single parents are much more common in urban areas as opposed to rural ones, and the financial stress of having to raise a child with a single income, the relative lack of care children face, and the environmental factors like exposure to higher pollution and mold stemming from being confined to cheaper housing can all contribute to the severity of asthma, particularly in younger children.
