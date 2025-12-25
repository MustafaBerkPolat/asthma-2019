# Socioeconomic and Environmental Determinants of Asthma ER Visits and Hospitalizations (U.S. Counties, 2019)

## Overview

Asthma is a chronic respiratory tract condition that is classified as ambulatory case sensitive. 
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
Additional datasets were cleaned in the initial steps but left out during the analysis due to redundancy.

## Key Findings
 - Socioeconomic and housing stressors explain more variance in asthma ER visits than clinical or environmental factors.
 - Racial disparities in ER utilization persist and are very significant even after controlling for income, insurance, and housing.
 - Transportation access is a major predictor of both ER visits and hospitalizations.
 - State-level effects suggest policy and regional context matter beyond county characteristics.

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

Afterwards, all the datasets were combined into a single dataframe to host the remaining 352 features. After this merging, features that were missing from at least 30% of the counties were removed, leaving 281 features. The asthma-related hospitalization and ER visit were then excluded, and the remaining features were imputed using the 5 nearest neighbors (KNN) for measures that are likely to be impacted by state-level laws by isolating the counties into state groups to prevent nearby counties from other states affecting the imputing, and multiple imputation by chained equations (MICE) for features that are not as likely to be impacted by state-level laws. To ensure location data still influenced the MICE model, the state information was encoded into dummies, and the latitude/longitude information from the USCB's gazetteer file was used. To identify the 'hard constraint' features to fill in with KNN imputing, a function was defined to select columns containing the following keywords:

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

The 'Asthma' category's features only included the hospitalization and ER visit rates, and their correlation coefficients to the entries in the other categories were then calculated. 
Feature selection was performed using LASSO regression with 5-fold cross‑validated penalty selection, leaving us with 84 features for ER visit rates and 33 features for hospitalization rates. All predictors were standardized prior to modeling. The optimal regularization parameter was chosen to minimize cross‑validated mean squared error. Selected predictors were subsequently refit using ordinary least squares to obtain unpenalized coefficient estimates, confidence intervals, and p‑values. Variance inflation factors were computed to assess residual multicollinearity. The results that have a p-value less than 0.05 and a variance inflation factor less than 10 in the order of highest absolute OLS coefficient are as follows, with the full tables available under the project files.

The feature names are left with minimal changes from the datasets. The 'estimate MOE' features represent the margin of error for the relevant feature.

