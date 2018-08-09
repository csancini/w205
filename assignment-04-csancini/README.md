
# Query Project
- In the Query Project, you will get practice with SQL while learning about Google Cloud Platform (GCP) and BiqQuery. You'll answer business-driven questions using public datasets housed in GCP. To give you experience with different ways to use those datasets, you will use the web UI (BiqQuery) and the command-line tools, and work with them in jupyter notebooks.
- We will be using the Bay Area Bike Share Trips Data (https://cloud.google.com/bigquery/public-data/bay-bike-share). 

#### Query Project Problem Statement
- You're a data scientist at Ford GoBike (https://www.fordgobike.com/), the company running Bay Area Bikeshare. You are trying to increase ridership, and you want to offer deals through the mobile app to do so. What deals do you offer though? Currently, your company has three options: a flat price for a single one-way trip, a day pass that allows unlimited 30-minute rides for 24 hours and an annual membership. 

- Through this project, you will answer these questions: 
  * What are the 5 most popular trips that you would call "commuter trips"?
  * What are your recommendations for offers (justify based on your findings)?

_______________________________________________________________________________________________________________


## Assignment 04 - Querying data - Answer Your Project Questions

### Your Project Questions

- Answer at least 4 of the questions you identified last week.
- You can use either BigQuery or the bq command line tool.
- Paste your questions, queries and answers below.

#### Question List

- Question 1: How does each landmark differ in terms of demand for bikes?

- Question 2: What are the busiest destination stations that would make ad placement impact a greater number of people?

- Question 3: Is the behavior of subscribers and customers different considering the hours of use?

- Question 4: Does the duration of the trip vary according to different regions and types of customers?

- Question 5: What are the most popular Subscribers routes, considering different regions? Is it posssible to identify commuting behavior?

- Question 6: What are the most popular Customer routes, considering different regions? Is it possible to identify patterns of behavior?

- Question 7: Are there top stations that have shortages of bicycles that could lead to loss of income or that represent a barrier to increase ridership?

- Question 8: Is it possible to identify behaviors associated with longer distances that can lead to insights on how to increase bike usage?

#### Answers

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

    For other landmarks only the first 2 of each are listed because the number of users decreases to very low values:

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
    SELECT * FROM MyRowSet WHERE row_num <= 2
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
- Question 7: Are there top stations that have shortages of bicycles that could lead to loss of income or that represent a barrier to increase ridership?
  * Answer:

    When analyzing the stations' log status (updated per minute), the busiest stations presented in the analysis of the 2nd question have problems of low availability of bikes. The criterion considered for low availability was no bikes available or only one bike available.
    
    This fact may represent an inhibitor of frequent users, who stop using bicycles or do not consider them as a means for commuting. It is necessary to investigate whether the return on investment associated with the increase of bikes is justifiable, considering peek hours and amount of time of unavailability.

    The table below shows the number of observations (in thousands) for each station in which there were low availability of bikes in San Francisco for each week day. Unavailability is concentrated in work days with only a few stations having problems in weekends.  
    
    Landmark	| Station	| S |	M | T | W |	T |	F |	S
    :--- |	:--- | :---: |	:---: |	:---: |	:---: |	:---: |	:---: |	:---:
    San Francisco | San Francisco Caltrain (Townsend at 4th) |	0.3 |	15.1 |	14.2 |	14.8 |	14.0 |	12.3 |	0.7
    San Francisco | San Francisco Caltrain 2 (330 Townsend) |	0.2 |	11.6 |	9.5 |	10.4 |	9.4 |	9.6 |	0.4
    San Francisco | Harry Bridges Plaza (Ferry Building) |	1.4 |	7.4 |	7.3 |	7.2 |	5.7 |	7.9 |	1.2
    San Francisco | Embarcadero at Sansome |	7.4 |	12.0 |	15.1 |	14.8 |	12.0 |	10.8 |	7.9
    San Francisco | 2nd at Townsend |	1.1 |	3.7 |	2.4 |	2.7 |	3.4 |	2.6 |	2.3
    San Francisco | Market at Sansome |	1.9 |	2.1 |	1.3 |	2.1 |	2.0 |	2.3 |	2.4
    San Francisco | Steuart at Market  |	0.5 |	7.0 |	5.7 |	5.4 |	6.8 |	5.5 |	0.8
    San Francisco | Townsend at 7th  |	3.6 |	10.1 |	10.7 |	9.1 |	9.7 |	7.1 |	2.5
    San Francisco | Temporary Transbay Terminal |	0.4 |	7.4 |	8.6 |	7.1 |	8.7 |	7.4 |	0.3
    San Francisco | Market at 4th |	2.7 |	6.6 |	16.1 |	16.2 |	17.5 |	13.5 |	4.2

    Landmark	| Station	| S |	M | T | W |	T |	F |	S
    :--- |	:--- | :---: |	:---: |	:---: |	:---: |	:---: |	:---: |	:---:
    San Jose |	San Jose Diridon Caltrain Station	|	0.1	|	1.5	|	2.9	|	1.7	| 2.3	|	2.3	|	1.6
    San Jose |	Santa Clara at Almaden	|	3.6	|	7.7	|	12.1	|	9.9	|	10.3	|	8.9	|	8.9
    Mountain View |	Mountain View Caltrain Station	|	0.6	|	1.6	|	0.9	|	2.2	|	3.5	|	2.6	|	0.2
    Mountain View	| Mountain View City Hall	|	6.4	|	4.9	|	7.2	|	6.3	|	4.7	|	5.9	|	8.3
    Palo Alto	|	Palo Alto Caltrain Station	|	1.6	|	2.0	|	1.8	|	2.8	|	2.0	|	3.6	|	2.8
    Palo Alto	|	University and Emerson	|	4.9	|	4.4	|	0.3	|	0.6	|	0.7	|	1.8	|	2.6
    Redwood City	|	Redwood City Caltrain Station	|	0.7	|	2.0	|	2.2	|	2.9	|	1.5	|	2.8	|	2.7
    Redwood City	|	Stanford in Redwood City	|	1.0	|	1.7	|	3.3	|	1.5	|	2.1	|	2.2	|	2.0

  * SQL query:
    ```sql
    #standardSQL
    WITH UnavailableBikesByStation AS
    (
      SELECT
        station_id,
        count(*) AS num_obs,
        EXTRACT(DAYOFWEEK FROM time) as day_of_week
      FROM
        `bigquery-public-data.san_francisco.bikeshare_status`
      WHERE
        bikes_available IN (0,1)
      GROUP BY
        station_id,
        day_of_week
      ORDER BY
        station_id, num_obs DESC, day_of_week
    )
    SELECT  
      b.landmark,
      b.name,
      ROUND(SUM(CASE WHEN day_of_week = 1   THEN num_obs END)/1000, 1) AS Sunday, 
      ROUND(SUM(CASE WHEN day_of_week = 2   THEN num_obs END)/1000, 1) AS Monday, 
      ROUND(SUM(CASE WHEN day_of_week = 3   THEN num_obs END)/1000, 1) AS Tuesday,
      ROUND(SUM(CASE WHEN day_of_week = 4   THEN num_obs END)/1000, 1) AS Wednesday,
      ROUND(SUM(CASE WHEN day_of_week = 5   THEN num_obs END)/1000, 1) AS Thursday,
      ROUND(SUM(CASE WHEN day_of_week = 6   THEN num_obs END)/1000, 1) AS Friday,
      ROUND(SUM(CASE WHEN day_of_week = 7   THEN num_obs END)/1000, 1) AS Saturday
    FROM
      UnavailableBikesByStation AS a
    INNER JOIN 
      `bigquery-public-data.san_francisco.bikeshare_stations` AS b
      ON a.station_id = b.station_id
    WHERE 
      a.station_id IN (70,69,50,60,61,77,74,65,55,76,28,27,34,35,22,25,2,4) -- the query was taking too long, so the stations where hardcoded 
    GROUP BY 
      a.station_id, 
      b.landmark,
      b.name  
    ```
- Question 8: Is it possible to identify behaviors associated with longer distances that can lead to insights on how to increase bike usage?
  * Answer:

    The aim behind this analysis is to identify if distance between stations is negatively correlated with ridership.

    The answer for this question involves mathematical calculations using latitude and longitude to identify approximate distances between stations. The distance `D` in miles is achieved through the formula:

    `D = ACOS(COS(RADIANS(90-Lat1)) * COS(RADIANS(90-Lat2)) + SIN(RADIANS(90-Lat1)) * SIN(RADIANS(90-Lat2)) * COS(RADIANS(Long2-Long1))) * 3958.756`

    This analysis will be done in Python, as this calculation would be quite complex in SQL since there are no such functions available. The query was used to obtain latitude and longitude for the top 10 subscriber routes in San Francisco.

    | Station A | Latitude A | Longitude A | Station B | Latitude B | Longitude B |
    | :--- | :--- | :--- | :--- | :--- | :--- |
    | San Francisco Caltrain 2 (330 Townsend)\s| 37.7766 | -122.39547 | Townsend at 7th | 37.771058 | -122.402717 |
    | 2nd at Townsend | 37.780526 | -122.390288 | Harry Bridges Plaza (Ferry Building) | 37.795392 | -122.394203 |
    | Steuart at Market | 37.794139 | -122.394434 | 2nd at Townsend | 37.780526 | -122.390288 |
    | San Francisco Caltrain (Townsend at 4th) | 37.776617 | -122.39526 | Harry Bridges Plaza (Ferry Building) | 37.795392 | -122.394203 |
    | Steuart at Market | 37.794139 | -122.394434 | San Francisco Caltrain (Townsend at 4th) | 37.776617 | -122.39526 |
    | San Francisco Caltrain (Townsend at 4th) | 37.776617 | -122.39526 | Temporary Transbay Terminal | 37.789756 | -122.394643 |
    | Market at Sansome | 37.789625 | -122.400811 | 2nd at South Park | 37.782259 | -122.392738 |
    | San Francisco Caltrain (Townsend at 4th) | 37.776617 | -122.39526 | Embarcadero at Folsom | 37.791464 | -122.391034 |
    | San Francisco Caltrain 2 (330 Townsend) | 37.7766 | -122.39547 | Powell Street BART | 37.783871 | -122.408433 |
    | San Francisco Caltrain 2 (330 Townsend) | 37.7766 | -122.39547 | 5th at Howard | 37.781752 | -122.405127 |

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
        b.landmark AS end_landmark,
        a.latitude AS start_latitude,
        a.longitude AS start_longitude,
        b.latitude AS end_latitude,
        b.longitude AS end_longitude
      FROM
        `bigquery-public-data.san_francisco.bikeshare_trips` trips
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` a
        ON trips.start_station_id = a.station_id
        INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` b
        ON trips.end_station_id = b.station_id
      WHERE
        subscriber_type = "Subscriber" AND
        b.landmark = "San Francisco"
      GROUP BY
        trips.subscriber_type,
        trips.start_station_id,
        trips.start_station_name,
        start_landmark,
        trips.end_station_id,
        trips.end_station_name,
        end_landmark,
        a.latitude,
        a.longitude,
        b.latitude,
        b.longitude
      ORDER BY
        num_trips DESC
    )
    SELECT
      a.start_station_name,
      a.start_latitude,
      a.start_longitude,    
      a.end_station_name,
      a.end_latitude,
      a.end_longitude,
      a.end_landmark
    FROM
      Routes a
      INNER JOIN Routes b
      ON a.end_station_id = b.start_station_id AND b.end_station_id = a.start_station_id
    WHERE
      a.start_station_id > a.end_station_id
    ORDER BY
      a.num_trips DESC,
      b.num_trips DESC
    LIMIT 10
    ```
