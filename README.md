# USD 259 bond analysis: PostGIS queries

PostGIS queries behind the USD 259 bond graphics.



```sql
WITH votes_2025 AS (
    SELECT "PRECINCT NAME" AS precinct_name,
           "REGISTERED VOTERS TOTAL"::int AS registered_voters,
           NULLIF("USD 259 Bond Question",   'REDACTED')::int AS yes_votes,
           NULLIF("USD 259 Bond Question_1", 'REDACTED')::int AS no_votes,
           NULLIF("USD 259 Bond Question",   'REDACTED')::int
             + NULLIF("USD 259 Bond Question_1", 'REDACTED')::int AS total_votes
    FROM elections."USD259_2025"
    WHERE "PRECINCT CODE" ~ '^\d+$'          -- numbered precincts only
),

votes_2026 AS (
    SELECT "PRECINCT NAME" AS precinct_name,
           "REGISTERED VOTERS TOTAL"::int AS registered_voters,
           NULLIF("USD 259 Proposition No. 2",   'REDACTED')::int AS yes_votes,
           NULLIF("USD 259 Proposition No. 2_1", 'REDACTED')::int AS no_votes,
           NULLIF("USD 259 Proposition No. 2",   'REDACTED')::int
             + NULLIF("USD 259 Proposition No. 2_1", 'REDACTED')::int AS total_votes
    FROM elections."USD259_2026"
    WHERE "PRECINCT CODE" ~ '^\d+$'
),

calculations AS (
    SELECT "2025".precinct_name,
           "2025".registered_voters AS registered_voters_2025,
           "2025".yes_votes         AS yes_votes_2025,
           "2025".no_votes          AS no_votes_2025,
           "2025".total_votes       AS total_2025,
           "2026".registered_voters AS registered_voters_2026,
           "2026".yes_votes         AS yes_votes_2026,
           "2026".no_votes          AS no_votes_2026,
           "2026".total_votes       AS total_2026,

           -- Yes share each year (NULLIF avoids dividing by zero)
           "2025".yes_votes / NULLIF("2025".total_votes, 0)::numeric AS perc_yes_2025,
           "2026".yes_votes / NULLIF("2026".total_votes, 0)::numeric AS perc_yes_2026,

           -- Change in Yes share, percentage points
           ROUND((("2026".yes_votes / NULLIF("2026".total_votes, 0)::numeric)
                - ("2025".yes_votes / NULLIF("2025".total_votes, 0)::numeric)) * 100, 1)
                AS percent_point_difference_yes_votes,

           -- Change in votes on the question
           "2026".total_votes - "2025".total_votes AS ballot_change,

           -- Change in share of registered voters voting on the question
           ROUND((("2026".total_votes / NULLIF("2026".registered_voters, 0)::numeric)
                - ("2025".total_votes / NULLIF("2025".registered_voters, 0)::numeric)) * 100, 1)
                AS percent_point_difference_turnout,

           "2026".yes_votes - "2025".yes_votes AS yes_change,

           -- Yes-minus-No margins
           "2026".yes_votes - "2026".no_votes  AS yes_margin_2026,
           "2025".yes_votes - "2025".no_votes  AS yes_margin_2025,
           ("2026".yes_votes - "2026".no_votes)
             - ("2025".yes_votes - "2025".no_votes) AS yes_margin_change
    FROM votes_2025 AS "2025"
    JOIN votes_2026 AS "2026"
      ON "2025".precinct_name = "2026".precinct_name
)

SELECT calculations.*, b."geom"
FROM calculations
JOIN boundaries."Sedgwick County Election Precincts" AS b
  ON calculations.precinct_name = b."Precinct"
ORDER BY percent_point_difference_yes_votes;
```

## 2. Schools by funding and support


```sql
WITH precinct_votes AS (
    SELECT "PRECINCT NAME" AS precinct_name,
           NULLIF("USD 259 Proposition No. 2",   'REDACTED')::int AS yes_votes,
           NULLIF("USD 259 Proposition No. 2_1", 'REDACTED')::int AS no_votes
    FROM elections."USD259_2026"
    WHERE "PRECINCT CODE" ~ '^\d+$'
),

precinct_pct AS (
    SELECT v.precinct_name,
           b.geom,
           v.yes_votes + v.no_votes AS total_votes,
           100.0 * v.yes_votes / (v.yes_votes + v.no_votes) AS yes_pct
    FROM precinct_votes v
    JOIN boundaries."Sedgwick County Election Precincts" b
      ON v.precinct_name = b."Precinct"
    WHERE v.yes_votes + v.no_votes > 200     -- skip small precincts
),

precinct_tiles AS (
    SELECT *,
           NTILE(3) OVER (ORDER BY yes_pct DESC) AS support_tile
    FROM precinct_pct
),

school_tiles AS (
    SELECT schoolnm,
           bond_funding_2026,
           ST_Transform(geom, 3420) AS geom,  -- match precincts' coordinate system
           NTILE(3) OVER (ORDER BY bond_funding_2026 DESC) AS funding_tile
    FROM "SedgwickCountySchools"
    WHERE bond_funding_2026 IS NOT NULL       -- funded schools only
)

SELECT s.schoolnm,
       s.bond_funding_2026,
       s.funding_tile,
       p.precinct_name,
       p.total_votes,
       ROUND(p.yes_pct, 1) AS yes_pct,
       p.support_tile,
       CASE
           WHEN s.funding_tile = 1 AND p.support_tile = 1 THEN 'High funding, high support'
           WHEN s.funding_tile = 3 AND p.support_tile = 1 THEN 'Low funding, high support'
           WHEN s.funding_tile = 1 AND p.support_tile = 3 THEN 'High funding, lower support'
           WHEN s.funding_tile = 3 AND p.support_tile = 3 THEN 'Low funding, lower support'
           ELSE 'Middle'
       END AS category,
       s.geom
FROM school_tiles s
JOIN precinct_tiles p
  ON ST_Within(s.geom, ST_CurveToLine(p.geom))  -- curved precincts -> ordinary polygons
ORDER BY category, s.bond_funding_2026 DESC;
```
