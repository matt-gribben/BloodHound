---
bh-rfc: 7
title: PostgreSQL as a Graph Database Backend
authors: |
    [BloodHound Engineering]
status: ACCEPTED
created: 2026-03-28
audiences: |
    BloodHound contributors and operators interested in the graph database architecture
---

# PostgreSQL as a Graph Database Backend

## 1. Overview

BloodHound supports PostgreSQL as a graph database backend in addition to Neo4j. The graph is not stored using a dedicated PostgreSQL graph plugin or extension. Instead, BloodHound uses **DAWGS** (Data Access With Graph Structures), an internal library that implements a property graph model on top of standard PostgreSQL tables, indexes, and PL/pgSQL functions. This document describes how the graph is modeled in PostgreSQL, how queries are executed, and how the two backends relate.

## 2. Motivation & Goals

-   **Reduce Operational Complexity** - Allow operators to run BloodHound with a single PostgreSQL instance rather than maintaining a separate Neo4j deployment.
-   **Database Agnosticism** - The application layer uses the same interfaces regardless of whether Neo4j or PostgreSQL is the active graph driver.
-   **No Vendor Extensions** - Keep the graph storage implementation portable using standard PostgreSQL features (JSONB, partitioned tables, PL/pgSQL) without requiring proprietary plugins such as Apache AGE or pgRouting.

## 3. Considerations

### 3.1 No PostgreSQL Graph Plugin

BloodHound does **not** use any PostgreSQL graph extension such as:

