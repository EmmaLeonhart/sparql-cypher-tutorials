# SPARQL Tutorial 5: Subqueries, VALUES, and BIND

## Overview

As queries grow more complex, you need ways to structure them. Subqueries let you compute intermediate results. `VALUES` lets you inject inline data. `BIND` lets you create computed variables. Together, these tools let you build sophisticated queries piece by piece.

## Query 1: VALUES -- Querying a Specific Set

```sparql
SELECT ?shrine ?shrineLabel ?shrineDescription ?prefectureLabel WHERE {
  VALUES ?shrine {
    wd:Q273196   # Ise Grand Shrine
    wd:Q639683   # Fushimi Inari-taisha
    wd:Q617572   # Meiji Shrine
    wd:Q1069719  # Kasuga Grand Shrine
    wd:Q819654   # Izumo-taisha
  }
  ?shrine wdt:P131 ?prefecture .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
```

**What this does:**
- `VALUES` provides an explicit list of items to query
- This is far more efficient than `FILTER(?shrine IN (wd:Q273196, wd:Q639683, ...))` because the query engine can look up each item directly
- Useful when you have a curated list from external research

**Expected output:**

| shrine | shrineLabel | prefectureLabel |
|--------|-------------|----------------|
| wd:Q273196 | Ise Grand Shrine | Ise, Mie |
| wd:Q639683 | Fushimi Inari-taisha | Fushimi-ku |
| wd:Q617572 | Meiji Shrine | Shibuya |
| ... | ... | ... |

## Query 2: BIND -- Computed Variables

```sparql
SELECT ?shrine ?shrineLabel ?inception ?age WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P571 ?inception .
  BIND(YEAR(NOW()) - YEAR(?inception) AS ?age)
  FILTER(?age > 1000)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
ORDER BY ?inception
LIMIT 20
```

**What this does:**
- `BIND(... AS ?age)` creates a new variable computed from existing data
- `YEAR(NOW())` extracts the current year
- `YEAR(?inception)` extracts the founding year
- The subtraction gives an approximate age in years
- We filter to shrines over 1,000 years old

**Expected output:** Ancient shrines ordered by founding date. Ise Grand Shrine (traditionally founded ~4 BCE), Izumo-taisha, and other ancient shrines.

## Query 3: Subquery -- Top N Per Group

```sparql
SELECT ?prefectureLabel ?shrineLabel ?shrineCount WHERE {
  {
    SELECT ?prefecture (COUNT(?s) AS ?shrineCount) WHERE {
      ?s wdt:P31 wd:Q728 ;
         wdt:P131 ?prefecture .
      ?prefecture wdt:P31 wd:Q50337 .
    }
    GROUP BY ?prefecture
    ORDER BY DESC(?shrineCount)
    LIMIT 5
  }
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P131 ?prefecture .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
ORDER BY DESC(?shrineCount) ?shrineLabel
```

**What this does:**
- The inner subquery finds the top 5 prefectures by shrine count
- The outer query then retrieves all shrines in those prefectures
- This is a common pattern: use a subquery to narrow down, then join back for details

**Expected output:** All shrines in the top 5 prefectures, grouped by prefecture with the shrine count shown.

## Query 4: VALUES with Multiple Columns

```sparql
SELECT ?item ?itemLabel ?expectedType WHERE {
  VALUES (?item ?expectedType) {
    (wd:Q273196  "Grand Shrine")
    (wd:Q639683  "Inari Shrine")
    (wd:Q617572  "Imperial Shrine")
    (wd:Q819654  "Taisha")
  }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
```

**What this does:**
- `VALUES` can bind multiple variables at once
- Each row in the VALUES block binds both `?item` and `?expectedType`
- Useful for enriching a dataset with annotations you define inline

**Expected output:**

| item | itemLabel | expectedType |
|------|-----------|-------------|
| wd:Q273196 | Ise Grand Shrine | Grand Shrine |
| wd:Q639683 | Fushimi Inari-taisha | Inari Shrine |
| wd:Q617572 | Meiji Shrine | Imperial Shrine |
| wd:Q819654 | Izumo-taisha | Taisha |

## Query 5: Combining Subqueries with BIND

```sparql
SELECT ?prefectureLabel ?total ?withCoords ?coverage WHERE {
  {
    SELECT ?prefecture
           (COUNT(?shrine) AS ?total)
           (COUNT(?coords) AS ?withCoords)
    WHERE {
      ?shrine wdt:P31 wd:Q728 ;
              wdt:P131 ?prefecture .
      ?prefecture wdt:P31 wd:Q50337 .
      OPTIONAL { ?shrine wdt:P625 ?coords . }
    }
    GROUP BY ?prefecture
  }
  BIND(
    CONCAT(STR(ROUND(?withCoords * 100 / ?total)), "%") AS ?coverage
  )
  FILTER(?total > 5)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
ORDER BY DESC(?total)
```

**What this does:**
- The subquery counts total shrines and shrines with coordinates per prefecture
- `BIND` computes a coverage percentage
- `CONCAT(STR(ROUND(...)), "%")` formats it as a percentage string
- `FILTER(?total > 5)` excludes prefectures with very few shrines
- This is a data quality report: which prefectures have good coordinate coverage?

**Expected output:**

| prefectureLabel | total | withCoords | coverage |
|----------------|-------|------------|----------|
| Kyoto Prefecture | ~100+ | ~80+ | ~80% |
| Tokyo | ~50+ | ~40+ | ~80% |
| ... | ... | ... | ... |

## When to Use Each

| Tool | Use When |
|------|----------|
| `VALUES` | You have a specific list of items to query |
| `BIND` | You need computed/derived variables |
| Subquery | You need intermediate aggregation or ranked results |
| `VALUES` + Subquery | You want to query a specific set and compute aggregates on it |

## Exercises

1. **Use VALUES to look up five countries of your choice.** Retrieve their population (P1082), capital (P36), and official language (P37).
2. **Use BIND to compute population density.** Query countries with both population (P1082) and area (P2046), then `BIND(?population / ?area AS ?density)`. Show the top 10.
3. **Write a subquery** that finds the 3 Japanese prefectures with the most Buddhist temples (Q44539), then list all temples in those prefectures.
4. **Combine VALUES and BIND:** Take 5 shrines, query their founding dates, and BIND a classification: "Ancient" (before 800 CE), "Medieval" (800-1600), "Modern" (after 1600).
