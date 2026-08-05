/* 

Query 1 - Query used for Slide 1

Question: How much more (or less) did Store 1 generate in rental revenue versus Store 2, in each film category? 

*/

/* table1 creates a table of every combination of film category and store_id.  It then uses an aggregation to create a column of the total rental revenue (sum of payment.amount) for each film category and store_id combination.  It then uses a Window Function to create a column to help compare store1 revenue versus store2 revenue for each film category.  Lastly, it creates a column representing the difference in store1 versus store2 revenue for each film category and store_id combination. */
WITH table1 AS (
SELECT c.name AS category_name,
       i.store_id,
       SUM (p.amount) AS rental_revenue,
       LEAD (SUM (p.amount)) OVER (PARTITION BY c.name ORDER BY i.store_id) AS next_store_rental_revenue,
       (SUM (p.amount)) - (LEAD (SUM (p.amount)) OVER (PARTITION BY c.name ORDER BY i.store_id)) AS store_rental_revenue_difference
  FROM inventory i
  JOIN rental r
    ON i.inventory_id = r.inventory_id
  JOIN payment p
    ON r.rental_id = p.rental_id
  JOIN film f
    ON f.film_id = i.film_id
  JOIN film_category fc
    ON fc.film_id = f.film_id
  JOIN category c
    ON fc.category_id = c.category_id
 GROUP BY 1, 2
 ORDER BY 1, 2
)
                                      
/* the output table cleans up table1 so that you are left with a column of category names and a second column representing the difference in rental revenue between store1 and store2 for each category.  A positive difference in rental revenue represents a category where store1 outperformed store2.  A negative difference in rental revenue represents a category where store1 underperformed store2. */
SELECT category_name,
       store_rental_revenue_difference
   FROM table1
WHERE store_rental_revenue_difference IS NOT NULL
ORDER BY 2 DESC;


/* 

Query 2 - Query used for Slide 2

Question: For each country where customers live, aggregate total rental payments received and divide by total rental payments owed to determine the "National Average Overpayment Rate." Which countries are in the top 10% of countries by National Average Overpayment Rate, and what is their rate of overpayment? 

*/

/* table1 creates a table of every rental_id, the country where the customer lives, the amount due, and the amount paid for the rental.  Running a couple initial queries determines that payment amounts do not necessarily correlate with the rental_rate for a given film.  In some cases, payment was made but it was more or less than the rental_rate.  In some cases, a payment was made but it was $0.00.  In one case, multiple payments were made for a single rental (rental_id 4591 appears in 5 different records with unique payments_id's).  Since there are some cases where no payment was made (a rental_id does not appear in any record in the payment database), the payment database needs to be LEFT JOINed to capture those records from the rental database that do not have a rental_id in the payment database.  The CASE WHEN statement is used to give the resulting records with a NULL value for p.amount a value so that they can be used to calculate balances in table2 */

WITH table1 AS (
SELECT r.rental_id,
       c3.country,
       f.rental_rate AS amount_due,
       CASE WHEN p.amount IS NULL THEN '0.00'
            ELSE p.amount END AS amount_paid
  FROM customer c
  JOIN rental r
    ON c.customer_id = r.customer_id
  LEFT JOIN payment p
    ON p.rental_id = r.rental_id
  JOIN inventory i
    ON i.inventory_id = r.inventory_id
  JOIN film f
    ON f.film_id = i.film_id
  JOIN address a
    ON c.address_id = a.address_id
  JOIN city c2
    ON a.city_id = c2.city_id
  JOIN country c3
    ON c2.country_id = c3.country_id
 ORDER BY 2
),

/* table2 sums the amount paid, amount due, and divides the two to determine the National Average Overpayment Rate, grouping by country, ordering from highest to lowest by National Average Overpayment Rate. */
table2 AS (
SELECT country,
       SUM (amount_paid) AS total_amount_paid,
       SUM (amount_due) AS total_amount_due,
       (SUM (amount_paid) / SUM (amount_due)) - 1 AS national_average_overpayment_rate
  FROM table1
 GROUP BY 1
 ORDER BY 4 DESC
),

