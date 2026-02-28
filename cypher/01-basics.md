# Cypher Tutorial 1: Basics

## What is Cypher?

Cypher is the query language for Neo4j and other labeled property graph databases. Where SPARQL works with triples (subject-predicate-object), Cypher works with nodes and relationships drawn as ASCII art patterns.

```
(node)-[:RELATIONSHIP]->(otherNode)
```

This visual syntax is Cypher's signature feature: you draw the pattern you want to find, and the database matches it.

## Setup

To follow along, you can use:
- **Neo4j Sandbox:** https://sandbox.neo4j.com/ (free, pre-loaded datasets)
- **Neo4j AuraDB Free:** https://neo4j.com/cloud/aura-free/ (free cloud instance)
- **Neo4j Desktop:** https://neo4j.com/download/ (local installation)

The examples below use a hypothetical shrine/temple knowledge graph. To create the sample data, run the setup query in Query 1 first.

## Query 1: CREATE -- Building Sample Data

```cypher
// Create shrine and deity nodes
CREATE (ise:Shrine {name: 'Ise Grand Shrine', founded: -4, prefecture: 'Mie', type: 'Grand Shrine'})
CREATE (fushimi:Shrine {name: 'Fushimi Inari-taisha', founded: 711, prefecture: 'Kyoto', type: 'Inari Shrine'})
CREATE (meiji:Shrine {name: 'Meiji Shrine', founded: 1920, prefecture: 'Tokyo', type: 'Imperial Shrine'})
CREATE (izumo:Shrine {name: 'Izumo-taisha', founded: -660, prefecture: 'Shimane', type: 'Taisha'})
CREATE (kasuga:Shrine {name: 'Kasuga Grand Shrine', founded: 768, prefecture: 'Nara', type: 'Grand Shrine'})

CREATE (amaterasu:Deity {name: 'Amaterasu', domain: 'Sun'})
CREATE (inari:Deity {name: 'Inari Okami', domain: 'Rice, Foxes'})
CREATE (okuninushi:Deity {name: 'Okuninushi', domain: 'Nation-building'})
CREATE (takemikazuchi:Deity {name: 'Takemikazuchi', domain: 'Thunder, Swords'})

CREATE (mie:Prefecture {name: 'Mie'})
CREATE (kyoto:Prefecture {name: 'Kyoto'})
CREATE (tokyo:Prefecture {name: 'Tokyo'})
CREATE (shimane:Prefecture {name: 'Shimane'})
CREATE (nara:Prefecture {name: 'Nara'})

// Create relationships
CREATE (ise)-[:ENSHRINES]->(amaterasu)
CREATE (fushimi)-[:ENSHRINES]->(inari)
CREATE (izumo)-[:ENSHRINES]->(okuninushi)
CREATE (kasuga)-[:ENSHRINES]->(takemikazuchi)

CREATE (ise)-[:LOCATED_IN]->(mie)
CREATE (fushimi)-[:LOCATED_IN]->(kyoto)
CREATE (meiji)-[:LOCATED_IN]->(tokyo)
CREATE (izumo)-[:LOCATED_IN]->(shimane)
CREATE (kasuga)-[:LOCATED_IN]->(nara)

CREATE (fushimi)-[:BRANCH_OF]->(ise)
CREATE (meiji)-[:INFLUENCED_BY]->(ise)

RETURN 'Sample data created' AS status
```

**What this does:**
- Creates 5 shrine nodes, 4 deity nodes, and 5 prefecture nodes
- Each node has a label (Shrine, Deity, Prefecture) and properties (name, founded, etc.)
- Creates relationships: ENSHRINES, LOCATED_IN, BRANCH_OF, INFLUENCED_BY
- Run this once to set up the example database

## Query 2: MATCH and RETURN -- Your First Read Query

```cypher
MATCH (s:Shrine)
RETURN s.name AS shrine, s.founded AS founded, s.prefecture AS prefecture
ORDER BY s.founded
```

**What this does:**
- `MATCH (s:Shrine)` -- find all nodes with the label `Shrine`, bind each to variable `s`
- `RETURN s.name` -- return the `name` property
- `ORDER BY s.founded` -- sort by founding year

