# Cypher Tutorial 4: Write Operations

## Overview

Cypher is a full read-write language. You can create, update, and delete nodes and relationships. The key operations are `CREATE` (always creates new), `MERGE` (create if not exists), `SET` (update properties), and `DELETE`/`DETACH DELETE` (remove data).

Understanding these operations is essential for building and maintaining graph databases, and for understanding how tools like the Wikidata QuickStatements system work under the hood (though Wikidata uses its own API, the concepts are parallel).

**Prerequisite:** Run the setup from Tutorials 1 and 3.

## Query 1: CREATE -- Adding New Data

```cypher
// Create a new shrine with relationships
CREATE (nikko:Shrine {
  name: 'Nikko Toshogu',
  founded: 1617,
  prefecture: 'Tochigi',
  type: 'Toshogu'
})
CREATE (ieyasu:Deity {
  name: 'Tokugawa Ieyasu',
  domain: 'Edo Shogunate'
})
CREATE (nikko)-[:ENSHRINES]->(ieyasu)

// Connect to existing prefecture or create new one
MERGE (tochigi:Prefecture {name: 'Tochigi'})
CREATE (nikko)-[:LOCATED_IN]->(tochigi)

RETURN nikko.name AS shrine, ieyasu.name AS deity
```

**What this does:**
- `CREATE` always makes new nodes, even if identical ones exist
- `MERGE` on the Prefecture only creates it if no `Prefecture {name: 'Tochigi'}` exists yet
- The mixed use of CREATE and MERGE is deliberate: we know the shrine is new, but the prefecture might already exist

**Expected output:**

| shrine | deity |
|--------|-------|
| Nikko Toshogu | Tokugawa Ieyasu |

## Query 2: MERGE -- Idempotent Upserts

```cypher
// MERGE: create only if not exists, otherwise match existing
MERGE (s:Shrine {name: 'Ise Grand Shrine'})
ON MATCH SET s.lastUpdated = datetime()
ON CREATE SET s.lastUpdated = datetime(), s.founded = -4

RETURN s.name, s.lastUpdated, s.founded
```

**What this does:**
- `MERGE` looks for a Shrine node named "Ise Grand Shrine"
- `ON MATCH SET` -- runs if the node already exists (updates lastUpdated)
- `ON CREATE SET` -- runs only if MERGE creates a new node
- This is idempotent: running it multiple times always produces the same result

**Key rule:** MERGE matches on ALL properties specified in the pattern. Be specific enough to uniquely identify the node, but not so specific that minor differences cause duplicates.

```cypher
// BAD: too many properties in MERGE -- slight variations create duplicates
MERGE (s:Shrine {name: 'Ise Grand Shrine', founded: -4, prefecture: 'Mie'})

// GOOD: merge on unique identifier, set other properties separately
MERGE (s:Shrine {name: 'Ise Grand Shrine'})
ON CREATE SET s.founded = -4, s.prefecture = 'Mie'
```

## Query 3: SET -- Updating Properties

```cypher
// Update a single property
MATCH (s:Shrine {name: 'Meiji Shrine'})
SET s.visitors_annually = 3000000
RETURN s.name, s.visitors_annually

// Add a new label to existing nodes
// MATCH (s:Shrine)
// WHERE s.type = 'Grand Shrine'
// SET s:GrandShrine
// RETURN s.name, labels(s)
```

**What this does:**
- `SET` adds or overwrites properties on matched nodes
- You can set multiple properties: `SET s.a = 1, s.b = 2`
- You can add labels: `SET s:NewLabel`
- You can copy all properties from a map: `SET s += {key: 'value', key2: 'value2'}`

**The `+=` operator** merges properties (adds new ones, updates existing, keeps unmentioned ones). The `=` operator on a map replaces ALL properties.

```cypher
// += preserves existing properties
MATCH (s:Shrine {name: 'Meiji Shrine'})
SET s += {visitors_annually: 3000000, hasMuseum: true}
RETURN s

// = replaces all properties (DANGEROUS -- loses everything else)
// SET s = {name: 'Meiji Shrine'}  -- would DELETE founded, prefecture, type!
```

## Query 4: DELETE and DETACH DELETE

```cypher
// First, let's see what we're about to delete
MATCH (s:Shrine {name: 'Nikko Toshogu'})-[r]-(connected)
RETURN s.name, type(r), connected.name
```

```cypher
// DETACH DELETE removes the node AND all its relationships
MATCH (s:Shrine {name: 'Nikko Toshogu'})
DETACH DELETE s
RETURN count(*) AS nodesDeleted
```