/* table3 uses a Window Function to determine the top 10% of countries by Country-Wide Overpayment Rate, including only countries that have overpaid (some countries have underpaid). */
table3 AS (
SELECT country,
       national_average_overpayment_rate,
       NTILE (10) OVER (ORDER BY national_average_overpayment_rate DESC) AS national_average_overpayment_rate_decile
  FROM table2
 WHERE national_average_overpayment_rate > 0
)

/* The final output shows the country and overpayment rate of only countries in the top 10%. */
SELECT country,
       national_average_overpayment_rate
  FROM table3
 WHERE national_average_overpayment_rate_decile = 1;
       

/* 

Query 3 - Query used for Slide 3

Question: What percent of each country's total rentals are represented by films in the shortest quartile (by length).  Show only the top 10 countries.

*/

/* table1 creates table of each rental_id, the country of its customer, its film_id, and whether that film was in the shortest quartile (by length), or not.  It uses a Window Function to determine films in the shortest quartile, and then uses as CASE WHEN statement to put a '1' in column ‘short’ if the film is in the shortest quartile.  Otherwise, it leaves the column null. */
WITH table1 AS (
SELECT country,
       rental_id,
       f.film_id,
       CASE WHEN i.film_id IN (

/* This subquery creates a table of film_id’s for the shortest quartile of films (by length) */
                               SELECT film_id
                                 FROM (
                                       SELECT f.film_id,
                                              f.length,
                                              NTILE (4) OVER (ORDER BY f.length) AS length_quartile
                                         FROM film f
                                        )sub1
                                 WHERE length_quartile = 1
                               ) THEN '1' END AS short
                               
  FROM country c
  JOIN city c2
    ON c.country_id = c2.country_id
  JOIN address a
    ON c2.city_id = a.city_id
  JOIN customer c3
    ON a.address_id = c3.address_id
  JOIN rental r
    ON c3.customer_id = r.customer_id
  JOIN inventory i
    ON r.inventory_id = i.inventory_id
  JOIN film f
    ON i.film_id = f.film_id
 ORDER BY 1
)
 
/* The output table uses an aggregation to group table1 by country, sums the number of rentals in the shortest quartile, sums the number of total rentals, and calculates the ratio of rentals in the shortest quartile to total rentals.  It is ordered by this ratio, from highest to lowest, showing only the top 10 countries. */
SELECT country,
       COUNT (short) AS short_count,
       COUNT (rental_id) AS rental_count,
       CAST (COUNT (short) as FLOAT) / (COUNT (rental_id)) * 100 AS percent_of_rentals_in_shortest_quartile
  FROM table1
 GROUP BY 1
 ORDER BY 4 DESC
 LIMIT 10;
                                                               

/* 

Query 4 - Query used for Slide 4

Question: Identify customers with a preference for films with a female protagonist ("Female Protagonist Positive").  Next, divide the customer base (including all customers, not just ones identified as Female Protagonist Positive) into quartiles, based on each customer's total rental payments made ("Customer Rental Revenue").  Within each quartile, how much total Customer Rental Revenue is represented by customers identified as Female Protagonist Positive versus all other customers?

*/ 

/* table1 is a list of every film with " Woman " or " Girl " in the film's description, before the word "who".  This means the film has at least one female protagonist.  If the only occurrence of " Woman " or " Girl " in the description is after the word "who", the film has been excluded, since the " Woman " or " Girl " is not the protagonist (e.g. inventory_id 115).  Leading and trailing spaces have been included in " Woman " and " Girl " to exclude any occurrences of those words within other words (e.g. "Womanizer" is the protagonist in inventory_id 260).  Note, while a womanizer could be a woman, we don't know they are female for certain, in the same way that a scientist could be female but we don't know for certain, one way or the other, and will thus not include "scientist" in the list.  Similarly, "Feminist" might not necessarily be a female, so we are not including "Feminist" in the list). */

WITH table1 AS (
SELECT f.film_id  
  FROM film f
 WHERE (
        STRPOS (f.description, ' Woman ') > 0 AND 
        STRPOS (f.description, ' Woman ') < STRPOS (f.description, ' who ')
       ) OR 
       (
        STRPOS (f.description, ' Girl ') > 0 AND 
        STRPOS (f.description, ' Girl ') < STRPOS (f.description, ' who ')
       )
),