**Expected output:**

| shrine | founded | prefecture |
|--------|---------|-----------|
| Izumo-taisha | -660 | Shimane |
| Ise Grand Shrine | -4 | Mie |
| Kasuga Grand Shrine | 768 | Nara |
| Fushimi Inari-taisha | 711 | Kyoto |
| Meiji Shrine | 1920 | Tokyo |

## Query 3: Pattern Matching -- Following Relationships

```cypher
MATCH (s:Shrine)-[:ENSHRINES]->(d:Deity)
RETURN s.name AS shrine, d.name AS deity, d.domain AS domain
```

**What this does:**
- `(s:Shrine)-[:ENSHRINES]->(d:Deity)` -- find shrine nodes connected to deity nodes via the ENSHRINES relationship
- The arrow `->` indicates direction: the shrine enshrines the deity, not the other way
- Only shrines with an ENSHRINES relationship appear; Meiji Shrine (no deity linked) is excluded

**Expected output:**

| shrine | deity | domain |
|--------|-------|--------|
| Ise Grand Shrine | Amaterasu | Sun |
| Fushimi Inari-taisha | Inari Okami | Rice, Foxes |
| Izumo-taisha | Okuninushi | Nation-building |
| Kasuga Grand Shrine | Takemikazuchi | Thunder, Swords |

## Query 4: Multi-Hop Patterns

```cypher
MATCH (s:Shrine)-[:LOCATED_IN]->(p:Prefecture)
OPTIONAL MATCH (s)-[:ENSHRINES]->(d:Deity)
RETURN p.name AS prefecture, s.name AS shrine, d.name AS deity
ORDER BY p.name
```

**What this does:**
- First `MATCH` finds shrines and their prefectures (required)
- `OPTIONAL MATCH` adds the enshrined deity if one exists (not required)
- This is Cypher's equivalent of a LEFT JOIN or SPARQL's `OPTIONAL`
- Meiji Shrine appears with `null` for deity

**Expected output:**

| prefecture | shrine | deity |
|-----------|--------|-------|
| Kyoto | Fushimi Inari-taisha | Inari Okami |
| Mie | Ise Grand Shrine | Amaterasu |
| Nara | Kasuga Grand Shrine | Takemikazuchi |
| Shimane | Izumo-taisha | Okuninushi |
| Tokyo | Meiji Shrine | null |

## Query 5: Counting and Aggregation

```cypher
MATCH (s:Shrine)-[:LOCATED_IN]->(p:Prefecture)
RETURN p.name AS prefecture, COUNT(s) AS shrineCount
ORDER BY shrineCount DESC
```

**What this does:**
- Counts shrines per prefecture
- `COUNT(s)` aggregates automatically when non-aggregated fields are in `RETURN`
- With our sample data, each prefecture has exactly 1 shrine

**Expected output:**

| prefecture | shrineCount |
|-----------|-------------|
| Kyoto | 1 |
| Mie | 1 |
| Nara | 1 |
| Shimane | 1 |
| Tokyo | 1 |

## Cypher vs. SPARQL: Key Differences

| Aspect | SPARQL | Cypher |
|--------|--------|--------|
| Data model | RDF triples | Labeled property graph |
| Pattern syntax | `?s ?p ?o .` | `(s)-[:REL]->(o)` |
| Schema | Schema-free (any URI as predicate) | Labels and relationship types |
| Properties | Separate triples | On nodes and relationships |
| Standards | W3C standard | Neo4j (GQL standard emerging) |
| Best for | Linked open data, federation | Application databases, traversals |

## Exercises

1. **Find all deity nodes.** Return their name and domain.
2. **Find shrines founded before the year 800.** Return their name and founding year.
3. **Find which shrine Fushimi Inari-taisha is a branch of.** Use the BRANCH_OF relationship.
4. **Write a query that returns all relationships** from Ise Grand Shrine to any connected node. (Hint: use `(ise)-[r]->(target)` with a WHERE clause on the name.)
5. **Add a new shrine node** for Toshogu (founded 1617, prefecture Tochigi, type Toshogu) and create an ENSHRINES relationship to a new deity node for Tokugawa Ieyasu.
