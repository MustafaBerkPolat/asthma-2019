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

The 'Asthma' category's features only included the hospitalization and ER visit rates, and their correlation coefficients to the entries in the other categories were then calculated. Using the scikit-learn and statsmodel Python packages, the LASSO model was used to select statistically relevant features from the other categories, leaving us with 82 features out of 324 for ER visit rates and 117 features for hospitalization rates. For these features, the variance inflation factors and p-values were calculated to spot significant relationships. The results that have a p-value less than 0.05 and a variance inflation factor less than 5 in the order of highest absolute OLS coefficient are as follows, with the full tables available under the project files.

### Age-adjusted ER Visit Rate for Asthma per 10,000 People (Adjusted R^2: 0.6300)
| Feature                                                                                                                                                    |   LASSO_Coef |   OLS_Coef |     P_Value |     VIF |   Correlation |
|:-----------------------------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|------------:|--------:|--------------:|
| Preventable hospital stays => Preventable Hosp. Rate                                                                                                       |     1.87994  |   1.9792   | 8.49257e-20 | 2.22355 |    0.249675   |
| PRIVATE INSURANCE ALONE OR IN COMBINATION => 75 years and over                                                                                             |    -1.69729  |  -1.96155  | 7.33662e-10 | 4.81136 |   -0.232277   |
| Sexually transmitted infections => Chlamydia Rate                                                                                                          |     1.71715  |   1.62456  | 1.27759e-07 | 4.49399 |    0.477599   |
| Violent crime => Violent Crime Rate                                                                                                                        |     1.76492  |   1.57849  | 5.39001e-11 | 2.744   |    0.556272   |
| (Socioeconomic) Flag - the percentage of persons with no high school diploma is in the 90th percentile (1 = yes, 0 = no) [F_NOHSDP]                        |    -1.49166  |  -1.47707  | 4.14607e-11 | 2.37418 |    0.0919456  |
| Access to exercise opportunities => % With Access                                                                                                          |     1.28918  |   1.34684  | 1.35739e-08 | 2.67047 |    0.107066   |
| (Minority Status/Language) Minority (all persons except white, non- Hispanic) estimate MOE, 2014-2018 ACS [M_MINRTY]                                       |     0.966357 |   1.24388  | 9.53123e-05 | 4.83739 |    0.174528   |
| COVERAGE ALONE => Public insurance alone => VA care coverage alone                                                                                         |     1.05202  |   1.22624  | 3.12844e-08 | 2.33145 |    0.0912883  |
| Driving alone to work => % Drive Alone                                                                                                                     |    -0.911848 |  -1.08811  | 0.000650798 | 4.85171 |    0.106046   |
| COVERAGE ALONE OR IN COMBINATION => Tricare/military  insurance alone or in combination => 65 years and over                                               |     0.898198 |   1.03194  | 0.000145506 | 3.51393 |    0.154172   |
| Alcohol-impaired driving deaths => % Alcohol-Impaired                                                                                                      |     0.96069  |   0.986121 | 3.53244e-08 | 1.51952 |   -0.0398717  |
| State_Louisiana                                                                                                                                            |    -0.879902 |  -0.964721 | 2.12959e-05 | 2.45089 |    0.0626951  |
| State_Pennsylvania                                                                                                                                         |    -0.722121 |  -0.920103 | 0.000118374 | 2.72005 |   -0.0153704  |
| (Household Composition/Disability) Persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [M_AGE17]                                      |     0.889966 |   0.907527 | 1.18282e-05 | 2.04182 |   -0.00891377 |
| (Housing Type/Transportation) Flag - the percentage of persons in institutionalize d group quarters is in the 90th percentile (1 = yes, 0 = no) [F_GROUPQ] |    -0.811804 |  -0.905184 | 5.90741e-05 | 2.41765 |    0.0280724  |
| (Household Composition/Disability) Flag - the percentage of single parent households is in the 90th percentile (1 = yes, 0 = no) [F_SNGPNT]                |    -0.749364 |  -0.86093  | 3.02798e-05 | 2.02723 |    0.315055   |
| Mental health providers => MHP Rate                                                                                                                        |     0.796397 |   0.860568 | 0.000336045 | 2.74312 |    0.160688   |
| State_Massachusetts                                                                                                                                        |     0.798787 |   0.847273 | 0.000613352 | 2.91402 |    0.0948835  |
| Value_Pollutant: 1,3-butadiene                                                                                                                             |     0.655424 |   0.830892 | 0.0021124   | 3.48167 |    0.28306    |
| Dentists => Dentist Rate                                                                                                                                   |     0.717809 |   0.771332 | 0.000533652 | 2.36276 |    0.0923008  |
| COVERAGE ALONE OR IN COMBINATION => VA care coverage alone or in combination => 19 to 64 years                                                             |    -0.480732 |  -0.758709 | 0.0104607   | 4.18696 |    0.0565713  |
| Children in poverty => % Children in Poverty (Hispanic)                                                                                                    |    -0.699719 |  -0.708353 | 0.000138725 | 1.64543 |    0.0959516  |
| Mammography screening => % Screened                                                                                                                        |    -0.531215 |  -0.633916 | 0.0107746   | 2.94655 |   -0.0832824  |
| State_Indiana                                                                                                                                              |    -0.275674 |  -0.629318 | 0.010878    | 2.91157 |   -0.0514016  |
| Median household income => Household income (Hispanic)                                                                                                     |     0.585516 |   0.579817 | 0.00446058  | 1.98168 |   -0.0998618  |
| COVERAGE ALONE OR IN COMBINATION => Tricare/military  insurance alone or in combination => Under 19                                                        |    -0.477471 |  -0.559946 | 0.0288132   | 3.12903 |    0.0387631  |
| (Household Composition/Disability) Persons aged 65 and older estimate MOE, 2014-2018 ACS [M_AGE65]                                                         |    -0.529398 |  -0.542558 | 0.00903095  | 2.05874 |    0.0459065  |
| COVERAGE ALONE OR IN COMBINATION => Medicare coverage alone or in combination => Under 19                                                                  |     0.526832 |   0.495417 | 0.00190453  | 1.21331 |    0.108122   |
| COVERAGE ALONE OR IN COMBINATION => VA care coverage alone or in combination => Under 19                                                                   |    -0.410789 |  -0.443356 | 0.00594518  | 1.23828 |    0.0640911  |
| (Socioeconomic) Percentage of persons below poverty estimate [EP_POV]                                                                                      |     0.228453 |   0.338012 | 0.0390092   | 1.27901 |    0.113229   |
| Value_Pollutant: Ethylene oxide                                                                                                                            |    -0.274024 |  -0.295554 | 0.0465764   | 1.05196 |    0.00454559 |


