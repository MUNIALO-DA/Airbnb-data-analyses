SELECT * FROM listings;
-- count total listings, distinct hosts, average price by room type
-- just to know the market composition, which room types are common and the pricing offers
select 
room_type, count(*) As total_listings,
round(AVG(price), 2) as avg_price,
round(Avg(number_of_reviews), 2) as avg_reviews
from listings
where price is not null
group by room_type 
order by avg_price desc;

SELECT neighbourhood, COUNT(*) AS number_of_listings
FROM listings
GROUP BY neighbourhood
ORDER BY number_of_listings DESC;

-- top perfomers in the market
-- achieve using Rank(), Dense_Rank to get top listings by price, reviews and popularity in neighbouhoods
-- discovering luxury pockets, highest priced listings, and what makes them unique

with ranked as (
select
neighbourhood, name, price,
Rank() over (partition by neighbourhood order by price desc) as price_rank
from listings
 where price is not null)
 select neighbourhood, name, price
 from ranked 
 where price_rank< 5 order by neighbourhood,price_rank;
select distinct neighbourhood
from listings;
-- host's listing perfomance- comparing reviews of consecutive listings
WITH host_listings AS (
    SELECT 
        host_id,
        host_name,
        name,
        number_of_reviews,
        price,
        ROW_NUMBER() OVER (PARTITION BY host_id ORDER BY number_of_reviews DESC) AS listing_rank
    FROM listings
)
SELECT 
    host_id,
    host_name,
    name,
    number_of_reviews AS current_reviews,
    LAG(number_of_reviews, 1) OVER (PARTITION BY host_id ORDER BY listing_rank) AS prev_reviews,
    price
FROM host_listings
WHERE listing_rank <= 3  -- only top 3 listings per host
ORDER BY host_id, listing_rank;

-- host concentration and superhodt potential
-- which hosts are most active and popular can iform ability for partnership or future recommendations
SELECT 
    host_id,
    host_name,
    COUNT(*) AS num_listings,
    SUM(number_of_reviews) AS total_reviews,
    RANK() OVER (ORDER BY SUM(number_of_reviews) DESC) AS host_rank
FROM listings
GROUP BY host_id, host_name
HAVING COUNT(*) > 1
ORDER BY total_reviews DESC
LIMIT 10;
