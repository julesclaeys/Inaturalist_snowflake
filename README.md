# Inaturalist_snowflake

This project aims at learning how to build Procedures, Tasks, Streams and other snowflake objects in order to maintain an existing Database following medallion architecture. The data comes from the inaturalist API. In this example, the goal of the data is to have a look at animals and plants present in all the US National Parks. 

At the end of the station new data will be staged to see if the different objects built functionned appropriately. 

## API 

Here is the API used: https://www.inaturalist.org/pages/api+reference 

iNaturalist is a citizen-science platform where people record, identify, and share observations of plants, animals, fungi, and other organisms. Two commands were used from this api: 

- Get /Places
- Get /Observations

Here are examples: 

- https://www.inaturalist.org/places.json?id=3157 (Places)
- https://api.inaturalist.org/v1/observations?place_id=1455&d1=2026-01-17&d2=2026-03-01&per_page=1 (Observations)

## Set up: 

There first command you will need to run is just to copy the Schema currently built 

```
CREATE SCHEMA **YOUR_SCHEMA_NAME**
CLONE JC_Nature;
```

Remember this clones everything, however I will only upload data to the stage in the original schema, make sure when you build objects to transfer data from the stage to the Bronze layer that you use the original stage in JC_Nature as the source of data. 

## Bronze Layer

Composed of two tables: 

- B_Observations
   - Single Variant column containing a row per observation in json
   - Observations records come in often
   - Needs to be updated whenever new data comes in.
- B_Places_Detail
    - Single Variant column containing a row per places
    - New parks are rare and changes to each park can occur
    - Needs to be checked daily for updates


## Silver Layer

Follows the schema in the presentation. 

- S_Observations
  - Fact Table
  - Daily update, records don’t change.
  - Contains all observations, can be linked to other tables via IDs

```
-- Silver -> Observation Facts Table
CREATE OR REPLACE TABLE S_Observations as
SELECT  DISTINCT
VARIANT_COL:id::string as Observation_ID
, VARIANT_COL:user:id::string as user_ID
, VARIANT_COL:source_place_id::string as Location_ID
, VARIANT_COL:taxon:id::string as Taxon_ID
, VARIANT_COL:place_ids::array as Place_ids
, VARIANT_COL:taxon:native::boolean as is_native
, VARIANT_COL:taxon:endemic::boolean as is_endemic
, VARIANT_COL:taxon:introduced::boolean as was_introduced
, VARIANT_COL:description::string as description
, VARIANT_COL:created_at::datetime as published_timestamp
, VARIANT_COL:time_observed_at::datetime as observation_timestamp
, VARIANT_COL:identification_disagreements_count::int as identification_disagreements_count
, VARIANT_COL:observation_photos:url::string as photo_url
, VARIANT_COL:project_ids::array as project_ID
, VARIANT_COL:quality_grade::string as quality
FROM JC_NATURE.B_OBSERVATIONS;
```

- S_Places
  - Dimension containing all the parks
  - New parks might be added, coordinates might change
  - Daily checks to see if needs updating and updated when bronze layer changes

```
-- SILVER -> Places Dimension Table 
CREATE OR REPLACE TABLE S_PLACES as
SELECT 
    VARIANT_COL:place_id::string as Place_ID
  , SPLIT_PART(VARIANT_COL:name::string, ',', 1) as Place_Name
  , SPLIT_PART(VARIANT_COL:name::string, ',', 2) as Country
  , SPLIT_PART(VARIANT_COL:name::string, ',', 3) as State

  ,VARIANT_COL:geometry_geojson:coordinates::varchar as Coordinates
  ,VARIANT_COL:place_type::string as Place_type_ID
  , SPLIT_PART(VARIANT_COL:location::string, ',', 1) as Latitude
  , SPLIT_PART(VARIANT_COL:location::string, ',', 2) as Longitude
FROM
  "JC_NATURE"."B_PLACES_DETAILS" ; 

```

- S_Users
  - Dimension containing information regarding the users posting their observations
  - Users data can change, new users will be added daily, must be easy to drop USER data and drop after 6 months without an observation. 
  - Daily updates, records will change and come in frequently

```
-- SILVER -> User Dimension Table 
CREATE OR REPLACE TABLE S_USER as
SELECT DISTINCT 
VARIANT_COL:user:id::string as user_ID
, VARIANT_COL:user:login::string as login
, VARIANT_COL:user:name::string as name
, VARIANT_COL:user:orcid::string as orcid
, VARIANT_COL:user:created_at::datetime as created_at
, VARIANT_COL:user:annotated_observations_count::int as observations_count
, VARIANT_COL:user:species_count::int as species_count
, VARIANT_COL:user:activity_count::int as activity_count
, VARIANT_COL:user:suspended::boolean as is_suspended --careul you cannot name a field suspended
FROM TIL_DATA_ENGINEERING.JC_NATURE.B_OBSERVATIONS
ORDER BY VARIANT_COL:user:login::string;
```

- S_Location
  - Dimension containing information where the observation was made, this will often be exact location within the park 
  - Location data won't change, needs to be updated daily,  
  - Daily Updates, table might get huge as often one location per observation. 

```
-- SILVER -> Location Dimension Table 
CREATE OR REPLACE TABLE S_Location as

SELECT DISTINCT 
VARIANT_COL:source_place_id::string as Location_ID
, VARIANT_COL:place_guess::string as Location_Guess
, VARIANT_COL:place_ids::array as Place_ids
, MIN(VARIANT_COL:geojson:coordinates[1]::double) as Latitude
, MIN(VARIANT_COL:geojson:coordinates[0]::double) as Longitude
FROM TIL_DATA_ENGINEERING.JC_NATURE.B_OBSERVATIONS
GROUP BY 1,2,3
ORDER BY VARIANT_COL:place_guess::string 
;
```

 
