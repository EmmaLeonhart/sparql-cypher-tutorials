# SPARQL Tutorial 4: Property Paths

## Overview

Property paths let you traverse chains of relationships in a single triple pattern. Instead of writing out each intermediate step, you can express "follow this property zero or more times" or "follow property A then property B." This is one of SPARQL's most powerful features for querying hierarchical and networked data.

## Path Operators

| Operator | Syntax | Meaning |
|----------|--------|---------|
| Sequence | `p1/p2` | Follow p1 then p2 |
| Alternative | `p1\|p2` | Follow p1 or p2 |
| Zero or more | `p*` | Follow p zero or more times (transitive closure) |
| One or more | `p+` | Follow p one or more times |
| Zero or one | `p?` | Follow p zero or one time (optional step) |
| Inverse | `^p` | Follow p in reverse direction |

## Query 1: Transitive Closure -- Class Hierarchy

```sparql
SELECT ?class ?classLabel WHERE {
  wd:Q728 wdt:P279* ?class .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

**What this does:**
- `wdt:P279*` -- follow "subclass of" (P279) zero or more times
- Starting from Q728 (Shinto shrine), find all parent classes up to the root
- `*` means: Q728 itself, its direct superclass, the superclass of that, and so on

**Expected output:** A chain like: Shinto shrine -> shrine -> religious building -> building -> architectural structure -> ... -> entity. The exact path depends on current Wikidata modeling.

## Query 2: One or More -- Finding All Subclasses

```sparql
SELECT ?subclass ?subclassLabel WHERE {
  ?subclass wdt:P279+ wd:Q728 .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
```

**What this does:**
- `wdt:P279+` -- follow "subclass of" one or more times, but in reverse (find things that are subclasses of Q728)
- This finds direct subclasses, sub-subclasses, etc.
- The `+` (not `*`) excludes Q728 itself from the results

**Expected output:** Shrine subtypes like: Inari shrine, Hachiman shrine, Tenjin shrine, Ichinomiya, Shikinai shrine (Shikinaisha), etc.

## Query 3: Sequence Paths

```sparql
SELECT ?shrine ?shrineLabel ?continentLabel WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          wdt:P17/wdt:P30 ?continent .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 30
```

**What this does:**
- `wdt:P17/wdt:P30` -- a sequence path: first follow P17 (country), then P30 (continent)
- Equivalent to writing two separate triple patterns:
  ```
  ?shrine wdt:P17 ?country .
  ?country wdt:P30 ?continent .
  ```
- The sequence path is more concise and avoids introducing the intermediate `?country` variable

**Expected output:** Shrines with their continent. Most will be in Asia (Japan), with a few in North America, South America, or Europe.

## Query 4: Alternative Paths

```sparql
SELECT ?item ?itemLabel ?locationLabel WHERE {
  ?item wdt:P31 wd:Q728 ;
        (wdt:P131|wdt:P276) ?location .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 25
```

**What this does:**
- `(wdt:P131|wdt:P276)` -- match P131 (located in admin entity) OR P276 (location)
- Some shrines use P131, others use P276 for their location
- The alternative path catches both without needing a `UNION`

**Expected output:** 25 shrines with their location, regardless of which property was used.

## Query 5: Inverse Paths and Combining Operators

```sparql
SELECT ?shrine ?shrineLabel ?related ?relatedLabel WHERE {
  ?shrine wdt:P31 wd:Q728 ;
          (wdt:P361|^wdt:P527) ?related .
  FILTER(?shrine != ?related)
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en,ja". }
}
LIMIT 30
```

**What this does:**
- `wdt:P361` = "part of" (shrine is part of something)
- `^wdt:P527` = inverse of "has part" (something has this shrine as a part -- reversed)
- These two often express the same relationship from opposite directions
- The combined path `(wdt:P361|^wdt:P527)` finds both
- `FILTER(?shrine != ?related)` avoids self-matches

**Expected output:** Shrines and the larger entities they belong to (shrine complexes, shrine networks, etc.)

## Performance Warning

Transitive property paths (`*` and `+`) can be expensive. On large datasets like Wikidata:

- `wdt:P279*` on a deep hierarchy is usually fine (class hierarchies are relatively shallow)
- `wdt:P131*` (administrative containment chains: city -> prefecture -> country) can be slow
- Always use `LIMIT` when experimenting with transitive paths
- Consider bounding the depth if performance is an issue:

```sparql
# Instead of unbounded P279*:
?x wdt:P279/wdt:P279? wd:Q728 .
# This limits to depth 1 or 2 only
```

## Exercises

1. **Find all superclasses of "human" (Q5)** using `wdt:P279*`. How deep does the hierarchy go?
2. **Find all subclasses of "religious building" (Q1370598)** using `wdt:P279+`. How many are there?
3. **Use a sequence path** to find Shinto shrines and the continent their country is on, but also include the country name in the output. (Hint: you will need to split the sequence path into two steps so you can capture the intermediate country variable.)
4. **Find items connected to Ise Grand Shrine (Q273196)** via either P361 (part of), P527 (has part), or P276 (location), using alternative paths.