-   [Apache AGE](https://age.apache.org/) – an openCypher-compatible graph engine for PostgreSQL
-   [pgRouting](https://pgrouting.org/) – a routing/network analysis extension

The PostgreSQL extensions that _are_ used are:

| Extension  | Purpose                                                         |
|------------|-----------------------------------------------------------------|
| `pg_trgm`  | Enables GIN trigram indexes for fast substring/prefix searches on node properties |
| `intarray`  | Provides extended integer array operations (e.g., unions) used when managing `kind_ids` arrays on nodes |

### 3.2 Impact on Existing Systems

The Neo4j driver remains fully supported. The graph driver used at runtime is determined by configuration and can be switched live via the migration API described in [Section 4.4](#44-migrating-from-neo4j-to-postgresql).

### 3.3 Drawbacks & Alternatives

Representing a property graph in a relational database involves trade-offs compared to a purpose-built graph database:

-   **Graph traversal performance** – Multi-hop path queries in PostgreSQL rely on iterative PL/pgSQL harness functions rather than native graph algorithms. Performance for very large, deeply connected graphs may differ from Neo4j.
-   **Query translation** – Cypher queries are parsed and translated into SQL at runtime by DAWGS. Complex Cypher patterns may not translate 1:1 and could require additional optimization.

An alternative considered was adopting Apache AGE, which natively understands openCypher inside PostgreSQL. It was not selected because it introduces an additional compiled extension dependency and its maturity at the time was not sufficient for production use.

## 4. Details of the Proposal

### 4.1 DAWGS: The Graph Abstraction Layer

The graph layer in BloodHound is mediated by **DAWGS** (`github.com/specterops/dawgs`). DAWGS exposes two interfaces that the application interacts with:

```go
// graph.Database wraps the lifecycle of a graph connection.
type Database interface {
    ReadTransaction(ctx context.Context, txDelegate TransactionDelegate, options ...TransactionOption) error
    WriteTransaction(ctx context.Context, txDelegate TransactionDelegate, options ...TransactionOption) error
    AssertSchema(ctx context.Context, schema Schema) error
    // ...
}

// graph.Transaction represents a single read or write unit of work.
type Transaction interface {
    CreateNode(node *Node) error
    CreateRelationship(relationship *Relationship) error
    // ...
}
```

The same application code runs against either the PostgreSQL driver (`drivers/pg`) or the Neo4j driver (`drivers/neo4j`). The active driver is selected at startup via `bootstrap.ConnectGraph()` based on the application configuration, and can be switched at runtime using the graph driver switch API.

### 4.2 PostgreSQL Graph Schema

DAWGS creates and manages the following tables in PostgreSQL to store graph data. These tables are separate from the BloodHound application metadata tables (users, audit logs, permissions, etc.).

#### 4.2.1 Core Tables

```sql
-- graph: Registry of named graphs in the database.
-- Each graph gets its own partition of the node and edge tables.
CREATE TABLE graph (
    id   bigserial    PRIMARY KEY,
    name varchar(256) NOT NULL UNIQUE
);

-- kind: Registry of all node and edge type names mapped to compact integer IDs.
-- Stored as smallint to minimise storage in node kind_ids arrays.
CREATE TABLE kind (
    id   smallserial  PRIMARY KEY,
    name varchar(256) NOT NULL UNIQUE
);

-- node: Partitioned table storing all graph nodes.
-- Partitioned by graph_id (LIST partitioning) so that each graph's nodes
-- occupy an isolated partition that can be scanned independently.
CREATE TABLE node (
    id         bigserial   NOT NULL,
    graph_id   integer     NOT NULL,
    kind_ids   smallint[]  NOT NULL,   -- array of kind IDs (nodes can have multiple kinds)
    properties jsonb       NOT NULL,   -- all node properties as a JSONB document

    PRIMARY KEY (id, graph_id),
    FOREIGN KEY (graph_id) REFERENCES graph(id) ON DELETE CASCADE
) PARTITION BY LIST (graph_id);

-- edge: Partitioned table storing all directed relationships between nodes.
CREATE TABLE edge (
    id         bigserial  NOT NULL,
    graph_id   integer    NOT NULL,
    start_id   bigint     NOT NULL,    -- source node ID
    end_id     bigint     NOT NULL,    -- target node ID
    kind_id    smallint   NOT NULL,    -- single relationship type
    properties jsonb      NOT NULL,

    PRIMARY KEY (id, graph_id),
    FOREIGN KEY (graph_id) REFERENCES graph(id) ON DELETE CASCADE,
    UNIQUE (graph_id, start_id, end_id, kind_id)
) PARTITION BY LIST (graph_id);
```

When a new named graph is asserted (e.g., `bloodhoundgraph`), DAWGS creates a dedicated partition for both `node` and `edge`:

```sql
-- e.g., for graph_id = 1
CREATE TABLE node_1 PARTITION OF node FOR VALUES IN (1);
CREATE TABLE edge_1 PARTITION OF edge FOR VALUES IN (1);
```

#### 4.2.2 Composite Types

DAWGS defines PostgreSQL composite types to allow functions to return whole rows efficiently:

```sql
CREATE TYPE nodeComposite AS (
    id         bigint,
    kind_ids   smallint[],
    properties jsonb
);

CREATE TYPE edgeComposite AS (
    id         bigint,
    start_id   bigint,
    end_id     bigint,
    kind_id    smallint,
    properties jsonb
);

CREATE TYPE pathComposite AS (
    nodes nodeComposite[],
    edges edgeComposite[]
);
```

#### 4.2.3 Indexes

| Table  | Index              | Type   | Purpose                                      |
|--------|--------------------|--------|----------------------------------------------|
| `node` | `node_graph_id_index` | BTree | Filter nodes by graph                         |
| `node` | `node_kind_ids_index` | GIN   | Accelerate lookups by node kind               |
| `edge` | `edge_graph_id_index` | BTree | Filter edges by graph                         |
| `edge` | `edge_start_id_index` | BTree | Traverse edges from a given source node       |
| `edge` | `edge_end_id_index`   | BTree | Traverse edges to a given target node         |
| `edge` | `edge_kind_index`     | BTree | Filter edges by relationship type             |

BloodHound's `graphschema` package additionally asserts property-level indexes (BTree and GIN) on the node `properties` JSONB column for high-cardinality fields such as `objectid`, `name`, `domain_sid`, and `distinguished_name`.

### 4.3 Graph Traversal and Query Execution

#### 4.3.1 Cypher to SQL Translation

BloodHound expresses all graph queries in **Cypher** (the Neo4j query language). DAWGS parses Cypher using an embedded parser and translates the resulting AST into SQL at runtime. This allows the same query logic to run against both graph backends without change.

For example, a Cypher match on a node kind:

```cypher
MATCH (n:User) WHERE n.enabled = true RETURN n
```

is translated by DAWGS into a parameterised SQL query against the `node` partition.

#### 4.3.2 Graph Traversal Functions

Deep path queries (e.g., finding all attack paths between nodes) are implemented as PL/pgSQL stored functions. DAWGS generates the SQL for the primer and recursive expansion steps, then calls one of several traversal harnesses depending on the query shape:

| Harness function                        | Use case                                       |
|-----------------------------------------|------------------------------------------------|
| `unidirectional_sp_harness`             | Single-source shortest path                    |
| `unidirectional_asp_harness`            | All shortest paths from a single source        |
| `bidirectional_sp_harness`              | Bidirectional shortest path (source + target)  |
| `bidirectional_asp_harness`             | All shortest paths, bidirectional              |

Each harness manages a set of temporary tables (`forward_front`, `backward_front`, `next_front`, `paths`, `visited`) that track the BFS frontier across expansion steps. The temporary tables are scoped to the transaction and dropped on commit.

This approach avoids recursive CTEs for variable-depth traversal, trading query plan complexity for explicit frontier management in PL/pgSQL.

#### 4.3.3 Node and Edge DML

Core insert statements used by the DAWGS PostgreSQL driver:

```sql
-- Single node creation (returns a nodeComposite)
INSERT INTO node (graph_id, kind_ids, properties)
VALUES (@graph_id, @kind_ids, @properties)
RETURNING (id, kind_ids, properties)::nodeComposite;

-- Batch node creation (unnests parallel arrays)
INSERT INTO node (graph_id, kind_ids, properties)
SELECT $1, unnest($2::text[])::int2[], unnest($3::jsonb[]);

-- Edge creation with upsert (merges properties on conflict)
INSERT INTO edge AS e (graph_id, start_id, end_id, kind_id, properties)
SELECT $1, unnest($2::int8[]), unnest($3::int8[]), unnest($4::int2[]), unnest($5::jsonb[])
ON CONFLICT (graph_id, start_id, end_id, kind_id)
DO UPDATE SET properties = e.properties || excluded.properties;
```

### 4.4 Migrating from Neo4j to PostgreSQL

Live migration between the two backends is supported without downtime. The migration API endpoints are:

| Endpoint                  | Method | Description                                      |
|---------------------------|--------|--------------------------------------------------|
| `/pg-migration/status/`   | GET    | Returns `"idle"`, `"migrating"`, or `"canceling"` |
| `/pg-migration/start/`    | PUT    | Starts an async migration from Neo4j to PostgreSQL |
| `/pg-migration/cancel/`   | PUT    | Cancels a running migration                       |
| `/graph-db/switch/pg/`    | PUT    | Switches the active driver to PostgreSQL           |
| `/graph-db/switch/neo4j/` | PUT    | Switches the active driver back to Neo4j           |

The feature flag `pg_migration_dual_ingest` enables writing to both backends simultaneously during a transition period.

Full operational instructions are documented in [`cmd/api/src/api/tools/PG_MIGRATE.md`](../cmd/api/src/api/tools/PG_MIGRATE.md).

### 4.5 Application Metadata vs. Graph Storage

It is worth distinguishing the two PostgreSQL schemas used by BloodHound:

| Schema type        | Managed by              | Contents                                                                     |
|--------------------|-------------------------|------------------------------------------------------------------------------|
| Application schema | BloodHound migrations   | Users, sessions, audit logs, permissions, asset groups, schema metadata      |
| Graph schema       | DAWGS (`schema_up.sql`) | `graph`, `kind`, `node`, `edge` tables, composite types, traversal functions |

Both schemas reside in the same PostgreSQL database but are managed independently. The application schema uses GORM-based stepwise migrations (`cmd/api/src/database/migration/migrations/`). The graph schema is asserted by DAWGS at startup and is versioned within the DAWGS library.
