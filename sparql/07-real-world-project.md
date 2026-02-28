# SPARQL Tutorial 7: Real-World Project -- Analyzing Shinto Shrines in Wikidata

## Project Overview

This tutorial applies everything from the previous six lessons to a real research project: a comprehensive analysis of Shinto shrines as represented in Wikidata. You will explore shrine distribution by province, examine the Engishiki shrine classification system, map shrine networks, and assess data quality.

This mirrors actual work done on Wikidata to model the Shikinai Ronsha (shrines listed in the Engishiki Jinmyocho, the 10th-century register of official shrines).

## Part 1: Shrine Census

### Query 1.1: Total Shrine Count by Type

```sparql
SELECT ?typeLabel (COUNT(?shrine) AS ?count) WHERE {
  ?shrine wdt:P31 ?type .
  ?type wdt:P279* wd:Q728 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
GROUP BY ?typeLabel
ORDER BY DESC(?count)
```

**Purpose:** Get a complete breakdown of shrine subtypes on Wikidata. This tells us how the community has classified shrines -- not just "Shinto shrine" but Inari shrines, Hachiman shrines, ichinomiya, etc.

**Expected output:** A ranked list of shrine types. "Shinto shrine" (Q728) will dominate, followed by specific subtypes.

### Query 1.2: Shrines by Japanese Prefecture

```sparql
SELECT ?prefLabel (COUNT(?shrine) AS ?count) WHERE {
  ?shrine wdt:P31/wdt:P279* wd:Q728 ;
          wdt:P131* ?pref .
  ?pref wdt:P31 wd:Q50337 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
GROUP BY ?prefLabel
ORDER BY DESC(?count)
```

**Purpose:** Geographic distribution of shrines across Japan's 47 prefectures. The `wdt:P131*` path follows the administrative hierarchy (city -> prefecture) so shrines at any admin level get counted in their prefecture.

**Note:** The transitive `P131*` can be slow. For faster results, limit the depth:

```sparql
?shrine (wdt:P131/wdt:P131?) ?pref .
```

## Part 2: The Engishiki System

The Engishiki Jinmyocho (927 CE) listed 2,861 shrines in 3,132 entries. These "Shikinai shrines" were the officially recognized shrines of the Heian court. On Wikidata, this system is modeled with several related items:

- **Q135022904** -- Shikinai Ronsha (the individual entry in the register)
- **Q728** -- Shinto shrine (the physical shrine)
- **P460** -- "said to be the same as" (linking a modern shrine to its Engishiki entry)
- **P361** -- "part of" (linking to the Engishiki Jinmyocho)
- **P1343** -- "described by source" (referencing the Engishiki)

### Query 2.1: Find All Shikinai Ronsha Entries

```sparql
SELECT ?entry ?entryLabel ?sameAsLabel WHERE {
  ?entry wdt:P31 wd:Q135022904 .
  OPTIONAL { ?entry wdt:P460 ?sameAs . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 50
```

**Purpose:** List Shikinai Ronsha entries and their modern shrine equivalents (linked via P460). Not all entries have a confirmed modern equivalent -- some ancient shrines have been lost or merged.

### Query 2.2: Disputed Identifications

```sparql
SELECT ?entry ?entryLabel
       (COUNT(?candidate) AS ?candidateCount)
       (GROUP_CONCAT(DISTINCT ?candidateLabel; SEPARATOR=", ") AS ?candidates)
WHERE {
  ?entry wdt:P31 wd:Q135022904 .
  ?entry p:P460 ?stmt .
  ?stmt ps:P460 ?candidate .
  ?candidate rdfs:label ?candidateLabel .
  FILTER(LANG(?candidateLabel) = "en")
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
GROUP BY ?entry ?entryLabel
HAVING(COUNT(?candidate) > 1)
ORDER BY DESC(?candidateCount)
```

**Purpose:** Find Engishiki entries where multiple modern shrines claim to be the same shrine. These are scholarly disputes: when the original shrine's identity is uncertain, multiple candidates exist. The more candidates, the more contested the identification.

**Expected output:** Entries with 2+ candidates listed, showing which historical shrines have disputed identifications.

### Query 2.3: Engishiki Properties -- Deprecated vs. Active

```sparql
SELECT
  ?shrine ?shrineLabel
  ?propLabel
  ?valueLabel
  ?rank
WHERE {
  ?shrine wdt:P31 wd:Q728 .
  ?shrine p:P361 ?stmt .
  ?stmt ps:P361 ?value ;
        wikibase:rank ?rank .
  FILTER(?value = wd:Q107387421)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
ORDER BY ?rank
LIMIT 30
```

**Purpose:** Find shrines with "part of: Engishiki Jinmyocho" (Q107387421) statements and check their rank. In the deprecation workflow, these statements get deprecated because the relationship between a modern shrine and the historical register is contested, not a simple "part of."

## Part 3: Shrine Networks

### Query 3.1: Shrine Lineage -- Main and Branch Shrines

```sparql
SELECT ?main ?mainLabel ?branch ?branchLabel WHERE {
  ?branch wdt:P31/wdt:P279* wd:Q728 ;
          wdt:P749 ?main .
  ?main wdt:P31/wdt:P279* wd:Q728 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 50
```