### Age-adjusted ER Visit Rate for Asthma per 10,000 People (Adjusted R^2: 0.6119)
| Feature                                                                                                                                       |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:----------------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Demographics => % African American                                                                                                            |     5.90747  |   6.45041  | 2.59575e-10 | 9.88461 |    0.535384   |
| (Household Composition/Disability) Percentage of persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [MP_AGE17]          |    -2.44867  |  -3.33277  | 8.21575e-10 | 2.79834 |   -0.18977    |
| Median household income => Household Income                                                                                                   |    -2.02376  |  -3.1244   | 0.000781937 | 8.29551 |   -0.139338   |
| (Minority Status/Language) Sum of series for Minority Status/Languag e theme 2018 DESCRIPTION [SPL_THEME3]                                    |     1.9096   |   2.7612   | 0.000306528 | 5.60711 |    0.37892    |
| State_Minnesota                                                                                                                               |    -2.38099  |  -2.71482  | 1.07503e-08 | 2.14495 |   -0.19243    |
| State_Louisiana                                                                                                                               |    -1.96718  |  -2.61273  | 5.19813e-06 | 3.14205 |    0.096494   |
| Adult smoking => % Smokers                                                                                                                    |    -0.753919 |  -2.54391  | 0.00413937  | 7.55559 |    0.102209   |
| (Housing Type/Transportation) Percentage of households with no vehicle available estimate [EP_NOVEH]                                          |     2.60954  |   2.43923  | 0.0142162   | 9.50855 |    0.446551   |
| Income inequality => Income Ratio                                                                                                             |    -1.48532  |  -2.31122  | 9.9615e-05  | 3.37762 |    0.331754   |
| HIV prevalence => HIV Prevalence Rate                                                                                                         |     2.2755   |   2.16262  | 0.000960942 | 4.1143  |    0.505364   |
| Life expectancy => Life Expectancy                                                                                                            |    -1.85205  |  -2.07679  | 0.0103308   | 6.29848 |   -0.190332   |
| Respiratory Therapist per 100,000 population                                                                                                  |    -1.64723  |  -2.05239  | 0.0113422   | 6.31107 |   -0.1518     |
| Diabetes prevalence => % Diabetic                                                                                                             |     1.28156  |   1.90583  | 0.0146024   | 5.85079 |    0.198298   |
| Preventable hospital stays => Preventable Hosp. Rate                                                                                          |     1.31357  |   1.86485  | 0.000115683 | 2.24093 |    0.139559   |
| (General) Adjunct variable - Percentage uninsured in the total civilian noninstitutiona lized population estimate, 2014-2018 ACS [EP_UNINSUR] |    -1.34382  |  -1.8488   | 0.00406763  | 3.97528 |    0.112136   |
| COVERAGE ALONE OR IN COMBINATION => Direct-purchase insurance alone or in combination => Under 19                                             |     0.903299 |   1.82837  | 3.81424e-05 | 1.88604 |   -0.137047   |
| State_Massachusetts                                                                                                                           |     1.39874  |   1.76324  | 0.000151883 | 2.07539 |    0.132306   |
| Demographics => % Asian                                                                                                                       |    -0.382374 |  -1.72936  | 0.0185692   | 5.1852  |    0.139659   |
| Demographics => % Rural                                                                                                                       |    -0.751175 |  -1.66067  | 0.0195136   | 4.85763 |   -0.291233   |
| (Socioeconomic) Flag - the percentage of persons with no high school diploma is in the 90th percentile (1 = yes, 0 = no) [F_NOHSDP]           |    -1.24328  |  -1.6356   | 0.000222208 | 1.87997 |    0.0250118  |
| Severe housing problems => Severe Housing Cost Burden                                                                                         |     1.84245  |   1.62568  | 0.00804503  | 3.61349 |    0.436264   |
| Sexually transmitted infections => Chlamydia Rate                                                                                             |     0.679757 |   1.59491  | 0.0127527   | 3.93909 |    0.481207   |
| State_Maine                                                                                                                                   |     1.04729  |   1.52384  | 0.000164743 | 1.56687 |    0.0305938  |
| (Housing Type/Transportation) Sum of series for Housing Type/ Transportation theme [SPL_THEME4]                                               |    -0.942657 |  -1.47508  | 0.0481629   | 5.35765 |    0.32746    |
| (Household Composition/Disability) Persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [M_AGE17]                         |     0.79669  |   1.36236  | 0.000345355 | 1.38888 |    0.0366692  |
| PRIVATE INSURANCE ALONE OR IN COMBINATION => 75 years and over                                                                                |    -0.158222 |  -1.17822  | 0.0185373   | 2.40554 |   -0.242517   |
| Violent crime => Violent Crime Rate                                                                                                           |     1.85649  |   1.16682  | 0.0296858   | 2.76785 |    0.534607   |
| State_Connecticut                                                                                                                             |     0.510532 |   0.800747 | 0.0305838   | 1.31785 |    0.0497299  |
| State_New Hampshire                                                                                                                           |     0.319998 |   0.7986   | 0.0327562   | 1.34469 |    0.00417388 |



### Age-adjusted Hospitalization Rate for Asthma per 10,000 People (Adjusted R^2: 0.4524)
| Feature                                                                                                                              |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:-------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Preventable hospital stays => Preventable Hosp. Rate                                                                                 |   0.414405   |   0.698937 | 8.16587e-14 | 1.88489 |     0.260846  |
| (Housing Type/Transportation) Percentage of households with no vehicle available estimate [EP_NOVEH]                                 |   0.405624   |   0.429859 | 6.64986e-05 | 2.5469  |     0.44344   |
| HIV prevalence => HIV Prevalence Rate                                                                                                |   0.277723   |   0.392361 | 0.00055575  | 2.8375  |     0.398091  |
| State_Minnesota                                                                                                                      |  -0.180722   |  -0.383413 | 1.75705e-05 | 1.74631 |    -0.173672  |
| Low birthweight => % LBW                                                                                                             |   0.0574876  |   0.377285 | 0.000279847 | 2.3674  |     0.237602  |
| Demographics => % Hispanic                                                                                                           |   0.116857   |   0.364236 | 0.00110092  | 2.73831 |     0.19989   |
| High school graduation => Graduation Rate                                                                                            |  -0.139854   |  -0.357841 | 5.16935e-05 | 1.71322 |    -0.260989  |
| State_Louisiana                                                                                                                      |  -0.0753757  |  -0.336877 | 1.70954e-05 | 1.34429 |     0.0217947 |
| State_Missouri                                                                                                                       |   0.0845323  |   0.327851 | 1.27768e-05 | 1.23577 |     0.0620509 |
| (Household Composition/Disability) Percentage of persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [MP_AGE17] |  -0.175769   |  -0.299473 | 0.037934    | 4.58979 |    -0.259581  |
| State_Kansas                                                                                                                         |   0.0464431  |   0.292978 | 0.000277489 | 1.42587 |    -0.0221605 |
| COVERAGE ALONE OR IN COMBINATION => Employer-based insurance alone or in combination => 65 years and over                            |   0.0570474  |   0.216632 | 0.0246221   | 2.048   |     0.232214  |
| Longitude                                                                                                                            |   0.090324   |   0.206533 | 0.0386515   | 2.19931 |     0.225984  |
| State_Massachusetts                                                                                                                  |   0.00227979 |   0.143542 | 0.0441842   | 1.12192 |     0.114854  |


