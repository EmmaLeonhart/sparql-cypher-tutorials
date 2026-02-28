# SPARQL Tutorial 6: Wikidata-Specific Features

## Overview

Wikidata's data model goes beyond simple triples. Every statement can have qualifiers, references, and a rank (preferred, normal, or deprecated). Understanding these layers is essential for accurate Wikidata queries. This tutorial covers the full statement model, the label service, and how to work with deprecated data.

## The Wikidata Statement Model

A simple truthy query like `?item wdt:P31 wd:Q728` only returns the best-ranked value. The full statement model looks like this:

```
Item  --p:P31-->  Statement Node  --ps:P31-->  Value
                                  --pq:P580->  start time (qualifier)
                                  --pq:P582->  end time (qualifier)
                                  --prov:wasDerivedFrom-->  Reference Node
                                  --wikibase:rank-->  Normal/Preferred/Deprecated
```

**Prefix guide:**
- `wdt:` -- truthy (shortcut, skips deprecated, returns best rank only)
- `p:` -- links item to statement node
- `ps:` -- links statement node to its value
- `pq:` -- links statement node to qualifier values
- `pr:` -- links reference node to reference values

## Query 1: Accessing Qualifiers

```sparql
SELECT ?shrine ?shrineLabel ?nameLabel ?startLabel WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          p:P1448 ?nameStatement .
  ?nameStatement ps:P1448 ?name .
  OPTIONAL { ?nameStatement pq:P580 ?start . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 20
```

**What this does:**
- `p:P1448` links the shrine to the statement node for "official name" (P1448)
- `ps:P1448` extracts the actual name value from the statement
- `pq:P580` extracts the "start time" qualifier (when this name began being used)
- This reveals the temporal dimension: shrines can have different names across history

**Expected output:** Shrines with their official names and (where available) the date that name was established.

## Query 2: Filtering by Rank

```sparql
SELECT ?item ?itemLabel ?nameLabel ?rank WHERE {
  VALUES ?item { wd:Q273196 }
  ?item p:P1448 ?stmt .
  ?stmt ps:P1448 ?name ;
        wikibase:rank ?rank .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
```

**What this does:**
- `wikibase:rank` retrieves the rank of each statement
- Possible values: `wikibase:PreferredRank`, `wikibase:NormalRank`, `wikibase:DeprecatedRank`
- Ise Grand Shrine (Q273196) may have multiple official names at different ranks

**Expected output:**

| item | itemLabel | nameLabel | rank |
|------|-----------|-----------|------|
| wd:Q273196 | Ise Grand Shrine | ... | wikibase:NormalRank |
| wd:Q273196 | Ise Grand Shrine | ... | wikibase:PreferredRank |

## Query 3: Finding Deprecated Statements

```sparql
SELECT ?item ?itemLabel ?valueLabel ?reasonLabel WHERE {
  ?item wdt:P31/wdt:P279* wd:Q728 .
  ?item p:P31 ?stmt .
  ?stmt wikibase:rank wikibase:DeprecatedRank ;
        ps:P31 ?value .
  OPTIONAL { ?stmt pq:P2241 ?reason . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 20
```

**What this does:**
- Finds Shinto shrine items that have deprecated P31 (instance of) statements
- `wikibase:DeprecatedRank` filters to only deprecated statements
- `pq:P2241` retrieves the "reason for deprecated rank" qualifier, if present
- Deprecated statements represent historical or incorrect classifications that editors have flagged

**Why this matters:** In the Shikinai Ronsha (Engishiki shrine) project, deprecated statements are used to preserve historical classifications while marking them as no longer current. A shrine might have been classified as a certain court rank in the Heian period, but that classification is deprecated because the Engishiki system is no longer active.

**Expected output:** Shrine items with their deprecated type classifications and (where available) the reason for deprecation.

## Query 4: Qualifiers with Roles

```sparql
SELECT ?shrine ?shrineLabel ?sameAsLabel ?roleLabel WHERE {
  ?shrine wdt:P31 wd:Q728 .
  ?shrine p:P460 ?stmt .
  ?stmt ps:P460 ?sameAs .
  OPTIONAL { ?stmt pq:P3831 ?role . }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 30
```

