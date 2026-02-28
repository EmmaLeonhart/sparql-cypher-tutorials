# sparql-cypher-tutorials

> Scaffolded with [cleanvibe](https://github.com/Immanuelle/cleanvibe).

## About

A hands-on tutorial series for learning SPARQL and Cypher graph query languages. Each tutorial is a self-contained lesson with explanations, executable queries, and exercises. Uses real-world data from Wikidata and example Neo4j databases.

### Why This Exists

Good SPARQL and Cypher tutorials are surprisingly rare on GitHub. Most resources are either dense academic papers or vendor documentation. This repo provides practical, worked examples organized by difficulty -- from "what is a triple?" to "federated queries across multiple endpoints."

### Tutorial Structure (Planned)

#### SPARQL Track
1. **Basics** -- Triples, SELECT, WHERE, LIMIT
2. **Filtering** -- FILTER, OPTIONAL, UNION, MINUS
3. **Aggregation** -- COUNT, GROUP BY, HAVING, ORDER BY
4. **Property paths** -- Transitive closure, sequence paths, alternatives
5. **Subqueries & VALUES** -- Inline data, nested queries
6. **Federated queries** -- SERVICE keyword, querying multiple endpoints
7. **Wikidata-specific** -- Labels, descriptions, sitelinks, qualifiers, references
8. **Real-world project** -- Build a complete Wikidata analysis (e.g., all Shinto shrines by province)

#### Cypher Track
1. **Basics** -- Nodes, relationships, MATCH, RETURN
2. **Filtering** -- WHERE, pattern matching, list predicates
3. **Aggregation** -- COUNT, COLLECT, UNWIND
4. **Path queries** -- Shortest path, variable-length patterns
5. **Write operations** -- CREATE, MERGE, SET, DELETE
6. **Real-world project** -- Model and query a knowledge graph

### Each Tutorial Contains
- Concept explanation with diagrams
- Executable queries you can run immediately
- Expected output shown inline
- Exercises with solutions
- Links to further reading

## Tech Stack

- **Format:** Markdown files with embedded queries
- **SPARQL endpoint:** Wikidata Query Service (no setup needed)
- **Cypher:** Neo4j Sandbox or AuraDB Free (instructions provided)

## Getting Started

Start with [`sparql/01-basics.md`](sparql/01-basics.md) or [`cypher/01-basics.md`](cypher/01-basics.md).
