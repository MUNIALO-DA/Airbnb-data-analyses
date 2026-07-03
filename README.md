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


SELECT 
    CASE 
        WHEN number_of_reviews < 20 AND reviews_per_month > 3 THEN 'New & Popular'
        WHEN number_of_reviews >= 50 THEN 'Established'
        ELSE 'Average / Other'
    END AS review_category,
    COUNT(*) AS listing_count,
    AVG(price) AS avg_price,
    AVG(reviews_per_month) AS avg_reviews_per_month
FROM listings
WHERE reviews_per_month IS NOT NULL AND price IS NOT NULL
GROUP BY review_category
ORDER BY avg_reviews_per_month DESC;
ORDER BY total_reviews DESC
LIMIT 10;


SELECT * FROM airbnb_analysis.listings;
SELECT * FROM airbnb_analysis.neighbourhoods;
SELECT * FROM airbnb_analysis.reviews;
-- just a general overview
select count(*) as total_listings from listings;
select count(*) as total_neighbourhoods from neighbourhoods;
select count(*) as total_reviews from reviews;
select count(distinct host_id) as unique_hosts from listings;

-- pricing analysis
select room_type,
avg(price) as avgprice,
count(*) as num_listings
from listings
where price is not null
group by room_type
order by avgprice desc;


SELECT
    price_decile,
    COUNT(*) AS count,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM (
    SELECT
        price,
        NTILE(10) OVER (ORDER BY price) AS price_decile
    FROM listings
    WHERE price IS NOT NULL
) AS ranked_prices
GROUP BY price_decile
ORDER BY price_decile;

-- Neighbourhood Analysis (Joining with neighbourhoods)
-- Listings per neighbourhood_group

-- Average price per neighbourhood (and per group)

-- Highest / lowest priced neighbourhoods

SELECT 
    n.neighbourhood_group,
    n.neighbourhood,
    COUNT(l.id) AS listing_count,
    AVG(l.price) AS avg_price
FROM neighbourhoods n
LEFT JOIN listings l ON n.neighbourhood = l.neighbourhood
WHERE l.price IS NOT NULL
GROUP BY n.neighbourhood_group, n.neighbourhood
ORDER BY avg_price DESC;

-- . Host Analysis
-- Hosts with the most listings
-- Superhosts (if we could infer from data – maybe not directly, but we can look at hosts with many listings and high review counts)
-- Average reviews per host

SELECT 
    host_id,
    host_name,
    COUNT(*) AS listing_count,
    SUM(number_of_reviews) AS total_reviews,
    AVG(number_of_reviews) AS avg_reviews_per_listing
FROM listings
GROUP BY host_id, host_name
ORDER BY listing_count DESC
LIMIT 10;


-- Availability and Occupancy
-- Average availability_365 per room type
-- Listings that are available year‑round (availability_365 = 365) vs. never available (0)

SELECT 
    room_type,
    AVG(availability_365) AS avg_availability,
    COUNT(CASE WHEN availability_365 = 365 THEN 1 END) AS always_available,
    COUNT(CASE WHEN availability_365 = 0 THEN 1 END) AS never_available
FROM listings
GROUP BY room_type;

--  Review Trends & Sentiment (Proxies)
-- Number of reviews per month (reviews_per_month) – average by neighbourhood

-- Listings with no reviews

-- Most reviewed listings

-- Seasonality: count of last_review by month


-- Average reviews per month by neighbourhood group
SELECT 
    neighbourhood_group,
    AVG(reviews_per_month) AS avg_reviews_per_month
FROM listings
WHERE reviews_per_month IS NOT NULL
GROUP BY neighbourhood_group;

-- Listings with zero reviews
SELECT COUNT(*) FROM listings WHERE number_of_reviews = 0;

-- Most reviewed listings (top 10)
SELECT id, name, number_of_reviews
FROM listings
ORDER BY number_of_reviews DESC
LIMIT 10;

-- Count reviews by month (last_review)
SELECT 
    EXTRACT(YEAR FROM last_review) AS year,
    EXTRACT(MONTH FROM last_review) AS month,
    COUNT(*) AS review_count
FROM listings
WHERE last_review IS NOT NULL
GROUP BY year, month
ORDER BY year, month;


-- Room Type & Minimum Nights
-- Distribution of room types
-- Average minimum_nights per room type
-- Listings that require long stays (minimum_nights > 30)

SELECT 
    room_type,
    COUNT(*) AS count,
    AVG(minimum_nights) AS avg_min_nights,
    MAX(minimum_nights) AS max_min_nights
FROM listings
GROUP BY room_type;

-- Correlation between price and number_of_reviews 
-- Correlation between availability and price
SELECT
(
    COUNT(*) * SUM(price * number_of_reviews)
    - SUM(price) * SUM(number_of_reviews)
)
/
SQRT(
    (COUNT(*) * SUM(price * price) - POWER(SUM(price),2))
    *
    (COUNT(*) * SUM(number_of_reviews * number_of_reviews)
    - POWER(SUM(number_of_reviews),2))
) AS price_reviews_corr
FROM listings
WHERE price IS NOT NULL
AND number_of_reviews IS NOT NULL;
-- there is a weak negative relationship meaning there is no linear relationship between the two.

-- Price Elasticity: Compare Price vs. Reviews per Month
-- Goal: See if higher-priced listings get fewer reviews per month (suggesting lower demand or bookings).

-- Strategy: Group prices into buckets (e.g., $0–100, $100–200, etc.) and calculate the average reviews_per_month for each bucket.