**Purpose:** `P749` (parent organization) connects branch shrines to their main shrine. This reveals shrine networks: Fushimi Inari-taisha is the head of roughly 30,000 Inari shrines nationwide (though only a fraction are on Wikidata).

### Query 3.2: Deity-Shrine Connections

```sparql
SELECT ?deityLabel (COUNT(?shrine) AS ?shrineCount) WHERE {
  ?shrine wdt:P31/wdt:P279* wd:Q728 ;
          wdt:P5607 ?deity .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
GROUP BY ?deityLabel
ORDER BY DESC(?shrineCount)
LIMIT 20
```

**Purpose:** `P5607` (patron saint / dedicatee) links shrines to the kami (deity) they enshrine. This query finds the most commonly enshrined deities across Wikidata.

**Expected output:** Top deities by shrine count. Inari Okami, Hachiman, Amaterasu, Susanoo, and Tenjin are likely to appear.

## Part 4: Data Quality Assessment

### Query 4.1: Completeness Report

```sparql
SELECT
  (COUNT(?shrine) AS ?total)
  (COUNT(?coords) AS ?hasCoords)
  (COUNT(?image) AS ?hasImage)
  (COUNT(?inception) AS ?hasFounding)
  (COUNT(?website) AS ?hasWebsite)
WHERE {
  ?shrine wdt:P31 wd:Q728 .
  OPTIONAL { ?shrine wdt:P625 ?coords . }
  OPTIONAL { ?shrine wdt:P18 ?image . }
  OPTIONAL { ?shrine wdt:P571 ?inception . }
  OPTIONAL { ?shrine wdt:P856 ?website . }
}
```

**Purpose:** A single-row dashboard showing how complete Shinto shrine data is on Wikidata. What percentage have coordinates? Images? Founding dates?

**Expected output:**

| total | hasCoords | hasImage | hasFounding | hasWebsite |
|-------|-----------|----------|-------------|------------|
| ~3000+ | ~2000+ | ~800+ | ~300+ | ~200+ |

Coordinates are the most complete; founding dates and websites are sparse.

### Query 4.2: Shrines Missing Key Properties

```sparql
SELECT ?shrine ?shrineLabel
  (BOUND(?coords) AS ?hasCoords)
  (BOUND(?image) AS ?hasImage)
  (BOUND(?desc) AS ?hasEnDesc)
WHERE {
  ?shrine wdt:P31 wd:Q728 .
  OPTIONAL { ?shrine wdt:P625 ?coords . }
  OPTIONAL { ?shrine wdt:P18 ?image . }
  OPTIONAL { ?shrine schema:description ?desc . FILTER(LANG(?desc) = "en") }
  FILTER(!BOUND(?coords) || !BOUND(?image) || !BOUND(?desc))
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 30
```

**Purpose:** Find shrines that are missing coordinates, images, or English descriptions. This is a practical tool for Wikidata editors looking for items to improve.

## Part 5: Visualization

The Wikidata Query Service supports several result visualizations. Add `#defaultView:` comments to trigger them.

### Query 5.1: Map of Shinto Shrines

```sparql
#defaultView:Map
SELECT ?shrine ?shrineLabel ?coords ?image WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P625 ?coords .
  OPTIONAL { ?shrine wdt:P18 ?image . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 500
```

**Purpose:** Renders an interactive map of shrines. Each dot is a shrine. Click for details. The `?image` variable adds thumbnail images to the popups.

### Query 5.2: Timeline of Shrine Founding

```sparql
#defaultView:Timeline
SELECT ?shrine ?shrineLabel ?inception WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P571 ?inception .
  FILTER(YEAR(?inception) > 0)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
```

**Purpose:** Renders a timeline visualization. You can see the historical pattern of shrine founding: clusters in the Nara period (710-794), Heian period (794-1185), and Meiji era (1868-1912).

### Query 5.3: Bubble Chart of Shrines per Prefecture

```sparql
#defaultView:BubbleChart
SELECT ?prefLabel (COUNT(?shrine) AS ?count) WHERE {
  ?shrine wdt:P31/wdt:P279* wd:Q728 ;
          wdt:P131* ?pref .
  ?pref wdt:P31 wd:Q50337 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
GROUP BY ?prefLabel
```

**Purpose:** Visual comparison of shrine density across prefectures.

## Exercises

1. **Build a complete report for a single prefecture.** Pick one (e.g., Kyoto, Q120730) and query: total shrines, shrines with coordinates, shrines with images, top 5 enshrined deities, and list all shrines with founding dates.
2. **Find all Shikinai Ronsha entries that do NOT have a modern shrine linked via P460.** These are "lost" shrines. How many are there?
3. **Create a map** showing both Shinto shrines (Q728) and Buddhist temples (Q44539) in a single prefecture, using different colors. (Hint: use `BIND` to create a color variable based on the type.)
4. **Write a data quality improvement query** that finds shrines with Japanese labels but no English labels. These are candidates for translation.
5. **Investigate the P460 (said to be the same as) network.** Find cases where shrine A says it is the same as entry X, but shrine B also says it is the same as entry X. These are the core scholarly disputes in Shikinai Ronsha studies.
