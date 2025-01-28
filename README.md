# NYC Accidents
A personal project covering collisions in New York City from 2014-2024.

## Table of Contents
- [Project Overview](#project-overview)
- [Data Preparation](#data-preparation)
- [Exploratory Data Analysis (EDA)](#exploratory-data-analysis-(eda))
- [Results](#results/findings)
- [Limitations](#limitations)
### Project Overview
This data analysis project aims to identify trends and patterns of NYC Traffic Collisions from 2014-2024. By analyzing the contributing factors for each crash, the location (borough) of the collisions, and the significant trends over time, this project aims find different solutions to preventing accidents in New York City.

### Data Sources
This data set was 

### Tools
- Draw.io - Creating Entity Relationship Diagram (ERD)
- SQL - Data Cleaning
- Python - GIS mapping
- Tableau - Data Visualization

### Data Preparation
The original dataset required cleaning for the following reasons:
1. Handling Missing Values
2. Categorial Simplification to Improve Interpretability

### Exploratory Data Analysis (EDA)
- What are the most common crashes in New York City?
``` postgreSQL
SELECT Vehicle1Factor, Vehicle2Factor, SUM(people_injured) AS total_injured, SUM(people_killed) AS total_killed
FROM transformed_nycaccidents
WHERE vehicle1factor <> 'Unspecified/Other' AND vehicle1factor <> 'Unspecified/Other'
  AND vehicle1factor <> 'No data' AND vehicle1factor <> 'No data'
  AND vehicle2factor <> 'Unspecified/Other' AND vehicle1factor <> 'Unspecified/Other'
  AND vehicle2factor <> 'No data' AND vehicle1factor <> 'No data'
GROUP BY Vehicle1Factor, Vehicle2Factor
ORDER BY SUM(people_injured) DESC, SUM(people_killed) DESC;
```
- Have there been any major changes in the number of crashes over time?
- Which types of crashes cause the most injuries/deaths? Who is the most vulnerable?
- Which part of New York City has the most crashes?


### Results/Findings
1. The most common insight was that collisions reached an all-time low during the COVID-19 Lockdowns (~March 2020).

### Limitations
Regarding the Tableau dashboard, it is important to remember the HeatMap is for collisions for 2+ Vehicles. It would have been overwhelming to include a heatmap and/or filters for 5 different vehicle contributing factors, even more so when we consider a majority of collisions don't involve more than 3 vehicles. 
While many observations without their designated zipcode and/or borough were assigned so through the use of Python's GeoPandas library, there is still an extensive amount of rows with missing values.
