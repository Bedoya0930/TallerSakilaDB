# TallerSakilaDB

### Excercise 1
Insert a record into the 'film' table using dummy values, ensuring referential integrity with other tables.
 ```
INSERT INTO film (title, description, release_year, language_id, rental_duration, rental_rate, length, replacement_cost, rating, special_features, last_update)
VALUES ('THE LIVES OF OTHERS', 'In 1984 East Berlin, an agent of the secret police, conducting surveillance on a writer and his lover, finds himself becoming increasingly absorbed by their lives.', 2006, 1, 4, 2.99, 137, 11.99, 'R', 'Deleted Scenes', NOW());
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/78753585-2b8f-4503-9a3b-c555b2564105)


### Excercise 2
Which films are longer than the average duration of films?
 ```
select title, length
from film
where length > (
    select avg(length)
    from film
);
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/b36d7fcf-9542-4f57-b344-af037342ef71)
 

### Excercise 3
Which films are currently rented at the store with store_id = 1?
 ```
SELECT f.title, i.store_id, r.return_date
FROM film f
JOIN inventory i ON f.film_id = i.film_id
JOIN rental r ON i.inventory_id = r.inventory_id
WHERE i.store_id = 1
AND r.return_date IS NULL; 
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/5ed8ff1e-7080-409b-949b-e0f4163de538)


### Excercise 4
Of the movies at the store with store_id = 1, which ones were rented for a longer duration than the average rental period?
 ```
select f.film_id, f.title, i.store_id
from film f
join inventory i on f.film_id = i.film_id
join rental r on i.inventory_id = r.inventory_id
where i.store_id = 1
and datediff(r.return_date, r.rental_date) > (
    select avg(datediff(return_date, rental_date))
    from rental
    where inventory_id in (
        select inventory_id
        from inventory
        where store_id = 1
    )
)
group by f.film_id, f.title;
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/fc3f64b1-51b6-49da-8cb3-8e4a623778ae)


### Excercise 5
Which actors are part of the cast of 5 or fewer movies?
 ```
SELECT actor_id, first_name, last_name
FROM actor
WHERE actor_id IN (
    SELECT actor_id
    FROM film_actor
    GROUP BY actor_id
    HAVING COUNT(film_id) <= 5
);
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/30269a57-40d1-49f3-a239-9ad139525d24)


### Excercise 6
Which last names do not repeat among different actors?
 ```
SELECT DISTINCT last_name
FROM actor;
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/77c90c5a-c4fb-497d-ab98-674ff5b6793a)


### Excercise 7
Create a view with the top 3 genres that generate the highest revenue. List them in descending order, considering the 'amount' field from the payment table for the calculation.
 ```
create view top_3_revenue_genres as
select fc.category_id, c.name as genre_name, sum(p.amount) as total_revenue
from payment p
join rental r on p.rental_id = r.rental_id
join inventory i on r.inventory_id = i.inventory_id
join film f on i.film_id = f.film_id
join film_category fc on f.film_id = fc.film_id
join category c on fc.category_id = c.category_id
group by fc.category_id, c.name
order by total_revenue desc
limit 3;

SELECT * FROM top_3_revenue_genres;
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/47b922f4-cbc0-4cef-91b5-583176d1e087)


### Excercise 8
Select the top two most-viewed movies in each city.
 ```
SELECT
    city,
    film_id,
    title,
    views
FROM (
    SELECT 
        c.city,
        f.film_id,
        f.title,
        count(*) AS views,
        ROW_NUMBER() OVER (PARTITION BY c.city ORDER BY COUNT(*) DESC) AS row_num
    FROM city c
    JOIN address a on c.city_id = a.city_id
    JOIN customer cu on a.address_id = cu.address_id
    JOIN rental r on cu.customer_id = r.customer_id
    JOIN inventory i on r.inventory_id = i.inventory_id
    JOIN film f on i.film_id = f.film_id
    GROUP BY c.city_id, c.city, f.film_id, f.title
) AS numbered_views
WHERE row_num <= 2
ORDER BY views DESC;
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/74e43867-a8e1-4d62-9724-aab67b8113d5)


### Excercise 9
Select the first name, last name, and email of all customers from the United States who have not made any film rentals in the last three months.
 ```
select c.customer_id, c.first_name, c.last_name, c.email
from customer c
join address a on c.address_id = a.address_id
join city ci on a.city_id = ci.city_id
join country co on ci.country_id = co.country_id
left join rental r on c.customer_id = r.customer_id
where co.country = 'united states' and (r.rental_id is null or r.rental_date < date_sub(now(), interval 3 month))
group by c.customer_id, c.first_name, c.last_name, c.email;
 ``` 
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/6f8f0a73-bb88-40d8-a9cb-642313f25b77)

### Excercise 10
Select the top 3 customers from each store based on the number of rentals made. Utilize the Rank, Dense_Rank, and Row_Number functions, and create an additional boolean field indicating records where these three functions return the same value (0) and records where these three functions do not return the same value (1).
 ```
WITH ranked_customers AS (
    SELECT
        *,
        RANK() OVER (PARTITION BY store_id ORDER BY rental_count DESC) AS rank_value,
        DENSE_RANK() OVER (PARTITION BY store_id ORDER BY rental_count DESC) AS dense_rank_value,
        ROW_NUMBER() OVER (PARTITION BY store_id ORDER BY rental_count DESC) AS row_num
    FROM (
        SELECT
            c.customer_id,
            i.store_id,
            COUNT(r.rental_id) AS rental_count
        FROM customer c
        JOIN rental r ON c.customer_id = r.customer_id
        JOIN inventory i ON r.inventory_id = i.inventory_id
        GROUP BY c.customer_id, i.store_id
    ) AS rental_counts
)
SELECT *,
       CASE
           WHEN rank_value = dense_rank_value AND dense_rank_value = row_num THEN 0
           ELSE 1
       END AS same_rank
FROM ranked_customers
WHERE rank_value <= 3;
 ```
![image](https://github.com/Bedoya0930/TallerSakilaDB/assets/163351947/779881cd-c72a-4d70-9880-13dc79a37506)

