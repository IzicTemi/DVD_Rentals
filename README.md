# DVD Rentals Exploration
The Project involves querying the [Sakila DVD Rental database](https://www.postgresqltutorial.com/postgresql-getting-started/postgresql-sample-database/) to gain an understanding of the customer base, such as what the patterns in movie watching are across different customer groups, how they compare on payment earnings, and how the stores compare in their performance.

The Database ERD Diagram is shown below.

![Alt text](https://www.postgresqltutorial.com/wp-content/uploads/2018/03/dvd-rental-sample-database-diagram.png "ERD Diagram")

Through the analysis, we seek to answer the following questions:

## 1. What are the top 10 cities where family-friendly movies are rented?
Family-friendly movies are movies with categories - Family, Music, Children, Classics, Comedy
```sql
/*this cte represents top 10 cities in terms of the number of rentals of family friendly films*/
WITH 
    top10_rental_cities AS
        (
        SELECT city, SUM(rentals)
        FROM
        (
        SELECT
                city.city as city
                , COUNT(r.rental_id) AS rentals
        FROM
        (
        SELECT 
                f.film_id AS film_id
                , c.name as category_name        
        FROM category c
        JOIN film_category fc
        ON c.category_id = fc.category_id
        JOIN film f
        ON f.film_id = fc.film_id
        ) sub
        JOIN inventory i
        ON i.film_id = sub.film_id
        JOIN rental r
        ON r.inventory_id = i.inventory_id
        JOIN customer cc
        ON cc.customer_id = r.customer_id
        JOIN address ad
        ON ad.address_id = cc.address_id
        JOIN city
        ON city.city_id = ad.city_id

        WHERE category_name='Family' 
                OR category_name='Music' 
                OR category_name='Animation' 
                OR category_name='Children' 
                OR category_name='Classics' 
                OR category_name='Comedy'
        GROUP BY 1
        
        )sub1
        GROUP BY 1
        ORDER BY 2 DESC
        LIMIT 10
        )
        /* this cte represents the number of rentals per city per family friendly category */
    , category_rental_cities AS
        (
        SELECT
                sub.category_name as category_name
                , city.city as city
                , COUNT(r.rental_id) AS rentals
        FROM
        (
        SELECT 
                f.film_id AS film_id
                , c.name as category_name        
        FROM category c
        JOIN film_category fc
        ON c.category_id = fc.category_id
        JOIN film f
        ON f.film_id = fc.film_id
        ) sub
        JOIN inventory i
        ON i.film_id = sub.film_id
        JOIN rental r
        ON r.inventory_id = i.inventory_id
        JOIN customer cc
        ON cc.customer_id = r.customer_id
        JOIN address ad
        ON ad.address_id = cc.address_id
        JOIN city
        ON city.city_id = ad.city_id

        WHERE category_name='Family' 
                OR category_name='Music' 
                OR category_name='Animation' 
                OR category_name='Children' 
                OR category_name='Classics' 
                OR category_name='Comedy'
        GROUP BY 1, 2
        )

SELECT category_rental_cities.*
FROM category_rental_cities
JOIN top10_rental_cities
ON category_rental_cities.city = top10_rental_cities.city
ORDER BY city, category_name;
```
![Top 10 cities](https://github.com/IzicTemi/DVD_Rentals/assets/19413520/a24932a2-db9b-4e72-a85b-60dbeee5f29e)


## 2. How much payment was processed by each staff?
```sql
SELECT
        CONCAT(s.staff_id,'-', s.first_name,' ', s.last_name) AS staff
        , DATE_TRUNC('month', p.payment_date) AS month
        , SUM(p.amount) AS total_amt
FROM staff s
JOIN payment p
ON p.staff_id = s.staff_id
GROUP BY 1, 2
ORDER BY month, total_amt;
```
![Staff revenue](https://github.com/IzicTemi/DVD_Rentals/assets/19413520/dc6c6528-33ec-4de7-ae8b-c8568cf6f06d)

## 3. What are the most revenue generating film markets?
A film market is a sub rating within a category. An example is G-rated Action films or PG-13 rated Comedy.
```sql
SELECT
        c.name AS category_name
        , f.rating AS rating
        , SUM(p.amount) AS total_rental_sales

FROM category c
JOIN film_category fc
ON c.category_id = fc.category_id
JOIN film f
ON f.film_id = fc.film_id
JOIN inventory i
ON i.film_id = f.film_id
JOIN rental r
ON r.inventory_id = i.inventory_id
LEFT JOIN payment p
ON p.rental_id = r.rental_id
GROUP BY 1, 2

ORDER BY category_name, rating;
```
![Film markets](https://github.com/IzicTemi/DVD_Rentals/assets/19413520/7a224747-80b5-485b-b137-2e280571ab36)

## 4. Does rating and category of film affect rental length?
Rental length is the difference between the rental date and return date.
```sql
WITH average_rentals AS
(
SELECT
        category_name
        , rating
        , SUM(number_of_films * rental_length)/SUM(number_of_films) AS avg_rental_length
FROM
(
SELECT DISTINCT
        c.name AS category_name
        , f.rating AS rating
        , DATE_PART('day', return_date - rental_date) AS rental_length
        , COUNT(DATE_PART('day', return_date - rental_date)) OVER (PARTITION BY c.name, f.rating ,DATE_PART('day', return_date - rental_date)) AS number_of_films

FROM category c
JOIN film_category fc
ON c.category_id = fc.category_id
JOIN film f
ON f.film_id = fc.film_id
JOIN inventory i
ON i.film_id = f.film_id
JOIN rental r
ON r.inventory_id = i.inventory_id
ORDER BY 1, 2
) movie_distribution

GROUP BY 1, 2
)
SELECT 
        category_name
        , rating
        , ROUND(avg_rental_length::numeric, 2) AS avg_rental_length

FROM average_rentals

ORDER BY category_name, rating;
```
![Film ratings](https://github.com/IzicTemi/DVD_Rentals/assets/19413520/b9c38f67-7782-4bd2-90a6-428bb58fbf36)
