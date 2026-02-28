# Cypher Tutorial 3: Paths and Traversals

## Overview

Graph databases excel at path queries: finding connections between nodes across multiple hops. Cypher provides dedicated syntax for variable-length patterns, shortest paths, and path inspection. These capabilities have no direct equivalent in SQL and are one of the primary reasons to use a graph database.

**Prerequisite:** Run the setup from Tutorial 1. Then run this additional data to create a richer network:

```cypher
// Add more shrines and relationships for path examples
CREATE (atsuta:Shrine {name: 'Atsuta Shrine', founded: 113, prefecture: 'Aichi', type: 'Grand Shrine'})
CREATE (tsurugaoka:Shrine {name: 'Tsurugaoka Hachimangu', founded: 1063, prefecture: 'Kanagawa', type: 'Hachiman Shrine'})
CREATE (usa:Shrine {name: 'Usa Jingu', founded: 725, prefecture: 'Oita', type: 'Hachiman Shrine'})
CREATE (hachiman:Deity {name: 'Hachiman', domain: 'War, Archery'})

CREATE (tsurugaoka)-[:ENSHRINES]->(hachiman)
CREATE (usa)-[:ENSHRINES]->(hachiman)
CREATE (tsurugaoka)-[:BRANCH_OF]->(usa)
CREATE (usa)-[:INFLUENCED_BY]->(atsuta)

// Connect to existing nodes
MATCH (ise:Shrine {name: 'Ise Grand Shrine'})
MATCH (atsuta)
WHERE atsuta.name = 'Atsuta Shrine'
CREATE (atsuta)-[:INFLUENCED_BY]->(ise)
```

## Query 1: Variable-Length Patterns

```cypher
MATCH path = (s:Shrine)-[:INFLUENCED_BY*1..3]->(t:Shrine)
RETURN s.name AS source, t.name AS target, length(path) AS hops
ORDER BY hops DESC
```

**What this does:**
- `[:INFLUENCED_BY*1..3]` -- follow INFLUENCED_BY relationships 1 to 3 hops
- `*1..3` means minimum 1 hop, maximum 3 hops
- `length(path)` returns the number of relationships in the path
- This finds both direct and indirect influence chains

**Expected output:**

| source | target | hops |
|--------|--------|------|
| Usa Jingu | Ise Grand Shrine | 2 |
| Atsuta Shrine | Ise Grand Shrine | 1 |
| Meiji Shrine | Ise Grand Shrine | 1 |
| Usa Jingu | Atsuta Shrine | 1 |

## Query 2: Shortest Path

```cypher
MATCH path = shortestPath(
  (start:Shrine {name: 'Tsurugaoka Hachimangu'})-[*..5]-(end:Shrine {name: 'Ise Grand Shrine'})
)
RETURN [n IN nodes(path) | n.name] AS nodeNames,
       length(path) AS hops
```

**What this does:**
- `shortestPath(...)` finds the shortest path between two nodes
- `[*..5]` allows any relationship type, up to 5 hops (the `..5` is a safety limit)
- No arrow direction (`-[*..5]-` not `-[*..5]->`) means relationships can be traversed in either direction
- `nodes(path)` extracts all nodes in the path as a list
- The list comprehension `[n IN nodes(path) | n.name]` maps each node to its name

**Expected output:**

| nodeNames | hops |
|-----------|------|
| [Tsurugaoka Hachimangu, Usa Jingu, Atsuta Shrine, Ise Grand Shrine] | 3 |

## Query 3: All Shortest Paths

```cypher
MATCH path = allShortestPaths(
  (start:Shrine {name: 'Tsurugaoka Hachimangu'})-[*..6]-(end:Shrine {name: 'Ise Grand Shrine'})
)
RETURN [n IN nodes(path) | n.name] AS route,
       [r IN relationships(path) | type(r)] AS relTypes,
       length(path) AS hops
```

