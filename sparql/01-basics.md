# SPARQL Tutorial 1: Basics

## What is SPARQL?

SPARQL (SPARQL Protocol and RDF Query Language) is the standard query language for RDF (Resource Description Framework) data. If SQL queries tables, SPARQL queries graphs. The fundamental unit of data in RDF is the **triple**: a subject-predicate-object statement.

```
Subject  --  Predicate  -->  Object
Q728     --  P31        -->  Q2122480
(Shinto shrine) -- (instance of) --> (sacred architecture)
```

Every fact in Wikidata is stored as a triple. When you query Wikidata, you are pattern-matching against millions of these triples.

## The SPARQL Endpoint

All queries in this tutorial can be run directly at:
**https://query.wikidata.org/**

Paste the query, click the blue play button, and see results. No setup needed.

## Query 1: Your First Query -- Select a Single Item

```sparql
SELECT ?item ?itemLabel WHERE {
  VALUES ?item { wd:Q728 }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

**What this does:**
- `SELECT ?item ?itemLabel` -- return these two variables
- `VALUES ?item { wd:Q728 }` -- bind `?item` to Wikidata item Q728
- `wd:Q728` -- the `wd:` prefix means "Wikidata entity"
- The `SERVICE wikibase:label` block automatically fetches English labels

**Expected output:**

| item | itemLabel |
|------|-----------|
| wd:Q728 | Shinto shrine |

## Query 2: Find Instances of a Class

```sparql
SELECT ?shrine ?shrineLabel WHERE {
  ?shrine wdt:P31 wd:Q728 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 10
```

**What this does:**
- `?shrine wdt:P31 wd:Q728` -- find all items where "instance of" (P31) is "Shinto shrine" (Q728)
- `wdt:` prefix means "Wikidata truthy statement" (the current, best-ranked value)
- `LIMIT 10` -- only return 10 results (Wikidata has thousands of shrines)
- The label service fetches English first, then Japanese if no English label exists

**Expected output:** 10 Shinto shrines with their labels, such as Ise Grand Shrine, Fushimi Inari-taisha, Meiji Shrine, etc.

## Query 3: Multiple Properties

```sparql
SELECT ?shrine ?shrineLabel ?countryLabel ?coords WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P17 ?country ;
          wdt:P625 ?coords .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 20
```

**What this does:**
- The semicolon `;` continues with the same subject (`?shrine`) but a new predicate
- `wdt:P17` -- country (P17)
- `wdt:P625` -- coordinate location (P625)
- This only returns shrines that have both a country AND coordinates

**Expected output:** 20 shrines with their country and geographic coordinates. Most will be in Japan. Coordinates appear as `Point(longitude latitude)`.

## Query 4: Retrieve Descriptions

```sparql
SELECT ?item ?itemLabel ?itemDescription WHERE {
  VALUES ?item { wd:Q728 wd:Q5 wd:Q515 wd:Q6256 }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

**What this does:**
- `VALUES` lets you list multiple items to query at once
- `?itemDescription` is automatically populated by the label service, just like `?itemLabel`
- Q728 = Shinto shrine, Q5 = human, Q515 = city, Q6256 = country

**Expected output:**

| item | itemLabel | itemDescription |
|------|-----------|-----------------|
| wd:Q728 | Shinto shrine | Japanese shrine of the Shinto religion |
| wd:Q5 | human | any member of Homo sapiens... |
| wd:Q515 | city | large and permanent human settlement |
| wd:Q6256 | country | distinct region in geography... |

## Query 5: Count Items

```sparql
SELECT (COUNT(?shrine) AS ?total) WHERE {
  ?shrine wdt:P31 wd:Q728 .
}
```

**What this does:**
- `COUNT(?shrine)` counts all matching items
- `AS ?total` names the output column
- This counts every item on Wikidata that is an "instance of" Shinto shrine

**Expected output:** A single number -- typically around 3,000-5,000 depending on how many shrines have been catalogued.

## Key Prefixes Reference

| Prefix | Expansion | Use |
|--------|-----------|-----|
| `wd:` | `http://www.wikidata.org/entity/` | Entities (items and properties) |
| `wdt:` | `http://www.wikidata.org/prop/direct/` | Truthy statements (simple values) |
| `p:` | `http://www.wikidata.org/prop/` | Statement nodes (for qualifiers) |
| `ps:` | `http://www.wikidata.org/prop/statement/` | Statement values |
| `pq:` | `http://www.wikidata.org/prop/qualifier/` | Qualifier values |

You will use `wd:` and `wdt:` constantly. The `p:/ps:/pq:` prefixes become essential when working with qualifiers (covered in Tutorial 6).

## Exercises

1. **Find 15 items that are instances of "city" (Q515).** Display their labels.
2. **Find items that are instances of "country" (Q6256)** and also have a population (P1082). Display the country label and population. Limit to 10.
3. **Count how many items on Wikidata are instances of "human" (Q5).** (Warning: this is a large number and may take a few seconds.)
4. **Pick any Wikidata item you find interesting** (search at wikidata.org) and write a query that retrieves its label, description, and one property of your choice.
