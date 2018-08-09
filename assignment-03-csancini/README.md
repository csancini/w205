# template-activity-03


# Query Project

- In the Query Project, you will get practice with SQL while learning about
  Google Cloud Platform (GCP) and BiqQuery. You'll answer business-driven
  questions using public datasets housed in GCP. To give you experience with
  different ways to use those datasets, you will use the web UI (BiqQuery) and
  the command-line tools, and work with them in jupyter notebooks.

- We will be using the Bay Area Bike Share Trips Data
  (https://cloud.google.com/bigquery/public-data/bay-bike-share). 

#### Problem Statement
- You're a data scientist at Ford GoBike (https://www.fordgobike.com/), the
  company running Bay Area Bikeshare. You are trying to increase ridership, and
  you want to offer deals through the mobile app to do so. What deals do you
  offer though? Currently, your company has three options: a flat price for a
  single one-way trip, a day pass that allows unlimited 30-minute rides for 24
  hours and an annual membership. 

- Through this project, you will answer these questions: 
  * What are the 5 most popular trips that you would call "commuter trips"?
  * What are your recommendations for offers (justify based on your findings)?


## Assignment 03 - Querying data from the BigQuery CLI - set up 

### What is Google Cloud SDK?
- Read: https://cloud.google.com/sdk/docs/overview

- If you want to go further, https://cloud.google.com/sdk/docs/concepts has
  lots of good stuff.

### Get Going

- Install Google Cloud SDK: https://cloud.google.com/sdk/docs/

- Try BQ from the command line:

  * General query structure

    ```
    bq query --use_legacy_sql=false '
        SELECT count(*)
        FROM
           `bigquery-public-data.san_francisco.bikeshare_trips`'
    ```

### Queries

1. Rerun last week's queries using bq command line tool (Paste your bq
   queries):

- What's the size of this dataset? (i.e., how many trips)

  ```sql
  bq query --use_legacy_sql=false 'SELECT count(*) FROM `bigquery-public-data.san_francisco.bikeshare_trips`'
  ```

- What is the earliest start time and latest end time for a trip?

  ```sql
  bq query --use_legacy_sql=false 'SELECT min(start_date), max(end_date) FROM `bigquery-public-data.san_francisco.bikeshare_trips`'
  ```

- How many bikes are there?

  ```sql
  bq query --use_legacy_sql=false 'SELECT count(distinct bike_number) FROM `bigquery-public-data.san_francisco.bikeshare_trips`'
  ```

2. New Query (Paste your SQL query and answer the question in a sentence):

- How many trips are in the morning vs in the afternoon?

  * Answer

    There are 412,339 trips in the morning and 571,309 trips in the afternoon, considering the start period as the criterion for counting the number of trips. If it is also considered the end period and the day of the trip, the number varies according to the table below.

    Devolution|Start Period|End Period|Number of Trips
    :---:|:---:| :---:|---:
    same day|morning|morning|398,132
    same day|morning|afternoon|13,990
    same day|afternoon|afternoon|568,943
    other day|morning|morning|141
    other day|morning|afternoon|76
    other day|afternoon|morning|1,871
    other day|afternoon|afternoon|495
  
  * Formatted Query
    ```sql
    #standardSQL
    SELECT
      IF ((EXTRACT(DAY FROM start_date) = EXTRACT(DAY FROM end_date)), "same day", "other day") AS devoution,
      IF ((EXTRACT(HOUR FROM start_date) BETWEEN 0 AND 11), "morning", "afternoon") AS start_period,
      IF ((EXTRACT(HOUR FROM end_date) BETWEEN 0 AND 11), "morning", "afternoon") AS end_period,
      COUNT(*) AS num_trips
    FROM
      `bigquery-public-data.san_francisco.bikeshare_trips`
    GROUP BY
      start_period,
      end_period,
      devoution
    ORDER BY
      devoution DESC,
      start_period DESC,
      end_period DESC
    ```

  * BQ Query
    ```sql
    bq query --use_legacy_sql=false 'SELECT IF ((EXTRACT(DAY FROM start_date) = EXTRACT(DAY FROM end_date)), "same day", "other day") AS devoution, IF ((EXTRACT(HOUR FROM start_date) BETWEEN 0 AND 11), "morning", "afternoon") AS start_period, IF ((EXTRACT(HOUR FROM end_date) BETWEEN 0 AND 11), "morning", "afternoon") AS end_period, count(*) as num_trips FROM `bigquery-public-data.san_francisco.bikeshare_trips` GROUP BY start_period, end_period, devoution ORDER BY devoution desc, start_period desc, end_period desc'
    ```