**What this does:**
- `allShortestPaths(...)` returns ALL paths of the shortest length (there may be several)
- `relationships(path)` extracts the relationships
- `type(r)` gets the relationship type as a string
- If there are multiple routes of equal length, all are returned

**Expected output:** One or more routes, each showing the node sequence and relationship types traversed.

## Query 4: Path Filtering

```cypher
MATCH path = (s:Shrine)-[:BRANCH_OF|INFLUENCED_BY*1..4]->(t:Shrine)
WHERE ALL(n IN nodes(path) WHERE n.founded < 1900)
RETURN s.name AS source, t.name AS target,
       [n IN nodes(path) | n.name] AS route
```

**What this does:**
- `[:BRANCH_OF|INFLUENCED_BY*1..4]` -- follow either relationship type, 1-4 hops
- `ALL(n IN nodes(path) WHERE ...)` -- every node in the path must satisfy the condition
- Here: every shrine in the chain must have been founded before 1900
- This excludes paths that go through Meiji Shrine (founded 1920)

**Other path predicates:**
- `ALL(x IN list WHERE condition)` -- every element satisfies condition
- `ANY(x IN list WHERE condition)` -- at least one element satisfies condition
- `NONE(x IN list WHERE condition)` -- no element satisfies condition
- `SINGLE(x IN list WHERE condition)` -- exactly one element satisfies condition

## Query 5: Collecting Paths into a Network View

```cypher
MATCH (s:Shrine)-[r]->(t)
RETURN s.name AS from,
       type(r) AS relationship,
       t.name AS to
ORDER BY s.name, type(r)
```

**What this does:**
- `(s:Shrine)-[r]->(t)` -- all outgoing relationships from shrine nodes
- `type(r)` -- the relationship type as a string
- This gives a complete adjacency list of the shrine network

**Expected output:**

| from | relationship | to |
|------|-------------|-----|
| Atsuta Shrine | INFLUENCED_BY | Ise Grand Shrine |
| Fushimi Inari-taisha | BRANCH_OF | Ise Grand Shrine |
| Fushimi Inari-taisha | ENSHRINES | Inari Okami |
| Fushimi Inari-taisha | LOCATED_IN | Kyoto |
| ... | ... | ... |

## When to Use Path Queries

| Use Case | Cypher Feature |
|----------|---------------|
| "Is A connected to B?" | `shortestPath` |
| "How are they connected?" | `allShortestPaths` with node/rel inspection |
| "What can I reach in N steps?" | `[*1..N]` variable-length pattern |
| "Find all paths satisfying X" | Path + `WHERE ALL/ANY/NONE` |
| "Show the full network" | `MATCH (a)-[r]->(b) RETURN a, r, b` |

## Comparison with SPARQL Property Paths

| SPARQL | Cypher | Meaning |
|--------|--------|---------|
| `wdt:P279+` | `-[:SUBCLASS_OF*1..]->`| One or more hops |
| `wdt:P279*` | `-[:SUBCLASS_OF*0..]->`| Zero or more hops |
| `wdt:P17/wdt:P30` | `-[:COUNTRY]->()-[:CONTINENT]->` | Sequence |
| `(wdt:P131\|wdt:P276)` | `-[:LOCATED_IN\|LOCATION]->` | Alternative |
| N/A | `shortestPath(...)` | No SPARQL equivalent |

Cypher's path query capabilities are significantly more powerful than SPARQL property paths. SPARQL has no built-in shortest path, no path variable binding, and no path predicate filters.

## Exercises

1. **Find all nodes reachable from Ise Grand Shrine** within 2 hops via any relationship. Return the node names and the hop count.
2. **Find the shortest path between any Hachiman shrine and Amaterasu** (the deity node). What relationship chain connects them?
3. **Use `NONE` to find paths** from Tsurugaoka Hachimangu to any other node that do NOT pass through a node in Tokyo prefecture.
4. **Build a query that detects cycles** in the network. (Hint: look for paths where the start and end node are the same, with length > 0.)
