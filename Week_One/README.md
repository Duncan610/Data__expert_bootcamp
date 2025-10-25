-- Step 1: Create composite type (struct)
CREATE TYPE film_info AS (
    film TEXT,
    votes INT,
    rating FLOAT,
    filmid TEXT
);

-- Step 2: Create dimensional table
CREATE TABLE actors (
    actor_id TEXT PRIMARY KEY,
    films film_info[],
    quality_class TEXT,
    is_active BOOLEAN
);

-- Step 3 & 4: Use CTEs to transform and insert data
WITH latest_year AS (
    -- Identify the most recent film year per actor
    SELECT 
        actorid,
        MAX(year) AS latest_year
    FROM actor_films
    GROUP BY actorid
),
actor_summary AS (
    -- Transform actor_films into aggregated dimensional format
    SELECT 
        af.actorid,
        ARRAY_AGG(ROW(af.film, af.votes, af.rating, af.filmid)::film_info) AS films,
        
        CASE 
            WHEN AVG(af.rating) FILTER (WHERE af.year = ly.latest_year) > 8 THEN 'star'
            WHEN AVG(af.rating) FILTER (WHERE af.year = ly.latest_year) > 7 THEN 'good'
            WHEN AVG(af.rating) FILTER (WHERE af.year = ly.latest_year) > 6 THEN 'average'
            ELSE 'bad'
        END AS quality_class,
        
        BOOL_OR(af.year = EXTRACT(YEAR FROM CURRENT_DATE)) AS is_active
    FROM actor_films af
    JOIN latest_year ly
    ON af.actorid = ly.actorid
    GROUP BY af.actorid, ly.latest_year
)
-- Step 5: Insert transformed data into target table
INSERT INTO actors (actor_id, films, quality_class, is_active)
SELECT actorid, films, quality_class, is_active
FROM actor_summary;
