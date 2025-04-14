
Correlation Analyzer Plus
-------------------------------------
This SQL script dynamically calculates pairwise **Pearson correlation coefficients** between all numeric columns in a given table. It is ideal for feature selection, exploratory data analysis (EDA), and identifying multicollinearity directly in SQL.

## Features
- Computes pairwise correlations using `corr()`
- Filters for significant correlations (e.g., |r| > 0.7)
- Uses system catalog (`information_schema.columns`) to detect numeric columns
- Example table `features` included with synthetic data

## Requirements
- **PostgreSQL** (because it supports `corr()` as an aggregate function)

## Use Case Examples
- Feature engineering: Detect correlated variables
- Data validation before training ML models
- SQL-native dashboards or pipelines with embedded drift checks

## How to Run
1. Open in a PostgreSQL-compatible SQL editor
2. Execute blockweise:
   - Create + populate table
   - Run correlation analysis (with optional WHERE filter)

## Optional Extensions
- Store correlation matrix in a reporting table
- Create alerts for strong dependencies (e.g., income vs. expenses)
- Visualize results with BI tools based on exported SQL

## Author
Janet Abdullah  
GitHub: [https://github.com/JanetAbdullah]  
This project is designed for reproducible, interpretable data profiling in SQL-only workflows.
*/

-- Step 1: Create demo table
DROP TABLE IF EXISTS features;
CREATE TABLE features (
    id SERIAL PRIMARY KEY,
    height REAL,
    weight REAL,
    age INTEGER,
    income REAL,
    expenses REAL
);

-- Step 2: Insert synthetic data
INSERT INTO features (height, weight, age, income, expenses)
SELECT
    round(random()*50 + 150, 1),
    round(random()*30 + 60, 1),
    (random()*40 + 20)::INT,
    round(random()*20000 + 30000, 2),
    round(random()*15000 + 10000, 2)
FROM generate_series(1, 1000);

-- Step 3: Compute pairwise correlations
-- using CROSS JOIN of columns and dynamic SQL via CTE
WITH cols AS (
    SELECT column_name
    FROM information_schema.columns
    WHERE table_name = 'features'
      AND data_type IN ('real', 'integer')
      AND column_name != 'id'
),
pairs AS (
    SELECT c1.column_name AS col1, c2.column_name AS col2
    FROM cols c1
    JOIN cols c2 ON c1.column_name < c2.column_name
),
correlations AS (
    SELECT
        col1,
        col2,
        (SELECT corr(f1, f2)
         FROM (SELECT
                  (f1::REAL) AS f1,
                  (f2::REAL) AS f2
               FROM features) AS sub
        ) AS correlation
    FROM (
        SELECT
            col1,
            col2,
            (SELECT f1 FROM features) AS f1,
            (SELECT f2 FROM features) AS f2
        FROM pairs
    ) AS joined
)
SELECT *
FROM correlations
WHERE ABS(correlation) > 0.7
ORDER BY ABS(correlation) DESC;

-- Note: For static/manual use, you can directly write:
-- SELECT corr(height, weight) FROM features;
-- SELECT corr(age, income) FROM features;

-- Optional: Create correlation matrix (manually)
-- SELECT
--   corr(height, weight) AS height_weight,
--   corr(height, age) AS height_age,
--   corr(height, income) AS height_income
-- FROM features;