**What this does:**
- `DELETE` removes a node but FAILS if it has relationships (safety check)
- `DETACH DELETE` removes the node AND all connected relationships
- Always query first to understand what will be affected

**Warning:** `DETACH DELETE` can have cascading effects. If you delete a node that is the only connection between two subgraphs, you split the graph.

### Deleting relationships only:

```cypher
// Remove a specific relationship without deleting the nodes
MATCH (s:Shrine {name: 'Fushimi Inari-taisha'})-[r:BRANCH_OF]->(t:Shrine)
DELETE r
RETURN s.name, t.name, 'relationship removed' AS status
```

### Deleting properties:

```cypher
// Remove a property from a node
MATCH (s:Shrine {name: 'Meiji Shrine'})
REMOVE s.visitors_annually
RETURN s.name, s.visitors_annually
```

## Query 5: Bulk Operations -- Real-World Patterns

### Bulk update with FOREACH:

```cypher
// Mark all ancient shrines (founded before 800)
MATCH (s:Shrine)
WHERE s.founded < 800
SET s.era = 'Ancient'
RETURN s.name, s.era
```

### Conditional creation with CASE:

```cypher
MATCH (s:Shrine)
SET s.era = CASE
  WHEN s.founded < 0 THEN 'Mythological'
  WHEN s.founded < 794 THEN 'Ancient'
  WHEN s.founded < 1185 THEN 'Classical'
  WHEN s.founded < 1603 THEN 'Medieval'
  WHEN s.founded < 1868 THEN 'Early Modern'
  ELSE 'Modern'
END
RETURN s.name, s.founded, s.era
ORDER BY s.founded
```

**What this does:**
- `CASE WHEN ... THEN ... END` works like a switch statement
- Classifies each shrine into a historical era based on founding date
- Sets the `era` property on all shrine nodes at once

**Expected output:**

| name | founded | era |
|------|---------|-----|
| Izumo-taisha | -660 | Mythological |
| Ise Grand Shrine | -4 | Mythological |
| Atsuta Shrine | 113 | Ancient |
| Fushimi Inari-taisha | 711 | Ancient |
| Usa Jingu | 725 | Ancient |
| Kasuga Grand Shrine | 768 | Ancient |
| Tsurugaoka Hachimangu | 1063 | Classical |
| Meiji Shrine | 1920 | Modern |

## Write Operations Reference

| Operation | Purpose | Idempotent? |
|-----------|---------|-------------|
| `CREATE` | Always create new node/relationship | No -- duplicates on re-run |
| `MERGE` | Create if not exists, match if exists | Yes |
| `SET` | Add or update properties | Yes (same value) |
| `REMOVE` | Remove properties or labels | Yes |
| `DELETE` | Remove node (must have no relationships) | Yes |
| `DETACH DELETE` | Remove node and all its relationships | Yes (but destructive) |

## Parallels to Wikidata Editing

If you work with Wikidata (as covered in the SPARQL tutorials), the concepts map like this:

| Wikidata Operation | Cypher Equivalent |
|-------------------|-------------------|
| Add claim (P31=Q728) | `CREATE (s)-[:INSTANCE_OF]->(type)` or `MERGE` |
| Set qualifier | `SET r.qualifier = value` (on relationship) |
| Remove claim | `DELETE r` (delete relationship) |
| Deprecate statement | `SET r.rank = 'deprecated'` |
| QuickStatements batch | Multiple `MERGE` operations in a transaction |

The key difference: Wikidata uses a statement model with ranks and references (see SPARQL Tutorial 6), while Neo4j uses a simpler property model on relationships.

## Exercises

1. **Create three new shrine nodes** for shrines in Osaka prefecture. Create the prefecture node (using MERGE) and LOCATED_IN relationships.
2. **Use MERGE to ensure no duplicates** when adding a shrine that might already exist. Add an `ON MATCH` clause that updates the `lastChecked` timestamp.
3. **Write a CASE expression** that categorizes shrines by type: 'Grand' (type contains 'Grand'), 'Branch' (has BRANCH_OF relationship), or 'Independent' (everything else).
4. **Delete all shrine nodes in a specific prefecture** using DETACH DELETE. First, write a query to preview what will be deleted. Then write the delete query.
5. **Create a complete "Hachiman network":** Add Iwashimizu Hachimangu (founded 859, Kyoto) and connect it to Usa Jingu via BRANCH_OF, then connect Tsurugaoka Hachimangu to Iwashimizu via BRANCH_OF, forming a three-level hierarchy.