SELECT 
    CASE 
        WHEN price < 100 THEN '0-100'
        WHEN price BETWEEN 100 AND 199 THEN '100-199'
        WHEN price BETWEEN 200 AND 399 THEN '200-399'
        WHEN price >= 400 THEN '400+'
        ELSE 'Unknown'
    END AS price_bucket,
    COUNT(*) AS listing_count,
    AVG(reviews_per_month) AS avg_reviews_per_month
FROM listings
WHERE price IS NOT NULL AND reviews_per_month IS NOT NULL
GROUP BY price_bucket
ORDER BY avg_reviews_per_month DESC;
-- in this case there is indeed price sensitivity as prices go up, yet for the price bucket from 100-199, there exist consumer loyalty. 


--  Seasonal Pricing (Proxy using last_review)

SELECT 
    EXTRACT(MONTH FROM last_review) AS review_month,
    COUNT(*) AS review_count,
    AVG(price) AS avg_price_of_reviewed_listings
FROM listings
WHERE last_review IS NOT NULL AND price IS NOT NULL
GROUP BY review_month
ORDER BY review_month;
-- the above shows the peak seasons as from july,august and december

-- Host Segmentation (by Listing Count, Price, and Reviews)
-- Goal: Group hosts into categories (e.g., “Solo Host”, “Small Business”, “Professional”) and compare their average price and total reviews.


WITH host_stats AS (
    SELECT 
        host_id,
        COUNT(*) AS num_listings,
        AVG(price) AS avg_price,
        SUM(number_of_reviews) AS total_reviews
    FROM listings
    WHERE price IS NOT NULL
    GROUP BY host_id
)
SELECT 
    CASE 
        WHEN num_listings = 1 THEN 'Solo (1)'
        WHEN num_listings BETWEEN 2 AND 3 THEN 'Small (2-3)'
        WHEN num_listings BETWEEN 4 AND 10 THEN 'Medium (4-10)'
        WHEN num_listings > 10 THEN 'Large (11+)'
    END AS host_segment,
    COUNT(*) AS host_count,
    AVG(num_listings) AS avg_listings_per_host,
    AVG(avg_price) AS avg_price_per_host,
    AVG(total_reviews) AS avg_total_reviews_per_host
FROM host_stats
GROUP BY host_segment
ORDER BY avg_listings_per_host;
-- large hosts have their average prices relatively low compared to solo host. they have more reviews.

-- Neighbourhood Group Comparison (Rank Using Window Functions)
-- Goal: Rank neighbourhoods within each neighbourhood group by average price. This helps you spot the most expensive (and cheapest) pockets inside each area
WITH neighbourhood_avg AS (
    SELECT 
        n.neighbourhood_group,
        n.neighbourhood,
        AVG(l.price) AS avg_price,
        COUNT(l.id) AS listing_count
    FROM neighbourhoods n
    LEFT JOIN listings l ON n.neighbourhood = l.neighbourhood
    WHERE l.price IS NOT NULL
    GROUP BY n.neighbourhood_group, n.neighbourhood
)
SELECT 
    neighbourhood_group,
    neighbourhood,
    avg_price,
    listing_count,
    ROW_NUMBER() OVER (PARTITION BY neighbourhood_group ORDER BY avg_price DESC) AS price_rank_within_group
FROM neighbourhood_avg
ORDER BY neighbourhood_group, price_rank_within_group;
-- crownhill and downtown among the cheapest and expensive neighbourhoods respectively 

-- License Status Analysis
-- Goal: Compare licensed listings (having a real license number) against unlicensed/exempt ones. We’ll define “Valid License” as license is not NULL, not empty, and not the string 'Exempt'.
 SELECT 
    CASE 
        WHEN license IS NOT NULL AND license != '' AND license != 'Exempt' THEN 'Licensed'
        ELSE 'Unlicensed / Exempt'
    END AS license_status,
    COUNT(*) AS listing_count,
    AVG(price) AS avg_price,
    AVG(reviews_per_month) AS avg_reviews_per_month,
    AVG(availability_365) AS avg_availability
FROM listings
WHERE price IS NOT NULL
GROUP BY license_status;
-- licensed listings are relatively more expensive and more availbility. high reviews could be to high proffesionalism.
-- licensing correlates to high professionalism and high quality.

-- Outlier Detection (Price > 3 Standard Deviations Above Mean)
-- Goal: Find listings that are abnormally expensive—those with a price more than 3 standard deviations above the average.

WITH stats AS (
    SELECT 
        AVG(price) AS avg_price,
        STDDEV(price) AS std_price
    FROM listings
    WHERE price IS NOT NULL
)
SELECT 
    id, name, neighbourhood, room_type, price,
    ROUND((price - avg_price) / std_price, 2) AS z_score
FROM listings, stats
WHERE price IS NOT NULL
  AND price > (avg_price + 3 * std_price)
ORDER BY price DESC;
-- the seattle magic  in fremont neighbourhood comes out as a luxury suite. its expensive with 4.9 rating


-- Rising stars
-- Goal: Find listings that are relatively new (low total reviews) but getting a high rate of new reviews per month—these are rising stars. Compare them to established listings.

SELECT 
    CASE 
        WHEN number_of_reviews < 20 AND reviews_per_month > 3 THEN 'New & Popular'
        WHEN number_of_reviews >= 50 THEN 'Established'
        ELSE 'Average / Other'
    END AS review_category,
    COUNT(*) AS listing_count,
    AVG(price) AS avg_price,
    AVG(reviews_per_month) AS avg_reviews_per_month
FROM listings
WHERE reviews_per_month IS NOT NULL AND price IS NOT NULL
GROUP BY review_category
ORDER BY avg_reviews_per_month DESC;