The full tables for these scores and the correlation values is available in the files.

### Conclusions

 - Socioeconomic determinants outweigh clinical factors for ER utilization. The model explains significantly more variance in ER visit rates (Adjusted R^2 : 0.612) than in hospitalization rates (Adjusted R^2 : 0.452). This suggests that while hospitalization may be driven by individual disease progression, the frequency of ER visits is heavily dictated by external socioeconomic barriers, environmental constraints, and community infrastructure.

 - Racial disparities persist beyond economic controls. The percentage of African American residents is the single strongest predictor of asthma ER visits (LASSO Coef: 5.90, P < 0.001). Crucially, this variable retains its high magnitude even after the model controls for median income, insurance coverage, and housing quality. This implies that economic variables alone cannot account for the disparity; structural inequities specific to race likely play a dominant role in asthma morbidity.

 - The model highlights a divergence between rural and urban minority populations:
   - Rural Populations: Show a robust negative association with ER visits (Coef: -0.75), likely driven by lower population density (reduced viral transmission) and the "distance decay" effect, where geographic distance from hospitals discourages ER utilization for non-critical attacks.
   - Asian Populations: Demonstrate a classic statistical suppression effect. While raw data shows a positive correlation with ER visits (likely due to residence in dense urban centers), the regression reveals a negative coefficient (Coef: -0.38) once urban stressors are controlled for. This suggests the demographic itself has protective health factors that are masked by their urban environment.

 - Transportation is a critical barrier to care. Lack of vehicle availability is a significant positive predictor for both ER visits and hospitalizations (P<0.05, VIF < 10). This indicates that transportation acts as a fundamental "social determinants of health" gateway; without personal transport, patients likely struggle to access routine outpatient care, causing manageable conditions to escalate into emergencies.

 - The results point to two distinct failures in the healthcare safety net:
   - Both % Uninsured and # Uninsured show negative coefficients for ER visits. This suggests "financial avoidance," where uninsured patients delay or avoid emergency care due to cost fears, rather than a lack of medical need.
   - The rate of Preventable Hospital Stays is a strong positive predictor of asthma ER rates. This confirms that communities with ineffective primary care infrastructure (where patients are hospitalized for manageable conditions) see a direct spillover effect into asthma emergency visits.

 - Economic housing stress trumps physical crowding. Severe Housing Cost Burden (paying >50% of income on housing) is a significant positive driver of ER visits (Coef: 1.84, P=0.008). Notably, this economic variable proved statistically significant while physical overcrowding did not (P > 0.05), suggesting that the financial trade-offs required to maintain housing (e.g., cutting medication costs) are more detrimental to asthma control than the physical arrangement of the household.

 - State-level geographic controls remained significant even after accounting for demographics. Minnesota and Louisiana act as protective factors (negative coefficients), while Massachusetts is associated with higher rates. This indicates that state-level public health policies, Medicaid expansion specifics, or regional climate variations exert an independent influence on asthma outcomes that local socioeconomic data cannot fully explain.

### Limitations
 - This analysis models age-adjusted asthma-related ER visit and hospitalization rates per 10,000 residents rather than outcomes conditional on asthma prevalence. Because county-level asthma diagnosis or prevalence data were not included, the study cannot distinguish whether observed associations reflect higher asthma prevalence, greater disease severity among individuals with asthma, or differences in healthcare utilization and access. Lower observed utilization in certain populations, including uninsured or rural communities, may therefore indicate underuse of services rather than lower asthma morbidity. State-level effects capture broad regional context but may mask substantial within-state heterogeneity in healthcare policy, infrastructure, and environmental conditions. As a result, some socioeconomic and environmental variables may appear to increase asthma morbidity when they instead influence care-seeking behavior or underlying case counts.

 - All variables are measured at the county level, introducing the possibility of ecological fallacy. Population-level associations may not hold at the individual level, and unmeasured individual characteristics such as asthma severity, medication adherence, household smoking exposure, and occupational or school-based triggers may confound the observed relationships. In addition, the cross-sectional design limits causal inference and does not allow for assessment of temporal relationships or disease progression.

 - Several environmental pollutant variables exhibit substantial multicollinearity, reflecting their close association with urban density and traffic-related exposures. Although LASSO regularization reduces overfitting, it may obscure or redistribute effects among correlated predictors, making it difficult to isolate the independent contribution of specific pollutants. Consequently, nonsignificant or counterintuitive pollutant coefficients should not be interpreted as evidence of no environmental effect on asthma morbidity.

 - While spatial information (latitude, longitude, and state indicators) was included to control for regional structure, cross‑validation was performed using random folds. As a result, predictive performance may be optimistic in the presence of spatial autocorrelation, and findings should be interpreted as descriptive associations rather than fully spatially generalizable effects

