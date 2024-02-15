# Google Data Analytics Capstone Project
## Introduction

### About the Company
Cyclistic is a bike-share program that features more than 5,800 bicycles and 600 docking stations. Cyclistic sets itself apart by also o ering reclining bikes, hand tricycles, and cargo bikes, making bike-share more inclusive to people with disabilities and riders who can’t use a standard two-wheeled bike. The majority of riders opt for traditional bikes; about 8% of riders use the assistive options. Cyclistic users are more likely to ride for leisure, but about 30% use the bikes to commute to work each day.

### Scenario
Lily Moreno, the director of marketing at Cyclistic believes the company’s future success depends on maximizing the number of annual memberships. Therefore, the team wants to understand how casual riders and annual members use Cyclistic bikes di erently. From these insights, a new marketing strategy will be designed to convert casual riders into annual members. But fi rst, recommendations must be backed up with compelling data insights and professional data visualizations, so Cyclistic executives must approve them.



## Ask
For this analysis, Moreno has set a clear goal. Design marketing strategies aimed at converting casual riders into annual members.

To help the marketing team accomplish this the below business task was created.

### Business Task
Analyse and understand how annual members and casual riders use Cyclistic bikes di erently.

### Stakeholders
**Lily Moreno:** The director of marketing and your manager. Moreno is responsible for the development of campaigns and initiatives to promote the bike-share program. These may include email, social media, and other channels.
 
**Cyclistic executive team:** The notoriously detail-oriented executive team will decide whether to approve the recommended marketing program.

## Prepare
Data to complete this analysis was stored in a [public database](https://divvy-tripdata.s3.amazonaws.com/index.html) published by Motivate International Inc. under this [license](https://divvybikes.com/data-license-agreement).

Twelve csv files were downloaded, covering the months of January 2023 - December 2023, creating a master version and working copy. 

The files are formatted in a "YYYYMM-divvy-tripdata.csv" format, indicating the year and month they represent. Opening the files shows headers including the following column headers:
- ride_id
- rideable_type
- started_at
- ended_at
- start_station_name
- start_station_id
- end_station_name
- end_station_id
- start_lat
- start_lng
- end_lat
- end_lng
- member_casual

## Process
Due to the large files sizes, **Google Bigquery** was implemented to work with the data, first combining all tables into one complete dataset.

### Combining the tables
In order to join all the tables together in order to work more efficiently, the below SQL was used:
```
CREATE TABLE IF NOT EXISTS `divvy-comp.data.2023-all-tripdata` AS
(
  SELECT * FROM `divvy-comp.data.202301-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202302-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202303-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202304-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202305-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202306-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202307-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202308-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202309-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202310-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202311-divvy-tripdata`
  UNION ALL
  SELECT * FROM `divvy-comp.data.202312-divvy-tripdata`
);
```
Completing this step it was important to check this new table included the entire dataset. Using 
```
  SELECT COUNT(*) AS number_of_rides
  FROM `divvy-comp.data.2023-all-tripdata` ;
```
Showed the new table contained 5,719,877 rides in the dataset. 

### Cleaning the Data
Despite having a table containing all the data, rows that are missing data have not been taken into account. Quickly running:
```
 SELECT 
  COUNTIF(ride_id IS null) ride_id,
  COUNTIF(rideable_type IS null) rideable_type,
  COUNTIF(started_at IS null) started_at,
  COUNTIF(ended_at IS null) ended_at,
  COUNTIF(start_station_name IS null) start_station_name,
  COUNTIF(start_station_id IS null) start_station_id,
  COUNTIF(end_station_name IS null) end_station_name,
  COUNTIF(end_station_id IS null) end_station_id,
  COUNTIF(start_lat IS null) start_lat,
  COUNTIF(start_lng IS null) start_lng,
  COUNTIF(end_lat IS null) end_lat,
  COUNTIF(end_lng IS null) end_lng,
  COUNTIF(member_casual IS null) member_casual
FROM `data.2023-all-tripdata` ;
```
Showed that there was data missing for the start station name and ID, end station name and ID as well as the end latitude and longitude.

In order to accurately analyse the data, it is important to first remove these values from the table. This was completed at the same time as transforming the data. 

While transforming the data, based on the started_at timestamp, the day of the week and month was extracted and placed in a new columns called day_of_week and month respectively. While doing this transformation the days and months were also converted into text for easier readability.

Additionally, ride length was calulated using the started_at and ended_at timestamps and placed in a new column called ride_length for later analysis.

All of this was completed while also exluding null values, and additionally rows where the ride length was less than a minute or more than 24 hours using the following query.
```
CREATE TABLE IF NOT EXISTS `divvy-comp.data.2023-cleaneddata` AS (
  SELECT 
    a.ride_id, rideable_type, started_at, ended_at, 
    ride_length,
    CASE EXTRACT(DAYOFWEEK FROM started_at) 
      WHEN 1 THEN 'Sun'
      WHEN 2 THEN 'Mon'
      WHEN 3 THEN 'Tue'
      WHEN 4 THEN 'Wed'
      WHEN 5 THEN 'Thu'
      WHEN 6 THEN 'Fri'
      WHEN 7 THEN 'Sat'    
    END AS day_of_week,
    CASE EXTRACT(MONTH FROM started_at)
      WHEN 1 THEN 'Jan'
      WHEN 2 THEN 'Feb'
      WHEN 3 THEN 'Mar'
      WHEN 4 THEN 'Apr'
      WHEN 5 THEN 'May'
      WHEN 6 THEN 'Jun'
      WHEN 7 THEN 'Jul'
      WHEN 8 THEN 'Aug'
      WHEN 9 THEN 'Sep'
      WHEN 10 THEN 'Oct'
      WHEN 11 THEN 'Nov'
      WHEN 12 THEN 'Dec'
    END AS month,
    start_station_name, end_station_name, 
    start_lat, start_lng, end_lat, end_lng, member_casual
  FROM `divvy-comp.data.2023-all-tripdata`  a
  JOIN (
    SELECT ride_id, (
      EXTRACT(HOUR FROM (ended_at - started_at)) * 60 +
      EXTRACT(MINUTE FROM (ended_at - started_at)) +
      EXTRACT(SECOND FROM (ended_at - started_at)) / 60) AS ride_length
    FROM `divvy-comp.data.2023-all-tripdata` 
  ) b 
  ON a.ride_id = b.ride_id
  WHERE 
    start_station_name IS NOT NULL AND
    end_station_name IS NOT NULL AND
    end_lat IS NOT NULL AND
    end_lng IS NOT NULL AND
    ride_length > 1 AND ride_length < 1440
);
```

## Analyse

## Share

## Act
