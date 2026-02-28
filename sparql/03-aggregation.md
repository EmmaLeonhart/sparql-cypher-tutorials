# SPARQL Tutorial 3: Aggregation

## Overview

Aggregation lets you summarize data: count items, compute averages, find extremes, and group results. If you have used `GROUP BY` in SQL, the SPARQL equivalents will feel familiar.

## Query 1: COUNT with GROUP BY

```sparql
SELECT ?countryLabel (COUNT(?shrine) AS ?shrineCount) WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P17 ?country .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
GROUP BY ?countryLabel
ORDER BY DESC(?shrineCount)
```

**What this does:**
- Counts Shinto shrines per country
- `GROUP BY ?countryLabel` -- one row per country
- `ORDER BY DESC(?shrineCount)` -- most shrines first
- Japan will dominate, but you may see a few shrines in other countries (USA, Brazil, Hawaii before statehood entries, etc.)

**Expected output:**

| countryLabel | shrineCount |
|-------------|-------------|
| Japan | ~3000+ |
| United States of America | ~10-20 |
| Brazil | ~5-10 |
| ... | ... |

## Query 2: HAVING -- Filter After Aggregation

```sparql
SELECT ?prefectureLabel (COUNT(?shrine) AS ?shrineCount) WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P131 ?prefecture .
  ?prefecture wdt:P31 wd:Q50337 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
GROUP BY ?prefectureLabel
HAVING (COUNT(?shrine) > 20)
ORDER BY DESC(?shrineCount)
```

**What this does:**
- Counts shrines per Japanese prefecture (Q50337 = prefecture of Japan)
- `wdt:P131` = located in administrative territorial entity
- `HAVING` filters groups *after* aggregation (unlike `FILTER` which operates before)
- Only shows prefectures with more than 20 shrines catalogued on Wikidata

**Expected output:** Prefectures with significant shrine counts. Kyoto, Nara, and Mie prefectures often rank high due to their historical importance.

## Query 3: Multiple Aggregations

```sparql
SELECT
  ?typeLabel
  (COUNT(?shrine) AS ?total)
  (MIN(?inception) AS ?earliest)
  (MAX(?inception) AS ?latest)
WHERE {
  ?shrine wdt:P31 ?type ;
          wdt:P571 ?inception .
  ?type wdt:P279* wd:Q728 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
GROUP BY ?typeLabel
ORDER BY DESC(?total)
LIMIT 15
```

**What this does:**
- Groups by shrine subtype (using `P279*` to find subclasses of Shinto shrine)
- For each type, shows the count, earliest founding date, and latest founding date
- `MIN(?inception)` / `MAX(?inception)` -- aggregate date values

**Expected output:** Various shrine types (Shinto shrine, Inari shrine, Hachiman shrine, etc.) with counts and date ranges.

## Query 4: SAMPLE and GROUP_CONCAT

```sparql
SELECT
  ?prefectureLabel
  (COUNT(?shrine) AS ?total)
  (GROUP_CONCAT(DISTINCT ?shrineLabel; SEPARATOR=", ") AS ?examples)
WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P131 ?prefecture ;
          rdfs:label ?shrineLabel .
  ?prefecture wdt:P31 wd:Q50337 .
  FILTER(LANG(?shrineLabel) = "en")
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
GROUP BY ?prefectureLabel
ORDER BY DESC(?total)
LIMIT 10
```

**What this does:**
- `GROUP_CONCAT` concatenates all shrine names into a single comma-separated string per prefecture
- `DISTINCT` avoids duplicate names
- `SEPARATOR=", "` specifies the delimiter
- This gives you a quick overview of which shrines are in each prefecture

**Expected output:** Top 10 prefectures with shrine counts and a list of example shrine names in each.

**Note:** `GROUP_CONCAT` results can be very long. Wikidata may truncate them. For large groups, consider using `SAMPLE` instead, which returns just one example:

```sparql
(SAMPLE(?shrineLabel) AS ?exampleShrine)
```

## Query 5: Nested Aggregation with Subquery

```sparql
SELECT ?range (COUNT(?prefecture) AS ?prefectureCount) WHERE {
  {
    SELECT ?prefecture (COUNT(?shrine) AS ?shrineCount) WHERE {
      ?shrine wdt:P31 wd:Q728 ;
              wdt:P131 ?prefecture .
      ?prefecture wdt:P31 wd:Q50337 .
    }
    GROUP BY ?prefecture
  }
  BIND(
    IF(?shrineCount < 10, "1-9",
    IF(?shrineCount < 50, "10-49",
    IF(?shrineCount < 100, "50-99",
    "100+"))) AS ?range
  )
}
GROUP BY ?range
ORDER BY ?range
```

**What this does:**
- The inner subquery counts shrines per prefecture
- The outer query bins those counts into ranges
- `BIND(IF(...))` creates categorical bins
- Result: a histogram of "how many prefectures have X shrines"

**Expected output:**

| range | prefectureCount |
|-------|----------------|
| 1-9 | ~20-30 |
| 10-49 | ~10-15 |
| 50-99 | ~3-5 |
| 100+ | ~1-3 |

## Aggregation Functions Reference

| Function | Purpose | Example |
|----------|---------|---------|
| `COUNT` | Count items | `COUNT(?x)` or `COUNT(DISTINCT ?x)` |
| `SUM` | Sum numeric values | `SUM(?population)` |
| `AVG` | Average | `AVG(?elevation)` |
| `MIN` | Minimum value | `MIN(?date)` |
| `MAX` | Maximum value | `MAX(?date)` |
| `SAMPLE` | One arbitrary value from group | `SAMPLE(?label)` |
| `GROUP_CONCAT` | Concatenate strings | `GROUP_CONCAT(?name; SEPARATOR=", ")` |

## Exercises

1. **Count the number of Buddhist temples (Q44539) per country.** Which country has the most?
2. **Find the average population of Japanese prefectures** (Q50337 with P1082).
3. **Use GROUP_CONCAT to list all languages** (P37 = official language) for each country (Q6256) in East Asia. Limit to 10 countries.
4. **Create a histogram** of how many countries fall into population ranges: under 1M, 1M-10M, 10M-100M, 100M+.