### Age-adjusted Hospitalization Rate for Asthma per 10,000 People (Adjusted R^2: 0.6058)
| Feature                                                                                                                             |   LASSO_Coef |   OLS_Coef |      P_Value |     VIF |   Correlation |
|:------------------------------------------------------------------------------------------------------------------------------------|-------------:|-----------:|-------------:|--------:|--------------:|
| Preventable hospital stays => Preventable Hosp. Rate                                                                                |    0.72587   |  0.743038  | 2.77885e-117 | 2.30075 |    0.342679   |
| High school graduation => Graduation Rate                                                                                           |   -0.237376  | -0.242681  | 2.78427e-16  | 2.1026  |   -0.19966    |
| Children in poverty => % Children in Poverty (Hispanic)                                                                             |   -0.237203  | -0.23544   | 8.58103e-19  | 1.68757 |   -0.0215559  |
| Median household income => Household income (Hispanic)                                                                              |    0.22887   |  0.230567  | 1.3493e-15   | 1.99142 |    0.00676522 |
| Limited access to healthy foods => % Limited Access                                                                                 |   -0.211594  | -0.211156  | 6.37452e-12  | 2.26487 |   -0.200312   |
| Primary care physicians => PCP Rate                                                                                                 |    0.213646  |  0.210171  | 2.41903e-09  | 2.98117 |    0.136764   |
| Violent crime => Violent Crime Rate                                                                                                 |    0.193099  |  0.197987  | 4.63191e-08  | 3.15686 |    0.407148   |
| (Socioeconomic) Flag - the percentage of persons with no high school diploma is in the 90th percentile (1 = yes, 0 = no) [F_NOHSDP] |    0.173304  |  0.183257  | 1.56914e-06  | 3.50656 |    0.171145   |
| Children in poverty => % Children in Poverty (White)                                                                                |    0.136546  |  0.157353  | 0.000116386  | 4.02025 |    0.159731   |
| COVERAGE ALONE OR IN COMBINATION => Medicare coverage alone or in combination => Under 19                                           |    0.162707  |  0.156573  | 1.85634e-10  | 1.44924 |    0.136931   |
| State_Minnesota                                                                                                                     |   -0.138095  | -0.145058  | 4.09075e-05  | 3.01389 |   -0.146096   |
| COVERAGE ALONE OR IN COMBINATION => Tricare/military  insurance alone or in combination => 65 years and over                        |   -0.135146  | -0.136287  | 0.00173269   | 4.56785 |    0.0567122  |
| Alcohol-impaired driving deaths => % Alcohol-Impaired                                                                               |    0.129505  |  0.121332  | 7.2803e-07   | 1.44417 |   -0.0470957  |
| (Housing Type/Transportation) Percentage of persons in group quarters estimate MOE, 2014-2018 ACS [MP_GROUPQ]                       |   -0.127137  | -0.118942  | 0.00246223   | 3.72415 |   -0.181013   |
| (Household Composition/Disability) Persons aged 65 and older estimate MOE, 2014-2018 ACS [M_AGE65]                                  |   -0.112792  | -0.117894  | 0.000601094  | 2.84823 |    0.105153   |
| (Minority Status/Language) Flag - the percentage those with limited English is in the 90th percentile (1 = yes, 0 = no) [F_LIMENG]  |    0.102736  |  0.110817  | 0.00468619   | 3.70714 |    0.159269   |
| Residential segregation - non-white/white => Segregation Index                                                                      |    0.098804  |  0.106782  | 0.000279327  | 2.0828  |    0.234399   |
| State_Massachusetts                                                                                                                 |    0.10183   |  0.102512  | 0.000325227  | 1.96199 |    0.0955795  |
| State_New Jersey                                                                                                                    |    0.0929166 |  0.0974425 | 0.000219192  | 1.67651 |    0.117657   |
| State_Missouri                                                                                                                      |    0.0989211 |  0.0949383 | 0.00426497   | 2.66395 |    0.0338618  |
| (Household Composition/Disability) Persons aged 17 and younger estimate MOE, 2014-2018 ACS 2018 DESCRIPTION [M_AGE17]               |    0.0820301 |  0.0947508 | 0.00150317   | 2.15041 |    0.0231698  |
| Access to exercise opportunities => % With Access                                                                                   |    0.0888632 |  0.0902469 | 0.00701388   | 2.70495 |    0.173351   |
| State_Louisiana                                                                                                                     |   -0.0804274 | -0.075448  | 0.0198757    | 2.53514 |    0.0104082  |
| Flu vaccinations => % Vaccinated (White)                                                                                            |    0.0747427 |  0.0740295 | 0.02297      | 2.55977 |    0.150881   |
| State_Rhode Island                                                                                                                  |    0.0608147 |  0.0614525 | 0.0061775    | 1.21597 |    0.0398706  |
| State_Maryland                                                                                                                      |   -0.0639126 | -0.0585103 | 0.0170037    | 1.45145 |    0.0303318  |
| State_Connecticut                                                                                                                   |    0.0539108 |  0.0576596 | 0.044355     | 1.98641 |    0.00528584 |
| State_South Carolina                                                                                                                |    0.0557342 |  0.0563888 | 0.0338543    | 1.70597 |    0.0890446  |
| COVERAGE ALONE OR IN COMBINATION => VA care coverage alone or in combination => Under 19                                            |   -0.0466549 | -0.0482044 | 0.0345087    | 1.25582 |    0.0234763  |


