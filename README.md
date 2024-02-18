# Google Data Analytics Capstone Project

## Introduction

### About the Company
Cyclistic is a bike-share program that features more than 5,800 bicycles and 600 docking stations. Cyclistic sets itself apart by also offering reclining bikes, hand tricycles, and cargo bikes, making bike-share more inclusive to people with disabilities and riders who can’t use a standard two-wheeled bike. The majority of riders opt for traditional bikes; about 8% of riders use the assistive options. Cyclistic users are more likely to ride for leisure, but about 30% use the bikes to commute to work each day.

### Scenario
Lily Moreno, the director of marketing at Cyclistic believes the company’s future success depends on maximising the number of annual memberships. Therefore, the team wants to understand how casual riders and annual members use Cyclistic bikes differently. From these insights, a new marketing strategy will be designed to convert casual riders into annual members. But first, recommendations must be backed up with compelling data insights and professional data visualisations, so Cyclistic executives must approve them.

## Ask
For this analysis, Moreno has set a clear goal. Design marketing strategies aimed at converting casual riders into annual members.

To help the marketing team accomplish this the below business task was created.

### Business Task
Analyse and understand how annual members and casual riders use Cyclistic bikes differently.

### Stakeholders
**Lily Moreno:** The director of marketing and your manager. Moreno is responsible for the development of campaigns and initiatives to promote the bike-share program. These may include email, social media, and other channels.
 
**Cyclistic executive team:** The notoriously detail-oriented executive team will decide whether to approve the recommended marketing program.

## Prepare
Data to complete this analysis was stored in a [public database](https://divvy-tripdata.s3.amazonaws.com/index.html) published by Motivate International Inc. under this [licence](https://divvybikes.com/data-license-agreement).

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
This query showed that there was data missing for the start station name and ID, end station name and ID as well as the end latitude and longitude. Running the below query handled rows with null values, creating a new table in the process.
```
CREATE TABLE IF NOT EXISTS `divvy-comp.data.2023-cleaneddata` AS (
  SELECT 
    ride_id, 
    rideable_type, 
    started_at, 
    ended_at,
    start_station_name, 
    end_station_name, 
    start_lat, start_lng, 
    end_lat, 
    end_lng, 
    member_casual
  FROM 
    `divvy-comp.data.2023-all-tripdata`  
  WHERE 
    start_station_name IS NOT NULL AND
    end_station_name IS NOT NULL AND
    end_lat IS NOT NULL AND
    end_lng IS NOT NULL
);
```

## Analyse
With a clean dataset it was important to perform calculations in order to better understand what the data was showing. To do this, it was important to know the following information on each ride: 
- Day of week
- Month
- Ride length

To complete these calculations we used the CASE EXTRACT function in Google BigQuery, utilising the started_at and ended_at timestamps in order to complete these calculations.

Running the below query created a new table (unless it existed) adding columns fora day of week, month and ride length. While performing these calculations it was decided to exclude rides where the ride length was less than 1 minute or more than 24 hours as these were deemed as outliers.
```
CREATE TABLE IF NOT EXISTS `divvy-comp.data.2023-cleaneddata_mod` AS (
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
  FROM `divvy-comp.data.2023-cleaneddata`  a
  JOIN (
    SELECT ride_id, (
      EXTRACT(HOUR FROM (ended_at - started_at)) * 60 +
      EXTRACT(MINUTE FROM (ended_at - started_at)) +
      EXTRACT(SECOND FROM (ended_at - started_at)) / 60) AS ride_length
    FROM `divvy-comp.data.2023-cleaneddata` 
  ) b 
  ON a.ride_id = b.ride_id
  WHERE
    ride_length > 1 AND ride_length < 1440
);
```

## Share
With the data now cleaned and processed, a table with all rides and no missing values was able to be used to create valid visualisations to adequately present findings. 

The tool used for this was Tableau, and the dashboard can be seen [here](https://public.tableau.com/views/GoogleDataAnalyticsCapstoneProjectCyclistic_17082839734260/CyclisticCaseStudy?:language=en-US&:sid=&:display_count=n&:origin=viz_share_link). 

![Dashboard Screenshot](<img width="1211" alt="Screenshot 2024-02-18 at 15 08 34" src="https://github.com/jxhos0/Google-Data-Analytics-Capstone-Project/assets/65057135/ff3b6309-d424-421f-babf-709c51227213">)

### Observations of Member Usage
Reviewing the data a few key observations for how members use Cyclistic were noted:
- Based on the times of the trips, members primarily used Cyclistic for commuting, primarily busiest from 6 - 8AM and from 4 - 6PM.
- Members have the lowest usage on weekend, with Tuesday to Thursday being the busiest. This again backs up the observation of members using the service to commute.
- The summer months are the most popular time for members to use the service, recording approximately twice as many rides during the peak in summer than the middle of winter. This would be attributed to weather and conditions.

### Observations of Casual Usage
- Casual usage steadily increases from 5AM till a peak of 5PM for casual riders.
- Usage by day indicates that casual riders are primarily using the service on weekends, the lowest number of trips being recorded during the week.
- Similar to the members, casual riders use the service the most through the summer months, and the least during the winter months.

### Comparing Casual Riders to Members
First and foremost it is important to point out that members make up 64.53% of all trips recorded in the 2023 year. This variance in usage is consistent across all ride type (except docker bikes), where members completed more than 60% of the trips. 

However, it is also important to note that across all timeframes (hour, day and month), the average trip duration of a casual rider was always longer than that of a member. 

Analysing the starting locations of both casual riders and members, there is no significant difference, with the bulk of trips being taken around the city centre and along the coast.

## Act
Following the analysis of the data, three (3) key recommendations can be made to help convert more casual riders to members.
1. Since casual riders on average have longer trips, providing discounts to members for extended trip durations could incentivise more casual riders to sign up.
2. As casual riders typically use the service most on weekends, providing a discounted weekday rate for new members could prove an effective way to promote more sign-ups.
3. Marketing member only deals for the winter months would prove useful for increasing usage by casual riders, as during this time casual trips are on average on 25% of the members. 