### Project Questions
Identify the main questions you'll need to answer to make recommendations (list
below, add as many questions as you need).

- Question 1: How does each landmark differ in terms of demand for bikes?

- Question 2: What are the busiest destination stations that would make ad placement impact a greater number of people?

- Question 3: Is the behavior of subscribers and customers different considering the hours of use?

- Question 4: Does the duration of the trip vary according to different regions and types of customers?

- Question 5: What are the most popular Subscribers routes, considering different regions? Is it posssible to identify commuting behavior?

- Question 6: What are the most popular Customer routes, considering different regions? Is it possible to identify patterns of behavior?

- Question 7: Are there stations that have shortages of bicycles at different times of the day that can lead to loss of income?

- Question 8: Are there underutilized stations that would benefit from incentives?

- Question 8: Is it possible to identify behaviors associated with longer distances that can lead to insights on how to increase bike usage?

### Answers

Answer at least 4 of the questions you identified above You can use either
BigQuery or the bq command line tool.  Paste your questions, queries and
answers below.

- Question 1: How does each landmark differ in terms of demand for bikes?
  * Answer:
    
    In first place, San Francisco concentrates 90% of demand of bikes. Sao Jose, the second place, have only 5% of the total demand.

    Number of Trips | Landmarks
    ---:|:---
    891,232 | San Francisco
    52,869 | San Jose
    24,707 | Mountain View
    9,923 | Palo Alto
    4,917 | Redwood City

  * SQL query:

    ```sql
    #standardSQL
    SELECT
      COUNT(trips.trip_id) AS num_trips,
      stations.landmark as landmark
    FROM
      `bigquery-public-data.san_francisco.bikeshare_trips` trips
      INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` stations 
        ON trips.end_station_id = stations.station_id
    GROUP BY
      stations.landmark
    ORDER BY
      num_trips DESC
    ```

- Question 2: What are the busiest destination stations that would make ad placement impact a greater number of people?
  * Answer:

    San Francisco's top 10 stations account for nearly 50% of the trips of this landmark:

    Number of Trips | Destination Station Name | Landmark
    ---:|:---|:---
    92,014 | San Francisco Caltrain (Townsend at 4th)      | San Francisco |
    58,713 | San Francisco Caltrain 2 (330 Townsend)       | San Francisco |
    50,185 | Harry Bridges Plaza (Ferry Building)          | San Francisco |
    46,197 | Embarcadero at Sansome                        | San Francisco |
    44,145 | 2nd at Townsend                               | San Francisco |
    40,956 | Market at Sansome                             | San Francisco |
    39,598 | Steuart at Market                             | San Francisco |
    38,545 | Townsend at 7th                               | San Francisco |
    35,477 | Temporary Transbay Terminal (Howard at Beale) | San Francisco |
    26,762 | Market at 4th                                 | San Francisco |

    For other landmarks only the first 2 of each are listed, as the number of users decreases to low values:

    Number of Trips | Destination Station Name | Landmark
    ---:|:---|:---
    13,315 | San Jose Diridon Caltrain Station | San Jose
    5,158 | Santa Clara at Almaden | San Jose
    9,231 | Mountain View Caltrain Station | Mountain View
    4,651 | Mountain View City Hall | Mountain View
    2,847 | Palo Alto Caltrain Station | Palo Alto
    2,351 | University and Emerson | Palo Alto
    1,870 | Redwood City Caltrain Station | Redwood City
    917 | Stanford in Redwood City | Redwood City

  * SQL query:
    
    ```sql
    -- ROW_NUMBER and PARTITION was used to create a ranking by landmark
    #standardSQL
    WITH MyRowSet AS
    (
      SELECT
        *, ROW_NUMBER() OVER (PARTITION BY landmark ORDER BY num_trips DESC) AS row_num
      FROM 
      (
        SELECT
          COUNT(trips.trip_id) AS num_trips,
          trips.end_station_name AS station,
          stations.landmark as landmark
        FROM
          `bigquery-public-data.san_francisco.bikeshare_trips` trips
          INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` stations 
            ON trips.end_station_id = stations.station_id
        GROUP BY
          trips.end_station_name,
          stations.landmark
        )
    )
    SELECT * FROM MyRowSet WHERE row_num <= 2 -- 10 was used for San Francisco
    ```

- Question 3: Is the behavior of subscribers and customers different considering the hours of use?
  * Answer:

    Yes. Peak times for subscribers occur during the early morning and in the late afternoon, while for customers usage is concentrated in the afternoon.

    | Number of Trips for Subscribers | Hour of Use |
    |---:|:---|
    |                127,171 |        8 |
    |                114,915 |       17 |
    |                 89,546 |        9 |
    |                 76,051 |       16 |
    |                 75,798 |       18 |
    |                 64,946 |        7 |

    | Number of Trips for Customers | Hour of Use |
    |---:|:---|
    |                 12,806 |       15 |
    |                 12,737 |       14 |
    |                 12,719 |       13 |
    |                 12,704 |       16 |
    |                 12,508 |       12 |
    |                 11,387 |       17 |
    |                 11,078 |       11 |
    |                  8,771 |       18 |

  * SQL query:
    ```sql
    #standardSQL
    SELECT
      count(trip_id) AS num_trips_subscribers, 
      EXTRACT(HOUR FROM start_date) as the_hour
    FROM
      `bigquery-public-data.san_francisco.bikeshare_trips`
    WHERE
      subscriber_type = 'Subscriber'
    GROUP BY
      the_hour
    ORDER BY 
      num_trips_subscribers DESC
    ```
    ```sql
    #standardSQL
    SELECT
      count(trip_id) AS num_trips_customers, 
      EXTRACT(HOUR FROM start_date) as the_hour
    FROM
      `bigquery-public-data.san_francisco.bikeshare_trips`
    WHERE
      subscriber_type = 'Customer'
    GROUP BY
      the_hour
    ORDER BY 
      num_trips_customers DESC
    ```
- Question 4: Does the duration of the trip vary according to different regions and types of customers?
  * Answer:
    
    Yes. The trip duration for subscribers is shorter and varies only a few minutes for the different regions, while for customers the trip duration is longer and the variation is wider.

    | Average Trip (min) | Subscriber Type |  Landmark    |
    | ---: | :--- |  :---    |
    |             152 | Customer        | Redwood City  |
    |             128 | Customer        | Palo Alto     |
    |             123 | Customer        | Mountain View |
    |              78 | Customer        | San Jose      |
    |              56 | Customer        | San Francisco |
    |              15 | Subscriber      | Palo Alto     |
    |              12 | Subscriber      | Redwood City  |
    |              10 | Subscriber      | San Francisco |
    |              10 | Subscriber      | San Jose      |
    |               9 | Subscriber      | Mountain View |

  * SQL query:
    ```sql
    #standardSQL
    SELECT
      ROUND(AVG(trips.duration_sec)/60) as avg_trip_duration,
      trips.subscriber_type,
      stations.landmark
    FROM
      `bigquery-public-data.san_francisco.bikeshare_trips` trips 
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` stations 
        ON trips.start_station_id = stations.station_id
    GROUP BY
      trips.subscriber_type,
      landmark
    ORDER BY
      avg_trip_duration DESC
    ```

- Question 5: What are the most popular Subscribers routes, considering different regions? Is it posssible to identify commuting behavior?
  * Answer:

    The most popular routes are associated with origins and destinations of the public transportation system (trains or ferries). Possibly, people use public transportation options to travel great distances and bikes to reach the final stop, or to change the type of transportation (from train to ferry and vice versa). This pattern seems to repeat in San Francisco and the others landmarks. Additionally, rations between trips (returing/going) closest to 1 indicates that most of the routes may be associated with commuters (higher frequency of round trips).
    
    Trips (going) |	Trips (returning) |	Ratio |	Station A |	Station B  |	Landmark
    :---: |	:---: |	:---: |	:--- |	:--- |	:---
    8,305 |	6,641 |	0.80 |	San Francisco Caltrain 2 (330 Townsend) |	Townsend at 7th  |	San Francisco
    6,931 |	6,332 |	0.91 |	2nd at Townsend |	Harry Bridges Plaza (Ferry Building) |	San Francisco
    5,758 |	4,079 |	0.71 |	Steuart at Market |	2nd at Townsend |	San Francisco
    5,709 |	4,980 |	0.87 |	San Francisco Caltrain (Townsend at 4th) |	Harry Bridges Plaza (Ferry Building) |	San Francisco
    5,695 |	4,182 |	0.73 |	Steuart at Market |	San Francisco Caltrain (Townsend at 4th) |	San Francisco
    5,089 |	5,699 |	1.12 |	San Francisco Caltrain (Townsend at 4th) |	Temporary Transbay Terminal (Howard at Beale) |	San Francisco
    4,919 |	5,205 |	1.06 |	Market at Sansome |	2nd at South Park |	San Francisco
    4,367 |	6,158 |	1.41 |	San Francisco Caltrain (Townsend at 4th) |	Embarcadero at Folsom	|	San Francisco
    4,352 |	4,281 |	0.98 |	San Francisco Caltrain 2 (330 Townsend) |	Powell Street BART |	San Francisco
    4,298 |	3,100 |	0.72 |	San Francisco Caltrain 2 (330 Townsend) |	5th at Howard	|	San Francisco

    Trips (going) |	Trips (returning) |	Ratio |	Station A |	Station B | Landmark
    :---: |	:---: |	:---: |	:--- |	:--- |	:---
    3,765 |	3,393 |	0.90 |	Mountain View Caltrain Station |	Mountain View City Hall |	Mountain View
    2,385 |	2,172 |	0.91 |	Castro Street and El Camino Real |	Mountain View Caltrain Station |	Mountain View
    3,146 |	3,140 |	1.00 |	Santa Clara at Almaden |	San Jose Diridon Caltrain Station |	San Jose
    1,504 |	1,598 |	1.06 |	San Pedro Square |	San Jose Diridon Caltrain Station |	San Jose
    1,219 |	1,117 |	0.92 |	Cowper at University |	Palo Alto Caltrain Station |	Palo Alto
    321 |	449 |	1.40 |	Park at Olive |	Palo Alto Caltrain Station |	Palo Alto
    580 |	666 |	1.15 |	Stanford in Redwood City |	Redwood City Caltrain Station |	Redwood City
    580 |	9 |	0.02 |	Stanford in Redwood City |	Redwood City Caltrain Station |	Redwood City

  * SQL query:
    ```sql
    --- First, the query just creates a temp table (Routes) with the popular routes. It also joins 
    --- with the stations table to get the landmark name. The table
    --- contain two lines for each route (A as origin and B as destination, and B  as origin and 
    --- A as destination).
    --- Second, Route table is joinned with itself to identify the round trips (going = # of trips
    --- from A to B, returning = # of trips from B to A). The ratio is simply a estimate measure 
    --- of round trips.
    #standardSQL
    WITH Routes AS (
      SELECT
        COUNT(*) AS num_trips,
        trips.subscriber_type,
        trips.start_station_id,
        trips.start_station_name,
        a.landmark AS start_landmark,
        trips.end_station_id,
        trips.end_station_name,
        b.landmark AS end_landmark
      FROM
        `bigquery-public-data.san_francisco.bikeshare_trips` trips
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` a
        ON trips.start_station_id = a.station_id
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` b
        ON trips.end_station_id = b.station_id
      WHERE
        subscriber_type = "Subscriber" AND
        b.landmark = "San Francisco"  -- <> for the 2nd table
      GROUP BY
        trips.subscriber_type,
        trips.start_station_id,
        trips.start_station_name,
        start_landmark,
        trips.end_station_id,
        trips.end_station_name,
        end_landmark
      ORDER BY
        num_trips DESC
    )
    SELECT
      a.num_trips going,
      b.num_trips return,
      round(b.num_trips/a.num_trips, 2) as ratio,
      a.start_station_name,
      a.end_station_name,
      a.end_landmark
    FROM
      Routes a
      INNER JOIN Routes b
      ON a.end_station_id = b.start_station_id AND b.end_station_id = a.start_station_id
    WHERE
      a.start_station_id > a.end_station_id
    ORDER BY
      going DESC,
      return DESC
    ```

- Question 6: What are the most popular Customer routes, considering different regions? Is it possible to identify patterns of behavior?
  * Answer:
    
    In San Francisco, most subscriber's routes may be related to ferry / boat transportation (or to city attractions/main places closest to the bay). Proportions less close to 1 indicate that usage is less associated with round trips. As seen earlier, other information that adds to this view is the longest average travel time for non-subscribers, and also the concentration of travel in the afternoon. Trips for Customers of other landmarks than San Francisco could also be associated with train transportation (or to city attractions/main places closest to train stations).

    Trips (going) |	Trips (returning) |	Ratio |	Station A |	Station B | Landmark
    :---: |	:---: |	:---: |	:--- |	:--- |	:---
    1638 |	3667 |	2.24 |	Embarcadero at Sansome |	Harry Bridges Plaza (Ferry Building) |	San Francisco
    868 |	420 |	0.48 |	Harry Bridges Plaza (Ferry Building) |	Embarcadero at Vallejo |	San Francisco
    847 |	674 |	0.80  |	Steuart at Market |	Embarcadero at Sansome |	San Francisco
    770 |	701 |	0.91 |	Market at 4th |	Embarcadero at Sansome |	San Francisco
    689 |	556 |	0.81 |	2nd at Townsend |	Harry Bridges Plaza (Ferry Building) |	San Francisco
    636 |	420 |	0.66 |	Market at Sansome |	Embarcadero at Sansome |	San Francisco
    619 |	576 |	0.93 |	Powell at Post (Union Square) |	Embarcadero at Sansome |	San Francisco
    613 |	476 |	0.78 |	Market at 4th |	Harry Bridges Plaza (Ferry Building) |	San Francisco
    608 |	1345 |2.21 |	Embarcadero at Sansome |	Embarcadero at Vallejo |	San Francisco
    546 |	583 |	1.07 |	2nd at Townsend |	Embarcadero at Sansome |	San Francisco
    
    Trips (going) |	Trips (returning) |	Ratio |	Station A |	Station B | Landmark
    :---: |	:---: |	:---: |	:--- |	:--- |	:---
    164 |	137 |	0.84 |	California Ave Caltrain Station |	University and Emerson |	Palo Alto
    107 |	113 |	1.06 |	California Ave Caltrain Station |	Palo Alto Caltrain Station |	Palo Alto
    146 |	94 |	0.64 |	Castro Street and El Camino Real |	Mountain View Caltrain Station |	Mountain View
    98 |	77 |	0.79 |	Evelyn Park and Ride |	Mountain View Caltrain Station |	Mountain View
    128 |	110 |	0.86 |	San Pedro Square |	San Jose Civic Center |	San Jose
    112 |	126 |	1.13 |	San Jose Civic Center |	San Jose Diridon Caltrain Station |	San Jose
    47 |	60 |	1.28 |	Stanford in Redwood City |	Redwood City Caltrain Station |	Redwood City
    47 |	4 |	0.09 |	Stanford in Redwood City |	Redwood City Caltrain Station |	Redwood City

  * SQL query:
    ```sql
    #standardSQL
    WITH Routes AS (
      SELECT
        COUNT(*) AS num_trips,
        trips.subscriber_type,
        trips.start_station_id,
        trips.start_station_name,
        a.landmark AS start_landmark,
        trips.end_station_id,
        trips.end_station_name,
        b.landmark AS end_landmark
      FROM
        `bigquery-public-data.san_francisco.bikeshare_trips` trips
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` a
        ON trips.start_station_id = a.station_id
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` b
        ON trips.end_station_id = b.station_id
      WHERE
        subscriber_type = "Customer" AND
        b.landmark = "San Francisco" -- <> for the 2nd table
      GROUP BY
        trips.subscriber_type,
        trips.start_station_id,
        trips.start_station_name,
        start_landmark,
        trips.end_station_id,
        trips.end_station_name,
        end_landmark
      ORDER BY
        num_trips DESC
    )
    SELECT
      a.num_trips going,
      b.num_trips return,
      round(b.num_trips/a.num_trips, 2) as ratio,
      a.start_station_name,
      a.end_station_name,
      a.end_landmark
    FROM
      Routes a
      INNER JOIN Routes b
      ON a.end_station_id = b.start_station_id AND b.end_station_id = a.start_station_id
    WHERE
      a.start_station_id > a.end_station_id
    ORDER BY
      going DESC,
      return DESC
    ```
