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

 - S_Taxon
  - Dimension containing information about species found in the park, one row per species, has information useful for further research on other levels of the taxonomy
  - New species can be found often, updates will occur to their conservation status
  - Daily Updates, new records added and returning species updated. 

```
-- SILVER -> Taxonomy Dimension Table 
CREATE OR REPLACE TABLE S_Taxon as
SELECT DISTINCT 
VARIANT_COL:taxon:id::string as Taxon_ID
,VARIANT_COL:taxon:name::string as Taxon_Name
, VARIANT_COL:taxon:preferred_common_name::string as Common_Name
, VARIANT_COL:taxon:iconic_taxon_name::string as Recognisable_Name
, VARIANT_COL:taxon:rank_level::int as Rank_Level
, VARIANT_COL:taxon:rank::string as Rank 
, VARIANT_COL:taxon:threatened::boolean as is_threatened 
, VARIANT_COL:taxon:ancestor_ids[1]::int as Kingdom_ID
, VARIANT_COL:taxon:ancestor_ids[2]::int as Phylum_ID
, VARIANT_COL:taxon:ancestor_ids[3]::int as Class_ID
, VARIANT_COL:taxon:ancestor_ids[4]::int as Order_ID
, VARIANT_COL:taxon:ancestor_ids[5]::int as Family_ID
, VARIANT_COL:taxon:ancestor_ids[6]::int as Genus_ID
, VARIANT_COL:taxon:complete_species_count::int as complete_species_count
, VARIANT_COL:taxon:extinct::boolean as is_extinct
, VARIANT_COL:taxon:conservation_status:status_name::string as conservation_status

FROM TIL_DATA_ENGINEERING.JC_NATURE.B_OBSERVATIONS
WHERE VARIANT_COL:taxon:name::string IS NOT NULL -- remove unidentified observations
ORDER BY VARIANT_COL:taxon:name::string
;
```
## Gold Layer

Gold Layer consisters of queries often used by the National Park Analyst alongside a view and a table. Your goal with this section is to find the best object for each query, should it remain as a query, be in a table, for the table how would you update it, making sure these queries are also as efficient as possible. 

```
-- Gold Layer Count of Species Observed per park per day

CREATE OR REPLACE VIEW G_Species_per_day_per_park_AGG as
SELECT
  p.place_name,
  DATE(o.observation_timestamp) AS observation_date,
  COUNT(DISTINCT o.taxon_id) AS species_count
FROM TIL_DATA_ENGINEERING.JC_NATURE.S_Observations o
JOIN LATERAL FLATTEN(input => o.place_ids) f
JOIN TIL_DATA_ENGINEERING.JC_NATURE.S_PLACES p
  ON f.value::STRING = p.place_id
GROUP BY 1, 2
ORDER BY species_count DESC;

-- Frequent Query, finding observations based on conservation status per park. 

SELECT
  p.place_name,
  t.conservation_status
  , COUNT(*)
FROM TIL_DATA_ENGINEERING.JC_NATURE.S_Observations o
JOIN S_Taxon  t
    ON o.taxon_id = t.taxon_id
JOIN LATERAL FLATTEN(input => o.place_ids) f
JOIN TIL_DATA_ENGINEERING.JC_NATURE.S_PLACES p
  ON f.value::STRING = p.place_id
  WHERE conservation_status IS NOT NULL
  GROUP BY 1,2
  ORDER BY COUNT(*) DESC;

  -- Main users by identification quality
CREATE OR REPLACE TABLE G_top_users AS
SELECT 
    u.login
    , u.name
    , u.orcid
    , o.quality
    , u.is_suspended
    , COUNT(*) as Count
FROM  TIL_DATA_ENGINEERING.JC_NATURE.S_Observations o
JOIN S_USER u
    ON o.user_id = u.user_id
GROUP BY 1, 2, 3, 4, 5
ORDER BY COUNT(*) DESC;


-- Most common species per park

SELECT
  p.place_name,
  t.taxon_name,
  t.common_name,
  COUNT(*) AS observations_count
FROM TIL_DATA_ENGINEERING.JC_NATURE.S_Observations o
JOIN S_Taxon  t
    ON o.taxon_id = t.taxon_id
JOIN LATERAL FLATTEN(input => o.place_ids) f
JOIN TIL_DATA_ENGINEERING.JC_NATURE.S_PLACES p
  ON f.value::STRING = p.place_id
GROUP BY 1, 2, 3
ORDER BY observations_count DESC;
```