/* table2 is a list of customer_id's of customers who have rented films with a female protagonist and how many times they have rented such films. */

table2 AS (
SELECT c.customer_id,
       COUNT (r.rental_id) AS female_protagonist_rental_count
  FROM rental r
  JOIN customer c
    ON r.customer_id = c.customer_id
  JOIN inventory i
    ON i.inventory_id = r.inventory_id
  JOIN film f
    ON f.film_id = i.film_id
 WHERE i.film_id IN (
                          SELECT *
                            FROM table1
                         )
 GROUP BY 1
 ORDER BY 1, 2 
),

/* table3 is a list of all customer_id's and how many total films each has rented. */

table3 AS (
SELECT c.customer_id,
       COUNT (r.rental_id) AS total_rental_count
  FROM rental r
  JOIN customer c
    ON r.customer_id = c.customer_id
 GROUP BY 1
 ORDER BY 1, 2 
),

/* table4 joins table2 and table3 to create a list of customer_id's of customers who have rented films with a female protagonist AND whose ratio of how many times they have rented such films to how many total films they have rented is greater than the ratio of films with a female protagonist to all films in the database.  This is a list of customers with a preference for films with a female protagonist since they have rented films with a female protagonist at a higher rate than would result from a random renting of films.  Note, I have deliberately chosen to use the ratio of film_id's rather than ratio of inventory_id's as a comparison to each customer's rental history, under the assumption that in general, having a greater number of copies of a certain film will not make it more likely to be rented. */

table4 AS (
SELECT t2.customer_id
  FROM table2 t2
  JOIN table3 t3
    ON t2.customer_id = t3.customer_id
 WHERE (CAST (t2.female_protagonist_rental_count AS FLOAT) / t3.total_rental_count) > 
       (CAST (
             (
              SELECT COUNT (*)
                FROM table1
             ) AS FLOAT) /
             ( 
              SELECT COUNT (f.film_id)
                FROM film f
             )
        )
),

/* table5 creates a table of every customer's customer_id, total amount of revenue they have generated through rental payments ("Customer Total Rental Revenue"), uses a Window Function to determine what Customer Total Rental Revenue quartile they are in, and whether they have rented films with a female protagonist at a higher rate than would result from a random renting of films ("Female Protagonist Positive") or not ("No Preference"). */


table5 AS (
SELECT c.customer_id,
       SUM (p.amount) AS customer_total_rental_revenue,
       NTILE (4) OVER (ORDER BY SUM (p.amount) DESC) AS customer_total_rental_revenue_quartile,
       CASE WHEN c.customer_id IN (
                                   SELECT *
                                    FROM table4
                                   ) THEN 'female protagonist positive'
            ELSE 'no preference' END AS female_protagonist_renter_profile
  FROM customer c
  JOIN payment p
    ON c.customer_id = p.customer_id
 GROUP BY 1
),

/* table6 shows the quartiles of customers based on Customer Total Rental Revenue.  Within each quartile, Customer Total Rental Revenue is summed for each of two categories of customers: (1) Female Protagonist Positive and (2) No Preference.  Two additional columns are created to help format the final exported table for Excel */

table6 AS (
SELECT customer_total_rental_revenue_quartile,
       female_protagonist_renter_profile,
       SUM (customer_total_rental_revenue) AS customer_total_rental_revenue,
       CONCAT ('$', LAG (SUM (customer_total_rental_revenue)) OVER (PARTITION BY customer_total_rental_revenue_quartile ORDER BY female_protagonist_renter_profile)) AS female_protagonist_positive,
       CASE WHEN female_protagonist_renter_profile  = 'no preference' THEN CONCAT ('$', SUM (customer_total_rental_revenue))
            ELSE NULL END AS no_preference
  FROM table5
 GROUP BY 1, 2
 ORDER BY 1, 2
)
       
/* the final output reformats table6 for Excel */
                                                                
SELECT customer_total_rental_revenue_quartile,
       female_protagonist_positive,
       no_preference
  FROM table6
 WHERE no_preference IS NOT NULL;