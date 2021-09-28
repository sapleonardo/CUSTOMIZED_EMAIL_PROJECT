# Customer Email Template w/ Data With Danny # 

![Email](https://user-images.githubusercontent.com/85455439/134821950-65dadbdd-9135-4ab6-b549-9f3bdcf44055.png)

## What is the goal of this project and analysis? ## 
###### Query and return the required information needed for the DVD Rental Co Marketing Team to launch their individualized customer emails ###### 
###### Reveal insights about each customers viewing behavior ######
###### Use individualized customer emails to drive sales and increase engagement from the DVD Rental Co Marketing Team Customer Base ######
###### Based upon analysis, generate first and second category insights for each customer #######

## What information is required to generate each individualized email for DVD Rental Co customers ? What other elements will be in the email? ##
###### Each customers top two film categories and their percentile in terms of views when compared to all other viewers of that film category #######
###### Two seperate groups of customer insights based off of analyses findings #######
###### Top three reccomendations of films based upon what each customers top two categories are ###### 
###### The number of films watched by each customer which had a specific actor ###### 
###### Additional reccomendations for films with each customers top actor ###### 

## What is the difference between group one and group two customer insights?##
###### Customer Insights Group One: Number of films customer has rented in one of their top categories, how much more that is than the average renter of that category
###### Each customers percentile in terms of rentals in each film category #######
###### Custopmer Insights Group Two: The number of films rented in the second top category, the total proportion when compared to each customers total rentals #######

# EMAIL TEMPLATE DATA PART A: CATEGORY INSIGHTS #
## What information will be provided by this category of analysis?##
###### The number of films each customer rented per category #######
###### The total number of films each customer rented in total ######
###### The top two categories of each customer #######
###### The average number of rentals per category for every customer #######
###### Each customers overall percentile when compared against every other rental of that category #######

## Category Insights: Baseline Data ## 

```sql
CREATE TEMP TABLE complete_joint_dataset AS 
SELECT 
  rental.customer_id, 
  inventory.film_id, 
  film.title, 
  category.name AS category_name, 
  rental.rental_date 
FROM dvd_rentals.rental 
INNER JOIN dvd_rentals.inventory 
  ON rental.inventory_id = inventory.inventory_id 
INNER JOIN dvd_rentals.film 
  ON inventory.film_id = film.film_id 
INNER JOIN dvd_rentals.film_category 
  ON film.film_id = film_category.film_id 
INNER JOIN dvd_rentals.category 
  ON film_category.category_id = category.category_id;
  ```
  
  ```sql
  SELECT * FROM complete_joint_dataset; 
  ```
  | customer\_id | film\_id | title           | category\_name | rental\_date             |
| ------------ | -------- | --------------- | -------------- | ------------------------ |
| 130          | 80       | BLANKET BEVERLY | Family         | 2005-05-24T22:53:30.000Z |
| 459          | 333      | FREAKY POCUS    | Music          | 2005-05-24T22:54:33.000Z |
| 408          | 373      | GRADUATE LORD   | Children       | 2005-05-24T23:03:39.000Z |

###### The baseline data includes an INNER JOIN of four seperate tables: rental, inventory, film, category and rental ###### 
###### The data provided by the baseline table includes the customer_id, the film_id, the film title with the category and the accompanying rental_date ###### 
###### Returns a basic list of each customers rental and the accompanying date #######
###### Ex: Customer_130 rented the family film Blanket Beverly on May 5th 2005 ###### 

## Rental COUNTS Per Category ## 

```sql
CREATE TEMP TABLE category_counts AS 
SELECT 
  customer_id, 
  category_name, 
  COUNT(*) AS rental_count, 
  MAX(rental_date) AS latest_rental_date 
FROM complete_joint_dataset 
GROUP BY 
  customer_id, 
  category_name;
  ```
 
 ```sql
 SELECT * FROM category_counts;
 ```
 | customer\_id | category\_name | rental\_count | latest\_rental\_date     |
| ------------ | -------------- | ------------- | ------------------------ |
| 538          | Comedy         | 1             | 2005-08-22T21:01:25.000Z |
| 314          | Comedy         | 3             | 2005-08-23T21:37:59.000Z |
| 286          | Travel         | 1             | 2005-08-18T18:58:35.000Z |
 ###### Table category_counts contains a rental count of every film category along with the latest rental date ######
 ###### Ex: Customer_id 538 rented one comedy film on August 22nd ######
 
 ## Total customer films watched ## 
 
 ```sql
 CREATE TEMP TABLE total_counts AS 
SELECT 
  customer_id, 
  SUM(rental_count) AS total_count 
FROM category_counts 
GROUP BY 
  customer_id; 
```

```sql
SELECT * FROM total_counts;
```
| customer\_id | total\_count |
| ------------ | ------------ |
| 184          | 23           |
| 87           | 30           |
| 477          | 22           |
###### The above code returns a total count of every rental watched per customer #######
###### Ex: Customer_id 87 has watched or rented a total of 30 films ######

## Top Two Categories Per Customer ## 

```sql
CREATE TEMP TABLE top_categories AS 
WITH ranked_cte AS (
  SELECT 
    customer_id, 
    category_name, 
    rental_count, 
    DENSE_RANK() OVER (
      PARTITION BY customer_id 
      ORDER BY 
        rental_count DESC, 
        latest_rental_date DESC, 
        category_name
      ) AS category_rank
    FROM category_counts
  )
  SELECT * FROM ranked_cte 
  WHERE category_rank <= 2; 
```

```sql
SELECT * FROM top_categories; 
``` 
| customer\_id | category\_name | rental\_count | category\_rank |
| ------------ | -------------- | ------------- | -------------- |
| 1            | Classics       | 6             | 1              |
| 1            | Comedy         | 5             | 2              |
| 2            | Sports         | 5             | 1              |
| 2            | Classics       | 4             | 2              |
###### The above code returns an assigned category ranking of 1 or 2 for each customer depending upon their respective rental counts per category name ####### 
###### Ex: Customer_id 2 top two categories are Classics and Comedy having rented 6 Classics movies and 5 Comedy movies ######

## Average rental count per category ## 

```sql
CREATE TEMP TABLE average_category_count AS 
SELECT 
  category_name, 
  FLOOR(AVG(rental_count)) AS category_average 
FROM category_counts 
GROUP BY category_name; 
```

```sql
SELECT * FROM average_category_count; 
```
| category\_name | category\_average |
| -------------- | ----------------- |
| Sports         | 2                 |
| Classics       | 2                 |
| New            | 2                 |
| Family         | 2                 |
###### The above code returns a table displaying the names of film categories and the average rentals/views films of that category ########
###### When taking into account all customers across the database in question #######
###### Ex: Sports movies have been rented on average 2 times #######

## Customer Percentiles Per Top Two Categories ##

```sql
CREATE TEMP TABLE top_category_percentile AS 
WITH calculated_cte AS (
SELECT 
  top_categories.customer_id, 
  top_categories.category_name AS top_category_name,
  top_categories.rental_count, 
  category_counts.category_name,
  top_categories.category_rank,
  PERCENT_RANK() OVER (
    PARTITION BY category_counts.category_name
    ORDER BY category_counts.rental_count DESC
    ) AS raw_percentile_value
  FROM category_counts
  LEFT JOIN top_categories 
    ON category_counts.customer_id = top_categories.customer_id
  )
  SELECT 
    customer_id, 
    category_name, 
    rental_count, 
    category_rank, 
    CASE 
      WHEN ROUND(100 * raw_percentile_value) = 0 THEN 1
      ELSE ROUND(100 * raw_percentile_value)
    END AS percentile 
  FROM calculated_cte 
  WHERE 
    category_rank = 1 
    AND top_category_name = category_name; 
```

```sql
SELECT * FROM top_category_percentile; 
```
| customer\_id | category\_name | rental\_count | category\_rank | percentile |
| ------------ | -------------- | ------------- | -------------- | ---------- |
| 323          | Action         | 7             | 1              | 1          |
| 506          | Action         | 7             | 1              | 1          |
| 151          | Action         | 6             | 1              | 1          |
| 410          | Action         | 6             | 1              | 1          |
###### The above code and returning table showcase the percentile of each customer based upon the number of views within each category ######
###### Ex: Customer_id 323 has rented a total of 7 Action movies. This makes it their most popular category and they are in 1st percentile of renters #######

## Customer Insights Group A

```sql
CREATE TEMP TABLE first_category_insights AS 
SELECT 
  base.customer_id, 
  base.category_name, 
  base.rental_count, 
  base.rental_count - average.category_average AS average_comparison, 
  base.percentile
FROM top_category_percentile AS base 
LEFT JOIN average_category_count AS average 
  ON base.category_name = average.category_name; 
```

```sql
SELECT * FROM first_category_insights; 
```
| customer\_id | category\_name | rental\_count | average\_comparison | percentile |
| ------------ | -------------- | ------------- | ------------------- | ---------- |
| 323          | Action         | 7             | 5                   | 1          |
| 506          | Action         | 7             | 5                   | 1          |
| 151          | Action         | 6             | 4                   | 1          |
| 410          | Action         | 6             | 4                   | 1          |
###### Above code and table showcases the first group of customer insights #######
###### First group of insights compares the customers rental_count to the category average 
###### As well as the percentile that customer is in regarding views of that category of film 
###### Ex: Customer_id 323 has watched a total of 7 Action films. This categories average is 5 rentals. Putting this customer in the 1st percentile ####### 

## Customer Insights Group B 

```sql
CREATE TEMP TABLE second_category_insights AS 
SELECT 
  top_categories.customer_id, 
  top_categories.category_name, 
  top_categories.rental_count, 
  ROUND(
    100 * top_categories.rental_count::NUMERIC / total_counts.total_count
    ) AS total_percentage 
  FROM top_categories 
  LEFT JOIN total_counts 
    ON top_categories.customer_id = total_counts.customer_id 
  WHERE category_rank = 2; 
```

```sql
SELECT * FROM second_category_insights; 
``` 
| customer\_id | category\_name | rental\_count | total\_percentage |
| ------------ | -------------- | ------------- | ----------------- |
| 184          | Drama          | 3             | 13                |
| 87           | Sci-Fi         | 3             | 10                |
| 477          | Travel         | 3             | 14                |
| 273          | New            | 4             | 11                |
###### Above code and table returns the second group of customer insights 
###### The second group of customer insights includes the total percentage each customers rental_count is when compared to the total films they've watched 
###### Ex: Customer_id 184 has watched a total of 3 Drama films. This makes up %13 of their entire rental history #######

# EMAIL_TEMPLATE DATA PART B: Category Reccomendations # 
## What information will be provided by this category of analysis? ##
###### Top three recommendations of film categories based off of each customers rental history #######


## Total Film COUNTS ## 

```sql
CREATE TEMP TABLE film_counts AS 
SELECT DISTINCT 
  film_id, 
  title, 
  category_name, 
  COUNT(*) OVER (
    PARTITION BY film_id 
  ) AS rental_count 
FROM complete_joint_dataset; 
```

```sql
SELECT * FROM film_counts; 
```
| film\_id | title            | category\_name | rental\_count |
| -------- | ---------------- | -------------- | ------------- |
| 655      | PANTHER REDS     | Sci-Fi         | 15            |
| 285      | ENGLISH BULWORTH | Sci-Fi         | 30            |
| 258      | DRUMS DYNAMITE   | Horror         | 13            |
| 809      | SLIPPER FIDELITY | Sports         | 16            |
###### The above code and returned table generates the total COUNT of rentals per film title along with their category name #######
###### Showcase the film title and category of film the customer has watched along with the number of times they have rented it #######
###### Ex: The Sci-Fi film Panther Reds has been rented a total of 15 times ######


## Film Exclusions ##

```sql
CREATE TEMP TABLE category_film_exclusions AS 
SELECT DISTINCT 
  customer_id, 
  film_id
FROM complete_joint_dataset;
```

```sql
SELECT * FROM category_film_exclusions; 
```
| customer\_id | film\_id |
| ------------ | -------- |
| 596          | 103      |
| 176          | 121      |
| 459          | 724      |
| 375          | 641      |
###### The above code returns the table with the titles of films each customer has already watched #######
###### The films which the customer has already watched will not be included in their future film reccomendations #######
###### Ex: Customer_id 596 has already rented film_id 103. This film will not be included in this customers category recommendations. ####### 


## Final Category Reccomendations ## 

```sql
CREATE TEMP TABLE category_recommendations AS 
WITH ranked_films_cte AS (
  SELECT 
    top_categories.customer_id, 
    top_categories.category_name,
    top_categories.category_rank,
    film_counts.film_id,
    film_counts.title,
    film_counts.rental_count,
    DENSE_RANK() OVER (
      PARTITION BY
        top_categories.customer_id,
        top_categories.category_rank
      ORDER BY 
        film_counts.rental_count DESC,
        film_counts.title
      ) AS reco_rank
    FROM top_categories
    INNER JOIN film_counts 
      ON top_categories.category_name = film_counts.category_name
    WHERE NOT EXISTS (
      SELECT 1
      FROM category_film_exclusions
      WHERE 
        category_film_exclusions.customer_id = top_categories.customer_id AND 
        category_film_exclusions.film_id = film_counts.film_id
        )
      )
      SELECT * FROM ranked_films_cte
      WHERE reco_rank <= 3;
      ```
```sql
SELECT * FROM category_recommendations; 
 ```
 | customer\_id | category\_name | category\_rank | film\_id | title          | rental\_count | reco\_rank |
| ------------ | -------------- | -------------- | -------- | -------------- | ------------- | ---------- |
| 1            | Classics       | 1              | 891      | TIMBERLAND SKY | 31            | 1          |
| 1            | Classics       | 1              | 358      | GILMORE BOILED | 28            | 2          |
| 1            | Classics       | 1              | 951      | VOYAGE LEGALLY | 28            | 3          |
| 1            | Comedy         | 2              | 1000     | ZORRO ARK      | 31            | 1          |
###### The code above returns table category recommendations ######
###### This table returns the top three movie categories w/titles recommended based off of the customers rental history ######
###### Ex: Customer_id top three film recommendations are all Classics ######

# EMAIL_TEMPLATE_DATA PART C: Actor Insights # 
## What information will be provided by this category of analysis? ##
###### The number of movies each customer has rented with a specific starring actor ######
###### Top three film recommendations starring each customers favorite actor ######

## Actor: Baseline Data ##
```sql
CREATE TEMP TABLE actor_joint_dataset AS
SELECT
  rental.customer_id,
  rental.rental_id,
  rental.rental_date,
  film.film_id,
  film.title,
  actor.actor_id,
  actor.first_name,
  actor.last_name
FROM dvd_rentals.rental
INNER JOIN dvd_rentals.inventory
  ON rental.inventory_id = inventory.inventory_id
INNER JOIN dvd_rentals.film
  ON inventory.film_id = film.film_id
INNER JOIN dvd_rentals.film_actor
  ON film.film_id = film_actor.film_id
INNER JOIN dvd_rentals.actor
  ON film_actor.actor_id = actor.actor_id;
```

```sql
SELECT * FROM actor_joint_dataset; 
```
| customer\_id | rental\_id | rental\_date             | film\_id | title           | actor\_id | first\_name | last\_name |
| ------------ | ---------- | ------------------------ | -------- | --------------- | --------- | ----------- | ---------- |
| 130          | 1          | 2005-05-24T22:53:30.000Z | 80       | BLANKET BEVERLY | 200       | THORA       | TEMPLE     |
| 130          | 1          | 2005-05-24T22:53:30.000Z | 80       | BLANKET BEVERLY | 193       | BURT        | TEMPLE     |
| 130          | 1          | 2005-05-24T22:53:30.000Z | 80       | BLANKET BEVERLY | 173       | ALAN        | DREYFUSS   |
| 130          | 1          | 2005-05-24T22:53:30.000Z | 80       | BLANKET BEVERLY | 16        | FRED        | COSTNER    |
###### The following code returns the table actor_joint_dataset which returns the titles of films customers have watched with the first and last names of actors ######
###### Customer_id 130 rented a film with Alan Dreyfuss on May 24th 2005 ######

## Actor COUNTS ## 

```sql
CREATE TEMP TABLE top_actor_counts AS 
WITH actor_counts AS (
  SELECT 
    customer_id,
    actor_id,
    first_name,
    last_name,
    COUNT(*) AS rental_count,
    MAX(rental_date) AS latest_rental_date
  FROM actor_joint_dataset
  GROUP BY 
    customer_id,
    actor_id,
    first_name,
    last_name
  ),
  ranked_actor_counts AS (
    SELECT 
      actor_counts.*,
      DENSE_RANK() OVER (
        PARTITION BY customer_id
        ORDER BY 
          rental_count DESC,
          latest_rental_date DESC,
          first_name,
          last_name
        ) AS actor_rank
      FROM actor_counts
      )
      SELECT 
        customer_id,
        actor_id,
        first_name, 
        last_name,
        rental_count
      FROM ranked_actor_counts 
      WHERE actor_rank = 1;
```

```sql
SELECT * FROM top_actor_counts; 
```
| customer\_id | actor\_id | first\_name | last\_name | rental\_count |
| ------------ | --------- | ----------- | ---------- | ------------- |
| 1            | 37        | VAL         | BOLGER     | 6             |
| 2            | 107       | GINA        | DEGENERES  | 5             |
| 3            | 150       | JAYNE       | NOLTE      | 4             |
| 4            | 102       | WALTER      | TORN       | 4             |
###### The following code returns the table top_actor_counts which shows the number of times each customer has rented a film starring a specific actor #######
###### Ex: Customer_id 4 has rented 4 films starring Walter Torn ######

## Official Actor Recommendations ##

```sql
CREATE TEMP TABLE actor_recommendations AS
WITH ranked_actor_films_cte AS (
  SELECT
    top_actor_counts.customer_id,
    top_actor_counts.first_name,
    top_actor_counts.last_name,
    top_actor_counts.rental_count,
    actor_film_counts.title,
    actor_film_counts.film_id,
    actor_film_counts.actor_id,
    DENSE_RANK() OVER (
      PARTITION BY
        top_actor_counts.customer_id
      ORDER BY
        actor_film_counts.rental_count DESC,
        actor_film_counts.title
    ) AS reco_rank
  FROM top_actor_counts
  INNER JOIN actor_film_counts
    ON top_actor_counts.actor_id = actor_film_counts.actor_id
WHERE NOT EXISTS (
    SELECT 1
    FROM actor_film_exclusions
    WHERE
      actor_film_exclusions.customer_id = top_actor_counts.customer_id AND
      actor_film_exclusions.film_id = actor_film_counts.film_id
  )
)
SELECT * FROM ranked_actor_films_cte
WHERE reco_rank <= 3;
```

```sql
SELECT * FROM actor_recommendations; 
```
| customer\_id | first\_name | last\_name | rental\_count | title             | film\_id | actor\_id | reco\_rank |
| ------------ | ----------- | ---------- | ------------- | ----------------- | -------- | --------- | ---------- |
| 1            | VAL         | BOLGER     | 6             | PRIMARY GLASS     | 697      | 37        | 1          |
| 1            | VAL         | BOLGER     | 6             | ALASKA PHANTOM    | 12       | 37        | 2          |
| 1            | VAL         | BOLGER     | 6             | METROPOLIS COMA   | 572      | 37        | 3          |
| 2            | GINA        | DEGENERES  | 5             | GOODFELLAS SALUTE | 369      | 107       | 1          |
###### The above code returns the table actor_recommendations which provides three recommended films for each customer with their favorite actor ######
###### Ex: Customer_id 1 has three film recommendations starring Val Bolger ####### 


