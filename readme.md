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
 - Racial disparities remain the dominant driver of asthma emergencies. The percentage of African American residents is the single strongest predictor of ER visits, outweighing economic factors like income, insurance, or housing quality.
 - Community safety is a critical health determinant. Violent crime rates remain a top-tier predictor for asthma attacks even when controlling for socioeconomic factors, suggesting that high-stress neighborhood environments directly contribute to poorer health outcomes.
 - Failures in the primary care safety net spill over into hospitals. High rates of preventable hospital stays and lack of vehicle access are the leading drivers of asthma hospitalizations, indicating that an inability to access routine care causes manageable conditions to become emergencies.
 - Financial housing stress hurts more than physical housing defects. Severe rent burden (paying >50% of income on housing) drives ER usage significantly more than physical crowding or lack of plumbing.

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
 - SVI 2018 -> 'svi_2018'

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

### Age-adjusted ER Visit Rate for Asthma per 10,000 People (Adjusted R^2: 0.5942)
| Feature                                                                                                                                       |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:----------------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Demographics => % African American                                                                                                            |     4.33779  |   5.2423   | 6.12996e-09 | 7.39978 |     0.535384  |
| (Household Composition/Disability) Percentage of persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [MP_AGE17]          |    -2.63719  |  -4.04776  | 4.42955e-13 | 2.82442 |    -0.189497  |
| Value_Pollutant: Formaldehyde                                                                                                                 |    -1.87501  |  -3.56924  | 5.00152e-07 | 4.60202 |     0.300947  |
| Adult smoking => % Smokers                                                                                                                    |    -1.40226  |  -3.42066  | 7.11681e-05 | 6.78954 |     0.102209  |
| Life expectancy => Life Expectancy                                                                                                            |    -2.43593  |  -3.21985  | 9.49332e-05 | 6.23202 |    -0.190749  |
| Median household income => Household Income                                                                                                   |    -1.55918  |  -2.86115  | 0.00114841  | 7.10298 |    -0.139338  |
| Demographics => % Hispanic                                                                                                                    |     1.80233  |   2.66906  | 0.000222521 | 4.78892 |     0.201783  |
| (Minority Status/Language) Sum of series for Minority Status/Languag e theme 2018 DESCRIPTION [SPL_THEME3]                                    |     2.29512  |   2.52238  | 0.00117973  | 5.54671 |     0.37892   |
| Population, All (County Level File)                                                                                                           |     1.73451  |   2.52057  | 4.79627e-07 | 2.28768 |     0.241582  |
| Severe housing problems => Severe Housing Cost Burden                                                                                         |     2.63243  |   2.4382   | 4.20978e-05 | 3.24383 |     0.436264  |
| HIV prevalence => HIV Prevalence Rate                                                                                                         |     2.70837  |   2.36988  | 0.000164683 | 3.62432 |     0.500156  |
| Income inequality => Income Ratio                                                                                                             |    -1.81826  |  -2.34211  | 4.33224e-05 | 3.00297 |     0.331754  |
| Poor mental health days => Mentally Unhealthy Days                                                                                            |     0.331287 |   2.11991  | 0.0160519   | 7.12252 |     0.274964  |
| Demographics => % Rural                                                                                                                       |    -0.97217  |  -2.05599  | 0.00604743  | 5.15049 |    -0.291233  |
| Diabetes prevalence => % Diabetic                                                                                                             |     1.78854  |   2.03553  | 0.0107507   | 5.84984 |     0.198298  |
| COVERAGE ALONE OR IN COMBINATION => Direct-purchase insurance alone or in combination => Under 19                                             |     1.13037  |   1.94768  | 1.53655e-05 | 1.85638 |    -0.137047  |
| Violent crime => Violent Crime Rate                                                                                                           |     2.4005   |   1.90545  | 0.000197374 | 2.40085 |     0.533786  |
| Longitude                                                                                                                                     |     1.54565  |   1.82143  | 0.00124818  | 2.9213  |     0.172522  |
| (Household Composition/Disability) Persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [M_AGE17]                         |     1.03155  |   1.70086  | 9.42872e-06 | 1.348   |     0.0367946 |
| (General) Adjunct variable - Percentage uninsured in the total civilian noninstitutiona lized population estimate, 2014-2018 ACS [EP_UNINSUR] |    -0.982217 |  -1.65275  | 0.00415515  | 3.05261 |     0.112076  |
| Demographics => % Asian                                                                                                                       |    -0.430163 |  -1.59945  | 0.0288446   | 4.92228 |     0.139659  |
| Adult obesity => % Obese                                                                                                                      |     0.8707   |   1.59806  | 0.0175173   | 4.1577  |     0.0522914 |
| Preventable hospital stays => Preventable Hosp. Rate                                                                                          |     0.545466 |   1.53783  | 0.000742315 | 1.90565 |     0.139013  |
| Unemployment => % Unemployed                                                                                                                  |    -0.152694 |  -1.49534  | 0.00664706  | 2.78761 |     0.220535  |
| (Socioeconomic) Flag - the percentage of persons with no high school diploma is in the 90th percentile (1 = yes, 0 = no) [F_NOHSDP]           |    -1.09028  |  -1.29045  | 0.00333881  | 1.77482 |     0.0250129 |
| Sexually transmitted infections => Chlamydia Rate                                                                                             |     0.4886   |   1.28482  | 0.0422174   | 3.67767 |     0.471192  |
| Demographics => % American Indian/Alaskan Native                                                                                              |    -0.819692 |  -1.24865  | 0.0128832   | 2.31576 |    -0.026051  |
| Access to exercise opportunities => % With Access                                                                                             |     0.729189 |   1.2111   | 0.0209073   | 2.52669 |     0.156239  |
| PRIVATE INSURANCE ALONE OR IN COMBINATION => Below 138 percent of the poverty threshold                                                       |    -0.971177 |  -1.16984  | 0.0367829   | 2.88504 |    -0.299506  |
| COVERAGE ALONE => Public insurance alone => VA care coverage alone                                                                            |     0.544756 |   0.798997 | 0.0364097   | 1.34049 |     0.0899133 |


