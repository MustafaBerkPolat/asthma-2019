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
 - [Environmental Protection Agency (EPA)](https://www.epa.gov/)'s national-scale air toxic assesments (051646 on the [EPH](https://ephtracking.cdc.gov/) system)
 - [Centers for Disease Control and Prevention (CDC)](https://www.cdc.gov/index.html)'s age-adjusted ER visits and hospitalizations for asthma per 10,000 population (051823 and 053054 respectively on the EPH system)
 - [HRSA Data Warehouse](data.HRSA.gov)'s active medical doctor and respiratory therapist estimations per state, from AMA Physician Professional Data 2019
 - [County Health Rankings & Roadmaps](https://www.countyhealthrankings.org/reports/2019-county-health-rankings-key-findings-report)' 2019 County Health Rankings dataset's 'Ranked Measure Data' and 'Additional Measure Data' tabs
 - [Agency for Toxic Substances and Disease Registry (ATSDR)](https://www.atsdr.cdc.gov/place-health/index.html)'s CDC Social Vulnerability Index for 2018
 - [United States Census Bureau](https://www.census.gov/geographies/reference-files/time-series/geo/gazetteer-files.html)'s census tracts for latitude and longitude information
Additional datasets were cleaned in the initial steps but left out during the analysis.

## Findings Summary

 -

 - While diesel particulate matter concentration has a non-negligible correlation with asthma-related hospitalization (r≈0.40) and ER visits (r≈0.32), it is not even among the top 20 predictors for either according to a LASSO model.
Considering the documented impact air pollution has on respiratory health, this implies that other factors like demographics act as a comprehensive proxy for the cumulative environmental burden in the multivariate model, given that '% African American' and '% Hispanic' are among top predictors for both ER visits and hospitalizations
 - A lack of vehicle availability (r≈0.31) is the 4th strongest predictor for ER visits but not one of the top 20 predictors for hospitalization, signifying that the impact of a lack of car is not explainable by its association with poverty alone, and that a lack of transportation is a driver of ER usage independent of financial situation.
 - The number of single-parent households and uninsured population were collinear (r≈0.95), and the model selected the '# Single-Parent Households' feature as the number 1 predictor for both ER visits and hospitalizations. Given that our target is a rate (per 10,000) but the prediction is driven by absolute counts of single-parent households while essentially ignoring uninsured populations, we can consider population density as a very strong driver as single parents are typically much more dense in urban areas while uninsured people are more likely to be in rural areas.
 - Both ER visits and hospitalizations for asthma have high coefficients for HIV, diabetes, preventable hospital stays and injury deaths. This supports the hypothesis that the same structural failures impacting both ambulatory and emergency treatment of asthma are also impacting other medical issues, and these syndemic conditions should not be considered in isolation of one another.

## Methodology

### Data Cleaning and Merging

The different datasets used in this project follow different formatting schemes, so the first step was to clean the data and standardize the format. All the datasets' names were initially manually parsed and renamed for clarity, then loaded into a dictionary of Polars dataframes in Python for the processing. All the state and county name columns were normalized into 'State' and 'County' respectively, with the capitalization set to title case, and any trailing phrases like ' County' were removed from county names for unification purposes. To avoid issues with duplicate county names like 'Orange County', another column was created to host the combined 'County (State)' name. Several of these datasets utilized code phrases as the headers for the columns, and these headers had to be re-assigned using the tables included either along the datasets themselves or the dictionary tables stored in the supplementary documentation by extracting them via pdfplumber. The processed dataframes were then saved into a separate folder as separate files, with the dataframe storing asthma-related hospitalization and ER visits combined into a target dataframe.

### Data Pre-processing

Before the statistical analysis, missing entries needed to be handled. As this project included datasets from numerous sources, the intersection of counties with no missing features was very limited. All the cleaned dataframes from the previous step were loaded and joined onto the dataframe containing asthma-related data, dropping counties that don't have any information for either of the asthma-related metrics. The end result was 1614 counties out of 3144 that had measurements for either ER visits or hospitalizations, 1512 counties with ER visit rate data, 1096 counties with hospitalization data, and 1037 counties with data for both at once. For these 1614 counties, 368 numeric features excluding the asthma-related mesaurements existed. From these 368 entries, the features where at least 30% of the counties were missing data were dropped, leaving 324 features. The remaining features were filled using spatial imputation where feasible, and MICE where it wasn't (like counties where their neighbors were also missing these features). 

In the next step, the existing features were separated into groups based on the presence of certain subwords within their headers, with the groups higher on the list taking precedence in the case a column includes subwords from multiple groups:

 - 'Asthma':         ['asthma']
 - 'Pollution':      ['pollutant', 'pollution']
 - 'Care Provider':  ['respiratory therapist', 'm.d.', 'physician', 'hospital']
 - 'Insurance':      ['insur', 'coverage', 'medicaid', 'private', 'medicare']
 - 'Other Health':   ['physical', 'smok', 'disability', 'disabl', 'medical', 'mental', 
                       'activity', 'hiv', 'diabet', 'food', 'nutrition', 'injury', 
                       'mortality', 'obese', 'life expectancy', 'birthweight']

 - 'Socioeconomic':  ['income', 'poverty', 'unemployment', 'education', 'socioeconomic', 
                       'minority', 'demographic', 'crime', 'college', 'education', 'social', 
                       'alone', 'single']
 - 'Housing':        ['rent', 'value', 'owner', 'vacant', 'crowd', 'hous']

### Statistical Analysis

The 'Asthma' category's features only included the hospitalization and ER visit rates, and their correlation coefficients to the entries in the other categories were then calculated. Using the scikit-learn and statsmodel Python packages, the LASSO model was used to select statistically relevant features from the other categories, leaving us with 82 features out of 324 for ER visit rates and 117 features for hospitalization rates. For these features, the variance inflation factors and p-values were calculated to spot significant relationships. The top 20 results by p-value are as follows:

### Age-adjusted ER Visit Rate for Asthma per 10,000 People (Adjusted R^2: 0.6300)
| Feature                                                                                                                                       |   LASSO_Coef |   OLS_Coef |     P_Value |   Conf_Int_Low |   Conf_Int_High |     VIF |   Correlation | Interaction             |
|:----------------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|---------------:|----------------:|--------:|--------------:|:------------------------|
| Demographics => % African American                                                                                                            |     5.37951  |   6.44116  | 8.39512e-14 |       4.76411  |        8.11821  | 7.90719 |    0.534484   | Asthma vs Socioeconomic |
| (Household Composition/Disability) Percentage of persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [MP_AGE17]          |    -2.40197  |  -3.23798  | 2.81258e-10 |      -4.23802  |       -2.23794  | 2.81167 |   -0.224994   | Asthma vs Other Health  |
| State_Minnesota                                                                                                                               |    -2.38514  |  -2.59954  | 9.36321e-10 |      -3.4275   |       -1.77159  | 1.92728 |   -0.186924   | Unspecified             |
| Preventable hospital stays => Preventable Hosp. Rate                                                                                          |     1.5686   |   2.19382  | 7.71017e-07 |       1.32679  |        3.06085  | 2.11347 |    0.149164   | Asthma vs Care Provider |
| COVERAGE ALONE OR IN COMBINATION => Direct-purchase insurance alone or in combination => Under 19                                             |     1.06032  |   2.09459  | 1.02881e-06 |       1.25713  |        2.93206  | 1.97179 |   -0.147826   | Asthma vs Insurance     |
| State_Massachusetts                                                                                                                           |     1.35441  |   1.88488  | 1.50247e-06 |       1.1194   |        2.65035  | 1.64737 |    0.132551   | Unspecified             |
| Population, All (County Level File)                                                                                                           |     1.50867  |   2.56532  | 4.44459e-06 |       1.47286  |        3.65777  | 3.35537 |    0.253062   | Unspecified             |
| HIV prevalence => HIV Prevalence Rate                                                                                                         |     3.15878  |   2.62822  | 8.0111e-06  |       1.47754  |        3.7789   | 3.72254 |    0.545019   | Asthma vs Other Health  |
| State_Maine                                                                                                                                   |     1.02737  |   1.50377  | 1.1364e-05  |       0.834004 |        2.17353  | 1.26118 |    0.0342723  | Unspecified             |
| State_Louisiana                                                                                                                               |    -1.65015  |  -2.36312  | 1.78933e-05 |      -3.44027  |       -1.28598  | 3.26196 |    0.0968816  | Unspecified             |
| Median household income => Household Income                                                                                                   |    -1.8992   |  -3.23827  | 4.97232e-05 |      -4.7995   |       -1.67704  | 6.85275 |   -0.12596    | Asthma vs Socioeconomic |
| Income inequality => Income Ratio                                                                                                             |    -1.55179  |  -2.16625  | 7.52795e-05 |      -3.23663  |       -1.09588  | 3.22109 |    0.323614   | Asthma vs Socioeconomic |
| (General) Adjunct variable - Percentage uninsured in the total civilian noninstitutiona lized population estimate, 2014-2018 ACS [EP_UNINSUR] |    -1.69926  |  -2.59148  | 9.36147e-05 |      -3.88923  |       -1.29374  | 4.73487 |    0.0881921  | Asthma vs Insurance     |
| (Socioeconomic) Flag - the percentage of persons with no high school diploma is in the 90th percentile (1 = yes, 0 = no) [F_NOHSDP]           |    -1.29815  |  -1.5839   | 0.000122887 |      -2.39083  |       -0.77697  | 1.83064 |    0.028619   | Asthma vs Socioeconomic |
| (Minority Status/Language) Percentile ranking for Minority Status/Languag e theme [RPL_THEME3]                                                |     1.98269  |   2.59473  | 0.00022248  |       1.21948  |        3.96997  | 5.31728 |    0.385522   | Asthma vs Socioeconomic |
| Sexually transmitted infections => Chlamydia Rate                                                                                             |     1.07411  |   1.91877  | 0.00119835  |       0.759057 |        3.07849  | 3.78123 |    0.500246   | Unspecified             |
| (Household Composition/Disability) Persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [M_AGE17]                         |     0.591494 |   1.13042  | 0.00168132  |       0.425832 |        1.835    | 1.39571 |    0.0253247  | Asthma vs Other Health  |
| Value_Pollutant: Formaldehyde                                                                                                                 |    -1.18139  |  -2.23385  | 0.00403838  |      -3.75554  |       -0.712156 | 6.51006 |    0.316333   | Asthma vs Pollution     |
| Alcohol-impaired driving deaths => % Alcohol-Impaired                                                                                         |     0.567712 |   0.936429 | 0.00410802  |       0.297331 |        1.57553  | 1.14833 |    0.00686554 | Unspecified             |
| Life expectancy => Life Expectancy                                                                                                            |    -1.22666  |  -1.98479  | 0.00460723  |      -3.35686  |       -0.612724 | 5.29274 |   -0.178869   | Asthma vs Other Health  |

### Age-adjusted Hospitalization Rate for Asthma per 10,000 People (Adjusted R^2: 0.6058)
| Feature                                                                                                                                  |   LASSO_Coef |   OLS_Coef |     P_Value |   Conf_Int_Low |   Conf_Int_High |      VIF |   Correlation | Interaction             |
|:-----------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|---------------:|----------------:|---------:|--------------:|:------------------------|
| Preventable hospital stays => Preventable Hosp. Rate                                                                                     |    0.740175  |   0.809054 | 3.64353e-36 |      0.68601   |        0.932097 |  2.23933 |     0.261959  | Asthma vs Care Provider |
| (Household Composition/Disability) Single parent household with children under 18 estimate, 2014-2018 ACS [E_SNGPNT]                     |    0.696973  |   1.66483  | 7.53968e-11 |      1.16679   |        2.16286  | 36.6876  |     0.31468   | Asthma vs Other Health  |
| State_Minnesota                                                                                                                          |   -0.330135  |  -0.518895 | 2.96734e-08 |     -0.701536  |       -0.336255 |  4.93393 |    -0.1817    | Unspecified             |
| High school graduation => Graduation Rate                                                                                                |   -0.254285  |  -0.294496 | 1.79109e-06 |     -0.414976  |       -0.174017 |  2.14696 |    -0.237971  | Unspecified             |
| PRIVATE INSURANCE ALONE OR IN COMBINATION => 65 to 74 years                                                                              |   -0.276394  |  -0.363586 | 4.97696e-06 |     -0.51922   |       -0.207952 |  3.58268 |    -0.204337  | Asthma vs Insurance     |
| (Household Composition/Disability) Percentile percentage of single parent households with children under 18 estimate [EPL_SNGPNT]        |   -0.203345  |  -0.520633 | 5.50986e-06 |     -0.744547  |       -0.296718 |  7.41587 |     0.244809  | Asthma vs Other Health  |
| Food environment index => Food Environment Index                                                                                         |    0.318978  |   0.372078 | 9.89352e-06 |      0.207502  |        0.536655 |  4.00623 |    -0.0476712 | Asthma vs Other Health  |
| Median household income => Household income (Hispanic)                                                                                   |    0.203538  |   0.231079 | 1.68087e-05 |      0.126097  |        0.336061 |  1.63015 |     0.0239192 | Asthma vs Socioeconomic |
| Low birthweight => % LBW                                                                                                                 |    0.2679    |   0.354808 | 1.93021e-05 |      0.192448  |        0.517168 |  3.89902 |     0.245399  | Asthma vs Other Health  |
| State_Missouri                                                                                                                           |    0.256954  |   0.274404 | 1.99995e-05 |      0.148603  |        0.400206 |  2.34084 |     0.050911  | Unspecified             |
| Primary care physicians => PCP Rate                                                                                                      |    0.162484  |   0.244587 | 6.14551e-05 |      0.1252    |        0.363974 |  2.1082  |     0.0882842 | Asthma vs Care Provider |
| State_Colorado                                                                                                                           |   -0.112462  |  -0.337779 | 6.15793e-05 |     -0.502674  |       -0.172884 |  4.02175 |    -0.0490981 | Unspecified             |
| Driving alone to work => % Drive Alone                                                                                                   |   -0.29709   |  -0.321779 | 0.00013555  |     -0.486747  |       -0.156811 |  4.0253  |    -0.0842191 | Asthma vs Socioeconomic |
| HIV prevalence => HIV Prevalence Rate                                                                                                    |    0.410455  |   0.346523 | 0.00016299  |      0.166703  |        0.526343 |  4.78273 |     0.445032  | Asthma vs Other Health  |
| Median age (years)                                                                                                                       |   -0.136095  |  -0.457689 | 0.000518727 |     -0.715785  |       -0.199594 |  9.8528  |    -0.163446  | Unspecified             |
| PUBLIC INSURANCE ALONE OR IN COMBINATION => 45 to 54 years                                                                               |    0.249918  |   0.340107 | 0.000763405 |      0.142301  |        0.537912 |  5.7873  |     0.244967  | Asthma vs Insurance     |
| (Socioeconomic) Percentage of persons with no high school diploma (25+) estimate MOE 2018 DESCRIPTION [MP_NOHSDP]                        |   -0.152249  |  -0.426911 | 0.00112256  |     -0.683454  |       -0.170368 |  9.73463 |    -0.156919  | Asthma vs Socioeconomic |
| Alcohol-impaired driving deaths => % Alcohol-Impaired                                                                                    |    0.109138  |   0.150552 | 0.00123293  |      0.0593303 |        0.241773 |  1.23081 |    -0.0366294 | Unspecified             |
| (Household Composition/Disability) Percentage of single parent households with children under 18 estimate MOE, 2014-2018 ACS [MP_SNGPNT] |    0.0895873 |   0.351739 | 0.00162149  |      0.133246  |        0.570232 |  7.06112 |    -0.214481  | Asthma vs Other Health  |
| State_New Jersey                                                                                                                         |    0.156858  |   0.17126  | 0.00218335  |      0.061814  |        0.280705 |  1.77172 |     0.147813  | Unspecified             |

The full tables for these scores and the correlation values is available in the files.

### Conclusions
 - For every unit increase in the percentage of African American population in a county, the age-adjusted ER vist rate rises considerably (LASSO coefficient of 5.37, OLS coefficient of 6.44, p-value of 8.39x10^(-14)). While the variane inflation factor being 7.91 indicates that this particular measure has high collinearity with a multitude of features (most of which are most likely to do with socioeconomic factors), the extremely low p-value implies the existence of strong systemic disparities that African American communities face that expose them to increased risk, such as environmental racism.
 - The percentage of households with no vehicle available has a LASSO coefficient of 2.03, p-value of 0.01, correlation of 0.45 and VIF of 7.93 for ER visits. A lack of car might be causing families to miss the routine management visits that serve as the ambulatory care for asthma, causing severe attacks that demand ER vists later on. Alternatively, families that lack cars might live in urban areas where hospitals are closer by, making regular controls easier to manage. Finally, it could be that a lack of vehicles is a signifier of low income, which would also deter families from routine checkups and push them to rely more on ER services.
 - Life expectancy acts as a proxy for the overall health and wealth of a community, with a LASSO coefficient of -1.23, p-value of 0.005, correlation of -0.18 and VIF of 5.29.
 - Areas with higher single-parent household counts also see higher hospitalization rates, with a LASSO score of 0.70, p-value of 7.54x10^(-11) and correlation of 0.31, though a VIF score of 36.69 implies this statistic is highly connected to other features, most likely socioeconomic factors. Single parents are much more common in urban areas as opposed to rural ones, and the financial stress of having to raise a child with a single income, the relative lack of care children face, and the environmental factors like exposure to higher pollution and mold stemming from being confined to cheaper housing can all contribute to the severity of asthma, particularly in younger children.
