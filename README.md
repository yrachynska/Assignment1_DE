# Assignment1_DE

## 1️⃣ Data Loading
```sql
CREATE TABLE raw_yelp_dataset
AS SELECT *
FROM read_json_auto('/Users/gorodok/DataGripProjects/Data_engeniring/Assignment1/yelp_academic_dataset_business.json',
     maximum_object_size = 107108864);
```
## 2️⃣ Data Parsing
```sql
CREATE TABLE yelp_general AS
SELECT
    business_id,
    name,
    address,
    city,
    state,
    postal_code,

    CAST(stars AS DOUBLE) AS stars,
    CAST(review_count AS INTEGER) AS review_count,

    TRY_CAST(json_extract_string(attributes, '$.BikeParking') AS BOOLEAN) AS bike_parking,
    TRY_CAST(json_extract_string(attributes, '$.BusinessAcceptsCreditCards') AS BOOLEAN) AS credit_cards
FROM raw_yelp_dataset;

SELECT * FROM yelp_general LIMIT 5;
```
<img width="1321" height="174" alt="Знімок екрана 2026-02-16 о 11 19 05" src="https://github.com/user-attachments/assets/1fcbd5a3-72a9-4204-a4ba-4c1e0171acc2" />

```sql
CREATE TABLE yelp_categories AS
SELECT
    business_id,
    UNNEST(string_split(categories, ', ')) AS category
FROM raw_yelp_dataset;

SELECT * FROM yelp_categories LIMIT 5;
```
<img width="497" height="174" alt="Знімок екрана 2026-02-16 о 11 22 44" src="https://github.com/user-attachments/assets/2aa99311-0a83-45fd-8a95-48f8de4f9c80" />

```sql
CREATE TABLE yelp_worktime AS
SELECT
    business_id,

    CAST(string_split(json_extract_string(hours, '$.Monday'), '-')[1] AS TIME) AS monday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Monday'), '-')[2] AS TIME) AS monday_close_time,
    CAST(string_split(json_extract_string(hours, '$.Tuesday'), '-')[1] AS TIME) AS tuesday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Tuesday'), '-')[2] AS TIME) AS tuesday_close_time,
    CAST(string_split(json_extract_string(hours, '$.Wednesday'), '-')[1] AS TIME) AS wednesday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Wednesday'), '-')[2] AS TIME) AS wednesday_close_time,
    CAST(string_split(json_extract_string(hours, '$.Thursday'), '-')[1] AS TIME) AS thursday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Thursday'), '-')[2] AS TIME) AS thursday_close_time,
    CAST(string_split(json_extract_string(hours, '$.Friday'), '-')[1] AS TIME) AS friday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Friday'), '-')[2] AS TIME) AS friday_close_time,
    CAST(string_split(json_extract_string(hours, '$.Saturday'), '-')[1] AS TIME) AS saturday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Saturday'), '-')[2] AS TIME) AS saturday_close_time,
    CAST(string_split(json_extract_string(hours, '$.Sunday'), '-')[1] AS TIME) AS sunday_open_time,
    CAST(string_split(json_extract_string(hours, '$.Sunday'), '-')[2] AS TIME) AS sunday_close_time
FROM raw_yelp_dataset;

SELECT * FROM yelp_worktime LIMIT 5;
```
<img width="1321" height="171" alt="Знімок екрана 2026-02-16 о 11 24 21" src="https://github.com/user-attachments/assets/4c14a4cb-9642-461c-8d36-40c59e498351" />

## 3️⃣ Data Analysis Using Window Functions

1. The level of dominance of each business in its city. What share of all customer attention does a particular business take up.
   ```sql
   SELECT
    city,
    name AS business_name,
    review_count,
    SUM(review_count) OVER(PARTITION BY city) AS sum_city_reviews,
    ROUND((review_count * 100.0) / SUM(review_count) OVER(PARTITION BY city), 2) AS review_percent
   FROM yelp_general
   QUALIFY review_percent != 100
   ORDER BY review_percent DESC;
   ```
   <img width="1306" height="245" alt="Знімок екрана 2026-02-16 о 11 52 43" src="https://github.com/user-attachments/assets/f6e4f56d-7289-4760-841f-441af941afd6" />