### Age-adjusted Hospitalization Rate for Asthma per 10,000 People (Adjusted R^2: 0.4125)
| Feature                                                                                                   |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:----------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Preventable hospital stays => Preventable Hosp. Rate                                                      |    0.331111  |   0.546102 | 5.82658e-10 | 1.57173 |      0.260481 |
| Demographics => % Hispanic                                                                                |    0.105494  |   0.417186 | 1.6335e-05  | 1.91286 |      0.19989  |
| HIV prevalence => HIV Prevalence Rate                                                                     |    0.243806  |   0.394293 | 0.0006227   | 2.71985 |      0.390014 |
| (Housing Type/Transportation) Percentage of households with no vehicle available estimate [EP_NOVEH]      |    0.381568  |   0.368779 | 0.000789865 | 2.4731  |      0.443439 |
| Poor mental health days => Mentally Unhealthy Days                                                        |    0.0477193 |   0.362052 | 0.00353402  | 3.15981 |      0.290828 |
| Drug overdose deaths => # Drug Overdose Deaths                                                            |    0.306322  |   0.341133 | 0.000615188 | 2.03195 |      0.401212 |
| Longitude                                                                                                 |    0.0795509 |   0.29727  | 0.002921    | 2.04635 |      0.226687 |
| COVERAGE ALONE OR IN COMBINATION => Employer-based insurance alone or in combination => 65 years and over |    0.0370125 |   0.215301 | 0.0293835   | 2.00732 |      0.232214 |
| High school graduation => Graduation Rate                                                                 |   -0.0902439 |  -0.205585 | 0.0185363   | 1.56566 |     -0.26099  |
| Violent crime => Violent Crime Rate                                                                       |    0.240157  |   0.205062 | 0.0332489   | 1.90632 |      0.408334 |
| COVERAGE ALONE OR IN COMBINATION => Medicare coverage alone or in combination => Under 19                 |    0.0286332 |   0.153922 | 0.033657    | 1.07903 |      0.150341 |


The full tables for these scores and the correlation values is available in the files.

