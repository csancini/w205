# Query Project
- In the Query Project, you will get practice with SQL while learning about Google Cloud Platform (GCP) and BiqQuery. You'll answer business-driven questions using public datasets housed in GCP. To give you experience with different ways to use those datasets, you will use the web UI (BiqQuery) and the command-line tools, and work with them in jupyter notebooks.
- We will be using the Bay Area Bike Share Trips Data (https://cloud.google.com/bigquery/public-data/bay-bike-share). 

#### Problem Statement
- You're a data scientist at Ford GoBike (https://www.fordgobike.com/), the company running Bay Area Bikeshare. You are trying to increase ridership, and you want to offer deals through the mobile app to do so. What deals do you offer though? Currently, your company has three options: a flat price for a single one-way trip, a day pass that allows unlimited 30-minute rides for 24 hours and an annual membership. 

- Through this project, you will answer these questions: 
  * What are the 5 most popular trips that you would call "commuter trips"?
  * What are your recommendations for offers (justify based on your findings)?


## Assignment 02: Querying Data with BigQuery

### What is Google Cloud?
- Read: https://cloud.google.com/docs/overview/

### Get Going

- Go to https://cloud.google.com/bigquery/
- Click on "Try it Free"
- It asks for credit card, but you get $300 free and it does not autorenew after the $300 credit is used, so go ahead (OR CHANGE THIS IF SOME SORT OF OTHER ACCESS INFO)
- Now you will see the console screen. This is where you can manage everything for GCP
- Go to the menus on the left and scroll down to BigQuery
- Now go to https://cloud.google.com/bigquery/public-data/bay-bike-share 
- Scroll down to "Go to Bay Area Bike Share Trips Dataset" (This will open a BQ working page.)


### Some initial queries
Paste your SQL query and answer the question in a sentence.

- What's the size of this dataset? (i.e., how many trips)

```sql
#standardSQL
SELECT count(*) FROM `bigquery-public-data.san_francisco.bikeshare_trips`
```
`983648`

- What is the earliest start time and latest end time for a trip?

```sql
#standardSQL
SELECT min(start_date) FROM `bigquery-public-data.san_francisco.bikeshare_trips`
```
`2013-08-29 09:08:00`

```sql
#standardSQL
SELECT max(end_date) FROM `bigquery-public-data.san_francisco.bikeshare_trips`
```
`2016-08-31 23:48:00`

- How many bikes are there?

```sql
#standardSQL
SELECT count(distinct bike_number) FROM `bigquery-public-data.san_francisco.bikeshare_trips`
```
`700`

### Questions of your own
- Make up 3 questions and answer them using the Bay Area Bike Share Trips Data.
- Use the SQL tutorial (https://www.w3schools.com/sql/default.asp) to help you with mechanics.

- Question 1: What are the busiest destination stations?

  * Answer: The top 10 stations are listed below: 

  ```sql
  +-----------+-----------------------------------------------+---------------+
  | num_trips |               end_station_name                |   landmark    |
  +-----------+-----------------------------------------------+---------------+
  |     92014 | San Francisco Caltrain (Townsend at 4th)      | San Francisco |
  |     58713 | San Francisco Caltrain 2 (330 Townsend)       | San Francisco |
  |     50185 | Harry Bridges Plaza (Ferry Building)          | San Francisco |
  |     46197 | Embarcadero at Sansome                        | San Francisco |
  |     44145 | 2nd at Townsend                               | San Francisco |
  |     40956 | Market at Sansome                             | San Francisco |
  |     39598 | Steuart at Market                             | San Francisco |
  |     38545 | Townsend at 7th                               | San Francisco |
  |     35477 | Temporary Transbay Terminal (Howard at Beale) | San Francisco |
  |     26762 | Market at 4th                                 | San Francisco |
  +-----------+-----------------------------------------------+---------------+
  ```
  
  * SQL query:

  ```sql
  #standardSQL
  SELECT
    COUNT(trips.trip_id) AS num_trips,
    trips.end_station_name,
    stations.landmark as landmark
  FROM
    `bigquery-public-data.san_francisco.bikeshare_trips` trips
    INNER JOIN `bigquery-public-data.san_francisco.bikeshare_stations` stations 
      ON trips.end_station_id = stations.station_id
  GROUP BY
    trips.end_station_name,
    stations.landmark
  ORDER BY
    num_trips DESC
  LIMIT
    10
  ```

- Question 2: Is the behavior of subscribers and customers different considering the hours of use?
  
  * Answer: Yes. Peak times for subscribers occur during the early morning and in the late afternoon, while for customers usage is concentrated in the afternoon.

  ```sql
  +-----------------------+----------+
  | num_trips_subscribers | the_hour |
  +-----------------------+----------+
  |                127171 |        8 |
  |                114915 |       17 |
  |                 89546 |        9 |
  |                 76051 |       16 |
  |                 75798 |       18 |
  |                 64946 |        7 |
  |                 35515 |       19 |
  |                 34820 |       15 |
  |                 34532 |       10 |
  |                 34442 |       12 |
  +-----------------------+----------+
  +-----------------------+----------+
  | num_trips_customers   | the_hour |
  +-----------------------+----------+
  |                 12806 |       15 |
  |                 12737 |       14 |
  |                 12719 |       13 |
  |                 12704 |       16 |
  |                 12508 |       12 |
  |                 11387 |       17 |
  |                 11078 |       11 |
  |                  8771 |       18 |
  |                  8250 |       10 |
  |                  6572 |        9 |
  +-----------------------+----------+
  ```

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
    num_trips_subscribers desc
  LIMIT
    10
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
    num_trips_customers desc
  LIMIT
    10
  ```

- Question 3: Does the duration of the trip vary according to different regions and types of customers?
  
  * Answer: Yes. The trip duration for subscribers is shorter and varies only a few minutes for the different regions, while for customers the trip duration is longer and the variation is wider.

  ```sql
  +-------------------+-----------------+---------------+
  | avg_trip_duration | subscriber_type |   landmark    |
  +-------------------+-----------------+---------------+
  |             152.0 | Customer        | Redwood City  |
  |             128.0 | Customer        | Palo Alto     |
  |             123.0 | Customer        | Mountain View |
  |              78.0 | Customer        | San Jose      |
  |              56.0 | Customer        | San Francisco |
  |              15.0 | Subscriber      | Palo Alto     |
  |              12.0 | Subscriber      | Redwood City  |
  |              10.0 | Subscriber      | San Francisco |
  |              10.0 | Subscriber      | San Jose      |
  |               9.0 | Subscriber      | Mountain View |
  +-------------------+-----------------+---------------+
  ```

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
  
