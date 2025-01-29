# NYC Accidents - Exploratory Data Analysis

## Table of Contents
- [Project Overview](#project-overview)
- [Data Sources](#data-sources)
- [Data Preparation](#data-preparation)
- [Exploratory Data Analysis](#exploratory-data-analysis)


### Project Overview
This data analysis project aims to identify trends and patterns of NYC Traffic Collisions from 2014-2024. By analyzing the contributing factors for each crash, the location (borough) of the collisions, and the significant trends over time, this project aims find different solutions to preventing accidents in New York City.

### Data Sources
This data set titled 'Motor Vehicle Collisions-Crashes" taken from [data.gov](data.gov), published by New York City agencies and updated every day. At the time of download, the file had over 2,000,000 rows of collision data starting from July 2012, and ending in December of 2024.

Given the original dataset had 2,000,000+ rows, creating a Entity Relationship Diagram was optimal to shorten the execution fo queries in PostgreSQL. Executing queries using only relevant tables significantly shortened the time needed to create queries, in addition to the use of Common Table Expressions(CTEs)

![image](https://github.com/user-attachments/assets/ba126c91-ff51-4f77-8ee2-ba1da2630e0b)

After cleaning the data set using PostgreSQL, the final dataset was exported as a CSV file and used in Tableau.

### Tools
- Draw.io - Creating Entity Relationship Diagram (ERD)
- SQL - Data Cleaning
- Python - GIS mapping
- Tableau - Data Visualization

### Data Preparation
***Handling Missing Values for Location Based Columns***

Many of the rows for columns 'Borough', 'Zipcode', 'Latitude', and 'Longitude' had missing values. After a quick query in PostgreSQL, I significant amount of rows had both Latitude and Longitude, but still didn't have a designated zipcode or borough. Thus, I decided to use Python to map locations to add missing values. Below is the full code:
``` python
import pandas as pd
import geopandas as gpd
from shapely.geometry import Point

# Reading in geospatial data
shapefile_path = "C:\\Users\\fcern\\OneDrive\\Documents\\tl_2024_us_zcta520\\tl_2024_us_zcta520.shp"

#Ensuring ZCTA data has same CRS as dataset (Coordinate Reference System)
geo_spatial_data = gpd.read_file(shapefile_path)
geo_spatial_data = geo_spatial_data.to_crs("EPSG:4326")     #code best for latitude/longitude coordinates

# Reading in the original dataset
csv_filepath = "C:\\Users\\fcern\\Downloads\\Motor_Vehicle_Collisions_-_Crashes.csv"
NYC_accidents = pd.read_csv(csv_filepath)

# Ensure zipcode column is an integer, and convert to '0' if null
NYC_accidents['cleaned_zipcode'] = pd.to_numeric(NYC_accidents['ZIP CODE'], errors='coerce')
NYC_accidents['cleaned_zipcode'] = NYC_accidents['cleaned_zipcode'].fillna(0).astype(int)

# Create geometry column from LATITUDE and LONGITUDE
NYC_accidents['geometry'] = NYC_accidents.apply(lambda row: Point(row['LONGITUDE'], row['LATITUDE']), axis=1)

# Convert the accidents Dataframe to a GeoDataFrame
accidents_gdf = gpd.GeoDataFrame(NYC_accidents,geometry='geometry',crs='EPSG:4326')

#joining dataframes
result = gpd.sjoin(accidents_gdf,geo_spatial_data,how="left",predicate="within")

#getting zipcodes (ZIP Code Tabulation Area)
result['zipcode_zcta'] = result['ZCTA5CE20']

#keeping only relevant columns
columns_to_keep = ['COLLISION_ID','LONGITUDE','LATITUDE','ZIP CODE','zipcode_zcta','geometry']
result = result[columns_to_keep]

# Export the DataFrame to a CSV file
result.to_csv("C:\\Users\\fcern\\OneDrive\\Documents\\NYCaccidents_zipcodes.csv", index=False)
```

After the code was executed, the csv file was imported in PostgreSQL as table 'zipcode' and joined with table 'crashes', with the following code:
``` sql
-- Join crashes and zipcode table

-- Using COALESCE to either assign zipcode, or denote as 'Unknown'
WITH cte_zipcodes AS (
SELECT c.collisionID, COALESCE(z.zipcode_zcta,c.zipcode,'Unknown') AS zipcode
FROM crashes AS c
LEFT JOIN zipcode AS z
USING(collisionID)),

-- CTE that ensures boroughs are correctly matched
cte_boroughs AS (
SELECT CASE WHEN zipcode ~ '^[0-9]+$' THEN
	   		CASE
				WHEN zipcode::INTEGER BETWEEN 10001 AND 10282 THEN 'Manhattan'
				WHEN zipcode::INTEGER BETWEEN 10451 AND 10475 THEN 'Bronx'
				WHEN zipcode::INTEGER BETWEEN 11201 AND 11256 THEN 'Brooklyn'
				WHEN zipcode::INTEGER BETWEEN 11004 AND 11109 OR
				     zipcode::INTEGER BETWEEN 11351 AND 11697 THEN 'Queens'
			    WHEN zipcode::INTEGER BETWEEN 10301 AND 10314 THEN 'Staten Island'
			    ELSE 'Other'
			END
		ELSE 'Unknown' END AS borough,
		CollisionID
		FROM cte_zipcodes
)

-- Update values from crashes table from the created CTEs
UPDATE crashes
SET zipcode = cte1.zipcode,
	borough = cte2.borough
FROM cte_zipcodes AS cte1, cte_boroughs AS cte2
WHERE crashes.collisionID = cte1.collisionID AND
      crashes.collisionID = cte2.collisionID;
```
After this process, location based columns were fully cleaned and ready for interpretation. To ensure the code above worked properly, original zipcodes were tested with the staging tables' and were found to be accurate.

***Categorical Simplification***

From a quick glance at the data, it was clear there were too many distinct values for both the collision's vehicle types and contributing factors. The following code was executed to quickly analyze common values for these columns:
``` sql
-- Query finding most common Vehicle types and factors for the first vehicle
SELECT Vehicle1Type, COUNT(*) AS obs
FROM crashstagingtable
GROUP BY Vehicle1Type
HAVING COUNT(*) >= 10
ORDER BY obs DESC
;

SELECT Vehicle1Factor, COUNT(*) AS obs
FROM crashstagingtable
GROUP BY Vehicle1Factor
HAVING COUNT(*) >= 10
ORDER BY obs DESC
;
```
Query 1

![image](https://github.com/user-attachments/assets/0c91ba76-55ee-4a37-9050-c91cef4b8b0d)

Query 2

![image](https://github.com/user-attachments/assets/9a0699bb-ea6f-4a94-bf5d-158189ce3ab9)

From the queries, you can see that there were over 200 DISTINCT vehicle types for vehicle 1 with >= observations and over 60 DISTINCT contributing factors with >= 10 observations. Considering this is similar for columns regarding vehicles 2-5, it was clear to see vehicle types and contributing factors must be broadly categorized to get more meaningful insights for the data analysis.

Below is a snippet of was done to each vehicle column to create a small group of values for each column.
**NOTE:** Regarding the vehicle contributing factors, the query above showed a majority of the collisions have the value 'unspecified' for vehicle1type. For vehicles 2-5, the value 'unspecified' progressively becomes less common. I made the assumption 'unspecified' indicates that a vehicle was in fact involved but not categorized. This assumption is further supported by the fact that for vehicles 2-5, NULL values for vehicle type and contributing factor progressively become more common.

The code snippet below shows what was done for each vehicle column for both vehicle type and vehicle contributing factor:
``` sql
-- Grouping each vehicle's contributing factor
WITH cte_factors AS (
SELECT
	CASE
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(inattention|distraction|fatigue|drowsy|asleep|electronic|cell phone|device|reaction|eating|text|headphones)%' THEN 'Distraction/Inattention'
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(yield|other|traffic|keep right)%' THEN 'Traffic Violation'
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(unsafe|following|tail|pass|improperly|driver|reaction|road rage|inexperience)%' THEN 'Human Error'
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(pavement|pedestrian|oversize|obstru|glare|animal|lane marking|shoulder)%' THEN 'Environmental'
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(defect|steering failure|inadequate)%' THEN 'Defective Vehicle'
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(lost conscious|disability|illnes|runaway)%' THEN 'Emergency'
		WHEN lower(Vehicle1Factor) SIMILAR TO '%(prescription|medication|drug|alcohol)%' THEN 'Alchohol/Drugs'
		WHEN Vehicle1Factor IS NULL THEN 'No data'
		ELSE 'Unspecified/Other'
	END AS Vehicle1Factor,
	CASE
		WHEN lower(Vehicle2Factor) SIMILAR TO
--
--
--
```
``` sql
-- Grouping contributing factors for each vehicle
WITH cte_vehicletype AS (
SELECT CASE
	   		WHEN lower(Vehicle1Type) SIMILAR TO '%(sedan|util|passenger|taxi|carry|
				                                limo|pick|pk|bus|convertible|sport|
												 van|sub|cab|scho|wagon|suv)%' THEN 'Passenger'
			WHEN lower(Vehicle1Type) SIMILAR TO '%(bike|bicycle|scoot|moped|e-bik|e-sco)%' THEN 'Two-Wheeler'
			WHEN lower(Vehicle1Type) SIMILAR TO '%(com|dump|garb|lunch|box|tractor|livery|motorcycle|
				                                tank|flat|concrete|armored|bev|stake|rack|multi|trac|
												 trail|open|bulk|pallet|usps)%' THEN 'Commercial'
			WHEN lower(Vehicle1Type) SIMILAR TO '%(ambu|fire|fypd|nypd|pol)%' THEN 'Emergency'
			WHEN lower(Vehicle1Type) SIMILAR TO '%(tow|snow|sweep|fork|sani)%' THEN 'Special'
			WHEN Vehicle1Type IS NULL THEN 'No data'
			ELSE 'Unknown'
	   END AS Vehicle1Type,
	   CASE
			WHEN lower(Vehicle2Type) SIMILAR TO
--
--
--
```

After this was done, the vehicles table was updated with the following code:
``` sql
UPDATE Vehicles
SET vehicle1type = cte.vehicle1type,
    vehicle2type = cte.vehicle2type,
	vehicle3type = cte.vehicle3type,
	vehicle4type = cte.vehicle4type,
	vehicle5type = cte.vehicle5type
FROM cte_vehicletype AS cte
WHERE vehicles.collisionID = cte.collisionID;

UPDATE factors
SET vehicle1factor = cte.vehicle1factor,
	vehicle2factor = cte.vehicle2factor,
	vehicle3factor = cte.vehicle3factor,
	vehicle4factor = cte.vehicle4factor,
	vehicle5factor = cte.vehicle5factor
FROM cte_factors AS cte
WHERE factors.collisionID = cte.collisionID;
```
Thankfully, columns within the 'Casualties' table were straightforward and denoted injuries and fatalities for each collision. The rest of the columns in the tables 'Crashes', 'Factors', and 'Vehicles' were now cleaned and ready for data analysis and visualization in Tableau. Values from all these tables were added to a single csv file titled 'transformed_nyc_accidents' and imported into Tableau


### Exploratory Data Analysis
***Have there been any major changes in the number of crashes over time? Is there a trend in the number of crashes over any given year?***

![image](https://github.com/user-attachments/assets/11c1913f-a4b5-46d3-a862-f674c2d5bd67)

From the created Tableau dashboard seen above, the 'Collisions Over Time' Section shows that collisions in New York City show a similar trend throughout each year between 2012-2024. That is, February typically has the lowest amount of collisions, and there is a slow buildup in the number of collisions up until May-June, until it tapers off for the rest of the year.

It is also important to note that from 2012-2024, New York City reached an all-time low in the number of collisions in April 2020 with only 4,129 crashes. This is likely due to the rise of COVID-19 cases and the impending lockdown. Since then, the number of collisions month-to-month are significantly down and not similar to that between the years 2012-2019.

***What are the most common crashes in New York City?***

![image](https://github.com/user-attachments/assets/457eeb79-3d03-4c1c-8c15-24cedf9dcd7a)

Disregarding 'Unspecified/Other' and 'No data' values for vehicle type and contributing factor, we see for vehicles involving 2+ vehicles the most common reasons for a collision are as a result of Distractions and Traffic Violations. In addition, Human Error is a common factor, which includes: unsafe maneuvers, road-rage, tailgating, etc.. 

The heatmap also shows rare occurances for collisions between 2+ vehicles. It's rare for collisions to be caused by defective vehicles, and interestingly there are thousands of collisions where the environment (slippery roads, improper lane markings, etc) was a contributing factor

***Which part of New York City has the most crashes***
Due to the sheer amount of collisions that the table contained, it was increasingly difficult to display the geographic columns into Tableau without overcrowding the worksheet. Therefore, I created a simple barplot to get an idea of which boroughs had the most crashes, as seen below.

![image](https://github.com/user-attachments/assets/284ff53d-fc5b-42f3-8e93-507da27f3e80)

From this barplot, it's easily seen Brooklyn and Queens have a significant amount of crashes with over 400,000 collisions respectively. Meanwhile, Manhattan has nearly 300,000 collisions, and the rest of the boroughs have greater than or equal to ~200,000 collisions.

***Which types of crashes cause the most injuries/deaths? Who is the most vulnerable?***

For this question, the interactive Tableau Dashboard was most effective. Utilizing the filters, I made the following insights:
- Collisions involving Alchohol/Drugs have the highest fatality rate

![image](https://github.com/user-attachments/assets/9bd762b8-0068-42a1-b5fd-f8571eab9436)

By far, a collision involving alchohol/drugs had the highest fatality rate when compared with other crash types

- Cyclists are susceptible to crashes where the contributing factor was 'Distraction/Inattention' and 'Traffic Violation'

![image](https://github.com/user-attachments/assets/12d9721b-ce93-4c26-b3f9-2d09582e8750)

Collisions with these contributing factors have a high concentration of injuries for cyclists, indicated by the Distribution Table in Tableau

- Pedestrians injuries/deaths aren't common with the created categories
 
After trying to find insights into pedestrian injuries/deaths, it become readily apparent pedestrians aren't injured in any specific car crash type. Once I changed the filter to 'Unspecified', the distribution of pedestrians injured increased significantly. Thus, I came to the realization pedestrians are injured in car crashes with unspecified contributing factors