### Conclusions (table changed, needs adjustment)
 - Socioeconomic determinants outweigh clinical factors for ER utilization. The model explains significantly more variance in ER visit rates (Adjusted R^2 : 0.594) than in hospitalization rates (Adjusted R^2 : 0.413). This suggests that while hospitalization may be driven by individual disease progression, the frequency of ER visits is heavily dictated by external socioeconomic barriers, environmental constraints, and community infrastructure.

 - Racial disparities persist beyond economic controls. The percentage of African American residents is the single strongest predictor of asthma ER visits (OLS Coef: 5.24, p < 0.001). Crucially, this variable retains its high magnitude even after the model controls for median income, insurance coverage, and housing quality. This implies that economic variables alone cannot account for the disparity; structural inequities specific to race likely play a dominant role in asthma morbidity. Similarly, the percentage of Hispanic residents is a robust positive predictor for both ER visits (OLS Coef: 2.67) and hospitalization rates (OLS Coef: 0.42). This suggests that even when controlling for wealth and geography, structural factors specific to minority communities drive emergency asthma care.

 - Violent crime and community stress act as health determinants. Violent Crime Rate was chosen by the model as a highly significant predictor for both ER visits and hospitalizations (OLS Coef: 1.91 for ER, 0.21 for hospitalization, p < 0.05 for both). High-stress neighborhoods likely contribute to the physiological asthma triggers (stress response) while simultaneously creating physical barriers that discourage residents from walking to local primary care centers, leading to reliance on emergency services.

 - Economic housing stress outweighs physical housing defects. For ER visits, severe housing cost burden (households paying >50% of income on housing) is a top-tier predictor (OLS Coef: 2.44), whereas physical crowding issues were not even picked by the model as relevant. This indicates that the financial trade-offs required to maintain housing, such as rationing medication to pay rent, are more detrimental to asthma control than the physical condition of the building itself.

 - Transportation barriers drive hospitalization. For hospitalization rates, the percentage of households with no vehicle is a key driver (OLS Coef: 0.37, p < 0.001). This suggests that a lack of personal transportation acts as a fundamental barrier to routine outpatient care, causing manageable conditions to escalate until they require inpatient hospitalization.

 - Primary care failure spills over into asthma outcomes. The rate of preventable hospital stays is the strongest predictor of asthma hospitalizations (OLS Coef: 0.55). This confirms that communities with weak primary care safety nets where patients are hospitalized for manageable conditions like dehydration or hypertension see a direct correlation with increased asthma severity.

 - Rural areas show a strong negative association with ER visits (OLS Coef: -2.06), and factors typically associated with poor health (like smoking and formaldehyde) also displayed negative coefficients. Rather than suggesting these factors are protective, this likely reflects "distance decay" and under-utilization: Residents in remote or heavily polluted industrial areas may face significant geographic barriers to reaching an ER, leading to lower documented visit rates despite high potential morbidity.

### Limitations
 - This analysis models age-adjusted asthma-related ER visit and hospitalization rates per 10,000 residents rather than outcomes conditional on asthma prevalence. Because county-level asthma diagnosis or prevalence data were not included, the study cannot distinguish whether observed associations reflect higher asthma prevalence, greater disease severity among individuals with asthma, or differences in healthcare utilization and access. Lower observed utilization in certain populations, including uninsured or rural communities, may therefore indicate underuse of services rather than lower asthma morbidity. State-level effects capture broad regional context but may mask substantial within-state heterogeneity in healthcare policy, infrastructure, and environmental conditions. As a result, some socioeconomic and environmental variables may appear to increase asthma morbidity when they instead influence care-seeking behavior or underlying case counts.

 - All variables are measured at the county level, introducing the possibility of ecological fallacy. Population-level associations may not hold at the individual level, and unmeasured individual characteristics such as asthma severity, medication adherence, household smoking exposure, and occupational or school-based triggers may confound the observed relationships. In addition, the cross-sectional design limits causal inference and does not allow for assessment of temporal relationships or disease progression.

 - Several environmental pollutant variables exhibit substantial multicollinearity, reflecting their close association with urban density and traffic-related exposures. Although LASSO regularization reduces overfitting, it may obscure or redistribute effects among correlated predictors, making it difficult to isolate the independent contribution of specific pollutants. Consequently, nonsignificant or counterintuitive pollutant coefficients should not be interpreted as evidence of no environmental effect on asthma morbidity.

 - While spatial information (latitude, longitude, and state indicators) was included to control for regional structure, cross‑validation was performed using random folds. As a result, predictive performance may be optimistic in the presence of spatial autocorrelation, and findings should be interpreted as descriptive associations rather than fully spatially generalizable effects

