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
