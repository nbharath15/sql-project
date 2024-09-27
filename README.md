# sql-project on ONLINE_SPORTS_RETAIL_REVENUE ON BRANDS

create database online_sports_revenue;

-- RENAMING THE TABLES
rename table brands_v2 to brands;
rename table info_v2 to info;
rename table reviews_v2 to reviews;
rename table traffic_v3 to traffic;


-- 1. COUNTING NUMBER OF ROWS

USE online_sports_revenue;

SELECT 
    COUNT(*) AS total_rows,
    SUM(info.description IS NOT NULL) AS count_description,  -- Count non-NULL descriptions
    SUM(finance.listing_price IS NOT NULL) AS count_listing_price,  -- Count non-NULL listing prices
    SUM(traffic.last_visited IS NOT NULL) AS count_last_visited  -- Count non-NULL last visited dates
FROM 
    info
INNER JOIN 
    finance ON info.product_id = finance.product_id  -- Replace with actual join condition
INNER JOIN 
    traffic ON info.product_id = traffic.product_id;

-- 2 NIKE VS ADIDAS PRICING

SELECT 
    brands.brand, 
    CAST(finance.listing_price AS UNSIGNED) AS listing_price, 
    COUNT(finance.product_id) AS product_count
FROM 
    brands
INNER JOIN 
    finance ON finance.product_id = brands.product_id
WHERE 
    finance.listing_price > 0
GROUP BY 
    brands.brand, CAST(finance.listing_price AS UNSIGNED)
ORDER BY 
    listing_price DESC;
    
-- 3  lABELLING PRICE RANGES

SELECT 
    b.brand, 
    COUNT(f.product_id) AS product_count,  -- Count a specific column instead of COUNT(f.*)
    SUM(f.revenue) AS total_revenue,
    CASE 
        WHEN f.listing_price < 42 THEN 'Budget'
        WHEN f.listing_price >= 42 AND f.listing_price < 74 THEN 'Average'
        WHEN f.listing_price >= 74 AND f.listing_price < 129 THEN 'Expensive'
        ELSE 'Elite' 
    END AS price_category
FROM 
    finance AS f
INNER JOIN 
    brands AS b ON f.product_id = b.product_id
WHERE 
    b.brand IS NOT NULL
GROUP BY 
    b.brand, 
    price_category  -- 'price_category' is fine here as it’s defined in SELECT
ORDER BY 
    total_revenue DESC;

-- 4 CORRELATION BETWEEN REVENUE AND REVIEWS

SELECT 
    (COUNT(*) * SUM(reviews.reviews * finance.revenue) 
     - SUM(reviews.reviews) * SUM(finance.revenue)) / 
    (SQRT((COUNT(*) * SUM(reviews.reviews * reviews.reviews) - POWER(SUM(reviews.reviews), 2)) * 
           (COUNT(*) * SUM(finance.revenue * finance.revenue) - POWER(SUM(finance.revenue), 2)))
    ) AS review_revenue_corr
FROM 
    reviews
INNER JOIN 
    finance ON finance.product_id = reviews.product_id;

-- 5 RATINGS AND REVIEWS BY PRODUCT DESCRIPTION LENGTH

SELECT 
    LENGTH(i.description) - MOD(LENGTH(i.description), 100) AS description_length,  -- Truncate length to nearest hundred
    ROUND(AVG(r.rating), 2) AS average_rating
FROM 
    info AS i
INNER JOIN 
    reviews AS r ON i.product_id = r.product_id
WHERE 
    i.description IS NOT NULL
GROUP BY 
    description_length
ORDER BY 
    description_length;

-- 6 REVIEWS BY MONTH AND BRAND

SELECT 
    b.brand, 
    MONTH(t.last_visited) AS month, 
    COUNT(r.product_id) AS num_reviews
FROM 
    brands AS b
INNER JOIN 
    traffic AS t ON b.product_id = t.product_id
INNER JOIN 
    reviews AS r ON t.product_id = r.product_id
WHERE 
    b.brand IS NOT NULL
    AND t.last_visited IS NOT NULL
GROUP BY 
    b.brand, 
    month
ORDER BY 
    b.brand, 
    month;
    
-- 7 FOOTWEAR PRODUCT PERFORMANCE

WITH footwear AS (
    SELECT i.description, f.revenue
    FROM info AS i
    INNER JOIN finance AS f 
        ON i.product_id = f.product_id
    WHERE (LOWER(i.description) LIKE '%shoe%'
           OR LOWER(i.description) LIKE '%trainer%'
           OR LOWER(i.description) LIKE '%foot%')
        AND i.description IS NOT NULL
),
ranked_footwear AS (
    SELECT revenue, 
           ROW_NUMBER() OVER (ORDER BY revenue) AS row_num,
           COUNT(*) OVER () AS total_count
    FROM footwear
)

SELECT 
    COUNT(*) AS num_footwear_products,
    CASE 
        WHEN total_count % 2 = 1 THEN 
            (SELECT revenue 
             FROM ranked_footwear 
             WHERE row_num = (total_count DIV 2) + 1)  -- Middle element for odd count
        ELSE 
            (SELECT AVG(revenue) 
             FROM ranked_footwear 
             WHERE row_num IN (total_count DIV 2, (total_count DIV 2) + 1))  -- Average of two middle elements for even count
    END AS median_footwear_revenue
FROM ranked_footwear;

-- 8 CLOTHING PRODUCT PERFORMANCE

WITH footwear AS (
    SELECT i.description
    FROM info AS i
    INNER JOIN finance AS f ON i.product_id = f.product_id
    WHERE i.description LIKE BINARY '%shoe%'
        OR i.description LIKE BINARY '%trainer%'
        OR i.description LIKE BINARY '%foot%'
        AND i.description IS NOT NULL
),
clothing AS (
    SELECT i.description, f.revenue
    FROM info AS i
    INNER JOIN finance AS f ON i.product_id = f.product_id
    WHERE i.description NOT IN (SELECT description FROM footwear)
      AND i.description IS NOT NULL
),
clothing_with_ranks AS (
    SELECT revenue, 
           ROW_NUMBER() OVER (ORDER BY revenue) AS row_num,
           COUNT(*) OVER () AS total_count
    FROM clothing
)

SELECT 
    (SELECT COUNT(*) FROM clothing) AS num_clothing_products,  -- Count of clothing products
    AVG(revenue) AS median_clothing_revenue  -- Median of revenue for clothing products
FROM clothing_with_ranks
WHERE row_num IN (
    FLOOR((total_count + 1) / 2), 
    CEIL((total_count + 1) / 2)
);
![Screenshot (123)](https://github.com/user-attachments/assets/062b63ff-6d7e-4783-91d1-b2f066f7479e)