**What this does:**
- `P460` = "said to be the same as" -- links items that may be identical
- `pq:P3831` = "object of statement has role" qualifier -- specifies in what capacity the items are the same
- This pattern is used extensively in the Shikinai Ronsha project, where modern shrines claim to be the same as historical Engishiki shrines, but the identification is disputed

**Expected output:** Shrines linked to other items they are "said to be the same as," with the role qualifier showing the nature of the claim (e.g., "Disputed Shikinaisha").

## Query 5: References -- Who Said It?

```sparql
SELECT ?shrine ?shrineLabel ?valueLabel ?refURLLabel WHERE {
  VALUES ?shrine { wd:Q273196 wd:Q639683 wd:Q617572 }
  ?shrine p:P31 ?stmt .
  ?stmt ps:P31 ?value ;
        prov:wasDerivedFrom ?ref .
  ?ref pr:P854 ?refURL .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
```

**What this does:**
- `prov:wasDerivedFrom` links the statement to its reference node
- `pr:P854` extracts the reference URL (P854 = reference URL)
- This reveals the sources behind Wikidata claims

**Expected output:** Instance-of claims for the three major shrines, along with URLs to the sources that support those claims.

## The Label Service in Depth

The Wikidata label service deserves special attention because it simplifies queries enormously.

```sparql
# Basic usage: append "Label" to any variable name
SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja,de,fr". }
```

**Rules:**
- For a variable `?foo`, it automatically creates `?fooLabel` and `?fooDescription`
- The language parameter is a fallback chain: try English first, then Japanese, then German, etc.
- It works on any variable that holds a Wikidata entity URI
- It does NOT work on literals (strings, numbers, dates)

**Common pitfall:** If you name your variable `?itemLabel` directly (from `rdfs:label`), the service will try to create `?itemLabelLabel`. Use different variable names for raw labels vs. service-generated labels.

## Combining Everything: Full Statement Analysis

```sparql
SELECT
  ?shrine ?shrineLabel
  ?propLabel
  ?valueLabel
  ?rank
  (GROUP_CONCAT(DISTINCT ?qualLabel; SEPARATOR="; ") AS ?qualifiers)
WHERE {
  VALUES ?shrine { wd:Q273196 }
  VALUES ?prop { wdt:P31 wdt:P361 wdt:P1343 }
  ?shrine ?propDirect ?anyVal .
  BIND(IRI(REPLACE(STR(?propDirect), "http://www.wikidata.org/prop/direct/", "http://www.wikidata.org/prop/")) AS ?propFull)
  ?shrine ?propFull ?stmt .
  ?stmt ?psP ?value ;
        wikibase:rank ?rank .
  FILTER(STRSTARTS(STR(?psP), "http://www.wikidata.org/prop/statement/"))
  OPTIONAL {
    ?stmt ?qualProp ?qualVal .
    FILTER(STRSTARTS(STR(?qualProp), "http://www.wikidata.org/prop/qualifier/"))
    ?qualEntity wikibase:directClaim ?qualProp .
    SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
  }
  BIND(IRI(REPLACE(STR(?propDirect), "direct/", "")) AS ?prop)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
GROUP BY ?shrine ?shrineLabel ?propLabel ?valueLabel ?rank
ORDER BY ?propLabel ?rank
```

**Note:** This is an advanced query that dynamically explores all statements, qualifiers, and ranks for specified properties. It may be slow. Use it as a reference pattern, not for production use.

## Exercises

1. **Find all P31 (instance of) statements for Meiji Shrine (Q617572)**, including their rank. Are any deprecated?
2. **Query P1448 (official name) statements with start time and end time qualifiers** for any shrine that has them. This reveals name changes over time.
3. **Find shrines that have references on their P31 statements.** How many different reference URLs are used? (Use COUNT and GROUP BY on the reference URL.)
4. **Write a query to find deprecated P361 (part of) statements** across all Shinto shrines. Include the reason for deprecation (P2241) if available.
