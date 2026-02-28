# Cypher Tutorial 2: Filtering and Pattern Matching

## Overview

Cypher's `WHERE` clause filters results using conditions on properties, relationships, and patterns. Cypher also supports inline pattern matching that goes beyond what a simple WHERE can express.

**Prerequisite:** Run the CREATE statements from Tutorial 1 first.

## Query 1: WHERE with Property Conditions

```cypher
MATCH (s:Shrine)
WHERE s.founded < 800
RETURN s.name AS shrine, s.founded AS founded, s.type AS type
ORDER BY s.founded
```

**What this does:**
- Finds all shrine nodes where the `founded` property is before 800 CE
- Comparison operators work as expected: `<`, `>`, `<=`, `>=`, `=`, `<>`

**Expected output:**

| shrine | founded | type |
|--------|---------|------|
| Izumo-taisha | -660 | Taisha |
| Ise Grand Shrine | -4 | Grand Shrine |
| Fushimi Inari-taisha | 711 | Inari Shrine |
| Kasuga Grand Shrine | 768 | Grand Shrine |

## Query 2: String Matching

```cypher
MATCH (s:Shrine)
WHERE s.name CONTAINS 'Grand'
RETURN s.name AS shrine, s.prefecture AS prefecture
```

**What this does:**
- `CONTAINS` performs a substring search (case-sensitive)
- Also available: `STARTS WITH`, `ENDS WITH`
- For regex: `WHERE s.name =~ '.*[Ss]hrine.*'`

**Expected output:**

| shrine | prefecture |
|--------|-----------|
| Ise Grand Shrine | Mie |
| Kasuga Grand Shrine | Nara |

### Other string predicates:

```cypher
// Starts with
WHERE s.name STARTS WITH 'Ise'

// Ends with
WHERE s.name ENDS WITH 'taisha'

// Regular expression (case-insensitive)
WHERE s.name =~ '(?i).*shrine.*'
```

## Query 3: Pattern-Based Filtering

```cypher
MATCH (s:Shrine)
WHERE EXISTS {
  MATCH (s)-[:ENSHRINES]->(:Deity {domain: 'Sun'})
}
RETURN s.name AS shrine, s.type AS type
```

**What this does:**
- `EXISTS { ... }` checks whether a pattern exists without returning it
- Finds shrines that enshrine a deity whose domain is "Sun"
- This is Cypher's equivalent of SPARQL's `FILTER EXISTS`

**Expected output:**

| shrine | type |
|--------|------|
| Ise Grand Shrine | Grand Shrine |

## Query 4: NOT EXISTS and Negation

```cypher
MATCH (s:Shrine)
WHERE NOT EXISTS {
  MATCH (s)-[:ENSHRINES]->(:Deity)
}
RETURN s.name AS shrine, s.type AS type
```

**What this does:**
- Finds shrines that do NOT have any ENSHRINES relationship
- Equivalent to SPARQL's `FILTER NOT EXISTS` or `MINUS`
- In our sample data, Meiji Shrine has no deity linked

**Expected output:**

| shrine | type |
|--------|------|
| Meiji Shrine | Imperial Shrine |

## Query 5: Combining Conditions with AND, OR, NOT

```cypher
MATCH (s:Shrine)-[:LOCATED_IN]->(p:Prefecture)
WHERE (s.founded < 800 OR s.type = 'Imperial Shrine')
  AND p.name <> 'Shimane'
RETURN s.name AS shrine, s.founded AS founded, p.name AS prefecture
ORDER BY s.founded
```

**What this does:**
- `OR` combines two conditions: ancient OR imperial
- `AND` adds a constraint: not in Shimane
- `<>` is "not equal to"
- Parentheses control precedence (just like in most programming languages)

**Expected output:**

| shrine | founded | prefecture |
|--------|---------|-----------|
| Ise Grand Shrine | -4 | Mie |
| Fushimi Inari-taisha | 711 | Kyoto |
| Kasuga Grand Shrine | 768 | Nara |
| Meiji Shrine | 1920 | Tokyo |

## Additional Filtering Techniques

### IN operator:

```cypher
MATCH (s:Shrine)
WHERE s.prefecture IN ['Kyoto', 'Nara', 'Mie']
RETURN s.name, s.prefecture
```

### IS NULL / IS NOT NULL:

```cypher
MATCH (s:Shrine)
OPTIONAL MATCH (s)-[:ENSHRINES]->(d:Deity)
WHERE d IS NULL
RETURN s.name AS shrineWithoutDeity
```

### Filtering on relationship properties:

```cypher
// If relationships had properties:
MATCH (a:Shrine)-[r:BRANCH_OF]->(b:Shrine)
WHERE r.since IS NOT NULL
RETURN a.name, b.name, r.since
```

## SPARQL Comparison

| SPARQL | Cypher |
|--------|--------|
| `FILTER(?x > 100)` | `WHERE x > 100` |
| `FILTER(CONTAINS(?label, "text"))` | `WHERE n.label CONTAINS 'text'` |
| `FILTER(LANG(?l) = "en")` | N/A (properties have no language tags) |
| `FILTER NOT EXISTS { ... }` | `WHERE NOT EXISTS { MATCH ... }` |
| `FILTER(?x IN (1, 2, 3))` | `WHERE x IN [1, 2, 3]` |
| `OPTIONAL { } FILTER(!BOUND(?x))` | `OPTIONAL MATCH ... WHERE x IS NULL` |

## Exercises

1. **Find all shrines whose name ends with "taisha"** (a term meaning "grand shrine" in a different tradition from "jingu").
2. **Find shrines that are located in prefectures that also contain the word "Shrine" in another node's name.** (Hint: use a pattern in the WHERE clause.)
3. **Find deities whose domain contains a comma** (meaning they have multiple domains listed). Use `CONTAINS`.
4. **Find all shrines NOT founded in the Meiji era** (after 1868). Combine `NOT` with a numeric comparison.