2. Top 3 categories with the longest working day on Sunday by state.
   ```sql
   WITH work_hours AS (
    SELECT
        yg.state,
        yc.category,
        date_diff('hour', yw.sunday_open_time, yw.sunday_close_time) AS sunday_work_hours
    FROM yelp_categories yc
    JOIN yelp_worktime yw ON yc.business_id = yw.business_id
    JOIN yelp_general yg ON yc.business_id = yg.business_id
   ),
   category_average AS (
    SELECT
        state,
        category,
        ROUND(AVG(sunday_work_hours), 1) AS avg_category_hours,
        COUNT(sunday_work_hours) AS business_count
    FROM work_hours
    GROUP BY state, category
    HAVING business_count >= 2
   )
   SELECT
    state,
    category,
    avg_category_hours,
    business_count,
    DENSE_RANK() OVER(PARTITION BY state ORDER BY avg_category_hours DESC) AS rank_in_region
   FROM category_average
   QUALIFY rank_in_region <= 3
   ORDER BY state, rank_in_region;
   ```
   <img width="1136" height="182" alt="Знімок екрана 2026-02-16 о 12 30 36" src="https://github.com/user-attachments/assets/4a5560cb-d9e8-4dab-b8e3-8042772bc037" />

## 4️⃣ Additional Task
1. Check NULLs
   ```sql
   SELECT
    SUM(CASE WHEN business_id IS NULL THEN 1 ELSE 0 END) AS missing_ids,
    SUM(CASE WHEN name IS NULL THEN 1 ELSE 0 END) AS missing_names,
    SUM(CASE WHEN city IS NULL THEN 1 ELSE 0 END) AS missing_cities,
    SUM(CASE WHEN address IS NULL THEN 1 ELSE 0 END) AS missing_addresses,
    SUM(CASE WHEN state IS NULL THEN 1 ELSE 0 END) AS missing_states,
    SUM(CASE WHEN postal_code IS NULL THEN 1 ELSE 0 END) AS missing_postal_codes,
    SUM(CASE WHEN bike_parking IS NULL THEN 1 ELSE 0 END) AS missing_bike_parking,
    SUM(CASE WHEN credit_cards IS NULL THEN 1 ELSE 0 END) AS missing_credit_cards
   FROM yelp_general;
   ```
   <img width="1036" height="61" alt="Знімок екрана 2026-02-16 о 12 43 05" src="https://github.com/user-attachments/assets/9f54b6d5-142e-4de5-bc8a-75c07a1517b3" />

2. Check duplicates
   ```sql
   SELECT
    business_id,
    COUNT(*) AS count
   FROM yelp_general
   GROUP BY business_id
   HAVING count > 1;
   ```
   <img width="317" height="81" alt="Знімок екрана 2026-02-16 о 12 45 57" src="https://github.com/user-attachments/assets/0d72dead-0813-4f52-98aa-b45f495f58e9" />

3. Schema validation
   ```sql
   SELECT
    COUNT(*) AS count_erors
   FROM yelp_worktime
   WHERE date_diff('hour', monday_open_time, monday_close_time) <= 0;
   
   SELECT
    business_id,
    monday_open_time,
    monday_close_time,
    date_diff('hour', monday_open_time, monday_close_time) AS hours_work
   FROM yelp_worktime
   WHERE hours_work <= 0
   ORDER BY hours_work;
   ```
   <img width="179" height="60" alt="Знімок екрана 2026-02-16 о 13 01 02" src="https://github.com/user-attachments/assets/fe938683-d638-48ab-a56e-fcb83b95cae6" />
   <img width="902" height="186" alt="Знімок екрана 2026-02-16 о 13 01 27" src="https://github.com/user-attachments/assets/c35cf961-bfb5-41fc-9ea1-41de9ee2d0ff" />


