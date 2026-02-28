# SPARQL Tutorial 2: Filtering

## Overview

Real queries almost always need to filter results. SPARQL provides several mechanisms: `FILTER` for conditions, `OPTIONAL` for data that may not exist, `UNION` for combining alternatives, and `MINUS` for exclusion.

## Query 1: FILTER with String Matching

```sparql
SELECT ?shrine ?shrineLabel WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          rdfs:label ?shrineLabel .
  FILTER(LANG(?shrineLabel) = "en")
  FILTER(CONTAINS(?shrineLabel, "Inari"))
}
LIMIT 20
```

**What this does:**
- `rdfs:label` retrieves the raw label (not through the label service)
- `FILTER(LANG(?shrineLabel) = "en")` -- only English labels
- `FILTER(CONTAINS(?shrineLabel, "Inari"))` -- label must contain "Inari"
- Inari shrines (dedicated to the kami Inari) are among the most numerous in Japan

**Expected output:** Shrines with "Inari" in their English name, such as "Fushimi Inari-taisha", "Toyokawa Inari", etc.

## Query 2: OPTIONAL -- Handling Missing Data

```sparql
SELECT ?shrine ?shrineLabel ?websiteLabel WHERE {
  ?shrine wdt:P31 wd:Q728 .
  OPTIONAL { ?shrine wdt:P856 ?website . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 25
```

**What this does:**
- `OPTIONAL` means: include `?website` if it exists, but still return the shrine even if it does not
- Without `OPTIONAL`, shrines lacking a website would be excluded entirely
- This is the SPARQL equivalent of a LEFT JOIN in SQL

**Expected output:** 25 shrines. Some will have a website URL; others will show an empty cell for `?websiteLabel`.

## Query 3: FILTER with Numeric Comparisons

```sparql
SELECT ?city ?cityLabel ?population WHERE {
  ?city wdt:P31 wd:Q515 ;
        wdt:P17 wd:Q17 ;
        wdt:P1082 ?population .
  FILTER(?population > 1000000)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
ORDER BY DESC(?population)
LIMIT 20
```

**What this does:**
- Finds cities (Q515) in Japan (Q17) with population (P1082) over 1 million
- `FILTER(?population > 1000000)` -- numeric comparison
- `ORDER BY DESC(?population)` -- sort largest first

**Expected output:** Major Japanese cities ordered by population: Tokyo (special ward area), Yokohama, Osaka, Nagoya, Sapporo, etc.

## Query 4: UNION -- Combining Patterns

```sparql
SELECT ?place ?placeLabel ?typeLabel WHERE {
  {
    ?place wdt:P31 wd:Q728 .
    BIND(wd:Q728 AS ?type)
  }
  UNION
  {
    ?place wdt:P31 wd:Q44539 .
    BIND(wd:Q44539 AS ?type)
  }
  ?place wdt:P17 wd:Q17 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 30
```

**What this does:**
- `UNION` combines two patterns: Shinto shrines (Q728) and Buddhist temples (Q44539)
- `BIND(... AS ?type)` labels which pattern matched, so you can see the type in results
- Both must be in Japan (Q17)

**Expected output:** A mix of Shinto shrines and Buddhist temples in Japan, each labelled with its type.

## Query 5: MINUS -- Excluding Patterns

```sparql
SELECT ?shrine ?shrineLabel WHERE {
  ?shrine wdt:P31 wd:Q728 .
  MINUS { ?shrine wdt:P625 ?coords . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 20
```

**What this does:**
- Finds Shinto shrines that do **not** have coordinate data (P625)
- `MINUS` removes any shrine that matches the inner pattern
- This is useful for finding incomplete data -- shrines that need geo-coordinates added

**Expected output:** 20 Shinto shrines that are missing geographic coordinates on Wikidata.

## FILTER vs. MINUS vs. NOT EXISTS

These three can sometimes achieve similar results, but they behave differently:

```sparql
# MINUS: set subtraction (removes matching solution mappings)
MINUS { ?shrine wdt:P625 ?coords . }

# FILTER NOT EXISTS: inline check (filters row by row)
FILTER NOT EXISTS { ?shrine wdt:P625 ?coords . }

# FILTER with bound check (older style, avoid)
OPTIONAL { ?shrine wdt:P625 ?coords . }
FILTER(!BOUND(?coords))
```

`MINUS` and `FILTER NOT EXISTS` usually produce the same results, but can differ when variables overlap in subtle ways. Prefer `FILTER NOT EXISTS` when in doubt -- it is more predictable.

## Common FILTER Functions

| Function | Example | Purpose |
|----------|---------|---------|
| `CONTAINS` | `FILTER(CONTAINS(?label, "shrine"))` | Substring match |
| `STRSTARTS` | `FILTER(STRSTARTS(?label, "Mount"))` | Starts with |
| `STRENDS` | `FILTER(STRENDS(?label, "jinja"))` | Ends with |
| `REGEX` | `FILTER(REGEX(?label, "^[A-Z]", "i"))` | Regular expression |
| `LANG` | `FILTER(LANG(?label) = "ja")` | Language tag |
| `BOUND` | `FILTER(BOUND(?x))` | Variable is bound |
| `YEAR` | `FILTER(YEAR(?date) > 1900)` | Extract year from date |

## Exercises

1. **Find Shinto shrines whose English label starts with "A".** Use `STRSTARTS`.
2. **Find Japanese cities with population between 100,000 and 500,000.** Use two `FILTER` conditions.
3. **Find Shinto shrines that have coordinates but no image (P18).** Combine a normal triple pattern with `MINUS`.
4. **Use UNION to find items that are either Shinto shrines (Q728) or Shinto deities (Q26988).** Display the type alongside the label.