The full tables for these scores and the correlation values is available in the files.

### Conclusions
 - For every unit increase in the percentage of African American population in a county, the age-adjusted ER vist rate rises considerably (LASSO coefficient of 5.37, OLS coefficient of 6.44, p-value of 8.39x10^(-14)). While the variane inflation factor being 7.91 indicates that this particular measure has high collinearity with a multitude of features (most of which are most likely to do with socioeconomic factors), the extremely low p-value implies the existence of strong systemic disparities that African American communities face that expose them to increased risk, such as environmental racism.
 - The percentage of households with no vehicle available has a LASSO coefficient of 2.03, p-value of 0.01, correlation of 0.45 and VIF of 7.93 for ER visits. A lack of car might be causing families to miss the routine management visits that serve as the ambulatory care for asthma, causing severe attacks that demand ER vists later on. Alternatively, families that lack cars might live in urban areas where hospitals are closer by, making regular controls easier to manage. Finally, it could be that a lack of vehicles is a signifier of low income, which would also deter families from routine checkups and push them to rely more on ER services.
 - Life expectancy acts as a proxy for the overall health and wealth of a community, with a LASSO coefficient of -1.23, p-value of 0.005, correlation of -0.18 and VIF of 5.29.
 - Areas with higher single-parent household counts also see higher hospitalization rates, with a LASSO score of 0.70, p-value of 7.54x10^(-11) and correlation of 0.31, though a VIF score of 36.69 implies this statistic is highly connected to other features, most likely socioeconomic factors. Single parents are much more common in urban areas as opposed to rural ones, and the financial stress of having to raise a child with a single income, the relative lack of care children face, and the environmental factors like exposure to higher pollution and mold stemming from being confined to cheaper housing can all contribute to the severity of asthma, particularly in younger children.
