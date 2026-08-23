# Chapter 3: Data Models and Query Languages

## Overview: Layering of Data Models

### The Abstraction Layers

**How data flows through an application** (highest to lowest):

1. **Application layer**: Real world modeled as objects, data structures, APIs specific to application
2. **Data model layer**: General-purpose data model (JSON, XML, relational tables, graph)
3. **Storage engine layer**: Bytes in memory, disk, network (querying, searching, manipulation)
4. **Hardware layer**: Electrical currents, light pulses, magnetic fields

**Key principle**: Each layer hides complexity of layers below via clean data model abstraction
- Allows different groups (DB vendors, app developers) to work together effectively

### Different Models for Different Purposes

**No one-size-fits-all model** — some data/queries easy in one model, awkward in another

**Models covered in this chapter**:
- Relational model
- Document model
- Graph-based models
- Event sourcing
- DataFrames

---

## Declarative Query Languages (Terminology)

### Definition

**Declarative language**: Specify pattern of data you want—what conditions results must meet, how to transform (sort, group, aggregate)—but NOT how to achieve it

**Examples**: SQL, Cypher, SPARQL, Datalog

### Contrast: Imperative Languages

**Imperative** (Python, Java): Write algorithm telling computer which operations to perform in which order

### Advantages of Declarative

- **Concise**: More concise and easier to write than explicit algorithm
- **Hidden implementation**: Hides implementation details → DB can introduce performance improvements without query changes
- **Parallelism**: Database can parallelize declarative query across CPU cores/machines without requiring developer effort
- **Optimization**: Query optimizer decides indexes, join algorithms, execution order

---

## Relational vs Document Models

### Historical Context

**Relational model** (Edgar Codd, 1970):
- Data organized into relations (tables)
- Each relation = unordered collection of tuples (rows)
- Theoretically proposed; widely doubted could be implemented efficiently
- By mid-1980s: Became tool of choice for structured data

**Competing approaches**: Network model, hierarchical model (1970s-80s); object databases (1980s-90s); XML databases (early 2000s)
- All generated hype, none lasted
- SQL expanded to incorporate other types (XML, JSON, graph)

**NoSQL movement** (2010s):
- Buzzword trying to overthrow relational dominance
- "NoSQL" = loose set of ideas (schema flexibility, scalability, open source)
- NewSQL = NoSQL scalability + relational guarantees
- Effect: Document model (usually JSON) became popular

---

## The Object-Relational Mismatch

### The Problem

**Challenge**: Application code in object-oriented languages; SQL uses relational tables

**Impedance mismatch**: Disconnect between object model and database model
- Awkward translation layer required
- Term borrowed from electronics (resistance to AC current; best power transfer when impedances match)

### Object-Relational Mapping (ORM) Frameworks

**Examples**: ActiveRecord, Hibernate

**Stated goal**: Reduce boilerplate code for translation layer

### ORM Problems

- **Can't hide differences**: Developers still think about both relational and object representations
- **OLTP only**: Used only for OLTP app dev; data engineers for analytics need underlying relational schema
- **Limited DB support**: Many ORMs only work with relational OLTP databases (not search engines, graph, NoSQL)
- **Auto-generated schemas**: Automatically generated relational schemas awkward for direct access, inefficient; customization complex
- **N+1 query problem**: ORM accidentally generates inefficient queries
  - Example: Fetch N comments with author IDs, then make separate query per comment to look up author name
  - Result: N+1 queries instead of one join
  - Solution: Tell ORM to fetch author info simultaneously

### ORM Advantages

- **Reduces boilerplate**: For well-suited relational data, reduces translation boilerplate
- **Query caching**: Some ORMs help cache query results, reducing DB load
- **Schema management**: Help with schema migrations and admin tasks
- **Simple cases**: ORMs good for simple/repetitive cases; complex queries handled outside ORM

---

## Document Model for One-to-Many Relationships

### Use Case: LinkedIn Profile (Résumé)

**Relational approach** (Figure 3-1):
- users table: first_name, last_name, user_id
- positions table: job_title, organization (foreign key to users)
- education table: school_name (foreign key to users)
- contact_info table: website, social media (foreign key to users)
- Problem: One-to-many relationships split across multiple tables

**JSON document approach** (Example 3-1):
```json
{
  "user_id": 251,
  "first_name": "Barack",
  "positions": [
    {"job_title": "President", "organization": "United States of America"},
    {"job_title": "US Senator (D-IL)", "organization": "United States Senate"}
  ],
  "education": [
    {"school_name": "Harvard University", "start": 1988, "end": 1991}
  ]
}
```

### JSON vs Relational: Trade-offs

**JSON advantages**:
- Reduces impedance mismatch (closer to object structure)
- **Better locality**: All relevant info in one place
  - Relational: Multiple queries or messy multiway joins needed
  - JSON: Single document, faster and simpler query
- Makes tree structure explicit (one-to-many relationships form tree)
- Lack of schema seen as advantage (more flexible)

**Relational advantages**:
- Better for many-to-one, many-to-many relationships
- Simpler queries for complex relationships

### When to Use Document Model

**Document model best for**:
- Document-like structure (tree of one-to-many relationships)
- Entire tree usually loaded at once
- Avoids cumbersome shredding (splitting document into multiple tables)

**Document model limitations**:
- Cannot refer directly to nested item (must say "second item in positions list")
- Relational approach better if need to reference nested items by ID
- Reorderable lists (to-do, issue tracker): Document model supports well (store IDs in array)
  - Relational databases lack standard way (tricks: integer column, linked list, fractional indexing)

**Note**: One-to-few vs One-to-many
- Résumé has few positions → one-to-few
- Celebrity social media has thousands of comments → one-to-many
- If truly large number, relational approach preferable (embedding unwieldy)

---

## Normalization, Denormalization, and Joins

### Why Normalize: The region_id Example

**Denormalized** (store "Washington, DC"):
- Pros: Self-contained
- Cons: 
  - Inconsistent spelling across profiles
  - Ambiguity (which Washington?)
  - Hard to update (must find all occurrences)
  - No localization support
  - Poor search functionality

**Normalized** (store ID reference):
- Pros:
  - Consistent style and spelling
  - No ambiguity
  - Easy to update (one place)
  - Localization support
  - Better search (can encode locations in region table)
  - ID never needs change (no meaning to humans)
- Cons: Must lookup ID to display human-readable info (join operation)

### Trade-offs of Normalization

**General principle**:
- **Normalized**: Faster to write (one copy), slower to query (requires joins)
- **Denormalized**: Faster to read (fewer joins), slower to write (multiple copies), more disk space

**View as derived data**: Denormalization = form of derived data needing update process

**Consistency considerations**:
- Atomic transactions help maintain consistency
- Not all databases offer atomicity across multiple documents
- Stream processing can ensure consistency (Chapter 12)

### When to Use Each

**Normalization better for**: OLTP systems (both reads and updates need to be fast)

**Denormalization better for**: Analytical systems (bulk updates, read-only queries dominant)

**At scale**: Small/moderate systems → normalized best; very large systems → join costs problematic

### Case Study: Social Network Timelines (Denormalization)

From Chapter 2:
- Materialized timeline = denormalized view of posts
- Issue: Plain denormalization stores post text, causing problem
- Solution: Store only post IDs + metadata, then hydrate (lookup) on read
  - Reason: Post data (likes, replies, username, profile photo) fast-changing
  - Denormalizing would waste storage and become stale
- Hydrating IDs: Easy to scale, parallelizes well, doesn't depend on follow count
- Lesson: Most scalable approach = denormalize some things, keep others normalized
- Normalization/denormalization not inherently good/bad → performance trade-offs

---

## Many-to-One and Many-to-Many Relationships

### Relationship Types in Résumé Example

**One-to-many** (one résumé, many positions):
- Positions and education tables example
- Also called one-to-few

**Many-to-one** (many people, one region):
- region_id field — many people live in same region
- Assume each person lives in only one region at time

**Many-to-many** (introduced if organizations/schools are entities):
- One person worked for several organizations
- One organization has several employees

### Representing Many-to-Many in Relational Model

**Associative table / join table** (Figure 3-3):
- positions table: user_id (FK to users), org_id (FK to organizations)
- Each row associates one user with one organization
- Standard relational approach

### Representing Many-to-Many in Document Model

**Challenge**: Many-to-many don't fit easily in one self-contained JSON document
- Best represented as references to other documents

**Example approach** (Example 3-2):
```json
{
  "positions": [
    {"start": 2009, "end": 2017, "job_title": "President", "org_id": 513}
  ]
}
```

**Querying both directions**:
- Résumé includes org IDs (find orgs person worked for)
- Organization document includes résumé IDs (find people worked there)
- This denormalized (relationship stored twice) — can become inconsistent

**Normalized approach**: Store relationship once, use indexes
- Relational: Indexes on user_id and org_id columns of positions table
- Document: Index org_id field inside positions array

---

## Stars and Snowflakes: Schemas for Analytics

### Context

**Data warehouses**: Usually relational with standardized structures for business analysts

**Common conventions**: Star schema, snowflake schema, dimensional modeling, one big table (OBT)

### Star Schema (Figure 3-5)

**Center**: Fact table (fact_sales) — each row = event at particular time
- Example: Customer purchase of product (other examples: page view, click)

**Columns in fact table**:
- **Attributes**: price sold, cost from supplier (allows profit calculation)
- **Foreign keys**: References to dimension tables

**Dimension tables**: Who, what, where, when, how, why of event
- dim_product: SKU, description, brand, category, fat content, package size
- dim_date: Can include public holidays (differentiate holiday vs non-holiday sales)
- dim_store: Services offered, in-store bakery, square footage, opened date, remodeled date, distance to highway

**Fact table structure**: Frequently over 100 columns, sometimes several hundred
- Dimension tables also wide (all metadata relevant for analysis)

**Why "star"**: Visual relationship — fact table in middle surrounded by dimension tables (like star rays)

### Snowflake Schema

**Variation**: Dimensions further broken into subdimensions
- Separate tables for brands and categories
- dim_product table references brand and category as foreign keys (more normalized)

**Comparison**:
- Snowflake = more normalized than star schema
- Star schema = simpler for analysts to work with (preferred)

### Relationships and Denormalization in Warehouse

**Primary relationships**: Many-to-one (many sales for one product, one store)
- Other relationship types often denormalized to simplify queries

**Multi-item transactions**: Not represented explicitly
- Instead: Separate fact table row for each product purchased
- Same customer ID, store ID, timestamp indicates multi-item transaction

**One Big Table (OBT) approach**:
- Folding dimension table info into denormalized fact table columns
- Precomputing join between fact and dimension tables
- Trade-off: More storage, sometimes faster queries

### Why Denormalization Works for Analytics

**Key difference from OLTP**: Data represents log of historical data
- Not going to change (except maybe error corrections)
- Data consistency and write overhead issues not pressing
- Denormalization unproblematic

---

## When to Use Which Model

### Document Model Strengths

**Best when**:
- Document-like structure (tree of one-to-many)
- Entire tree loaded at once
- Data has similar structure

**Advantages**:
- Schema flexibility
- Better performance due to locality
- Closer to object model used by application

### Document Model Weaknesses

- Cannot refer directly to nested item
- Limitations on referencing capability
- Reorderable lists harder (relational has standard approaches)

### Relational Model Strengths

- Better support for joins
- Better for many-to-one, many-to-many relationships
- When all records expected to have same structure, schemas useful
- Document structure forces explicit relationships

---

## Schema Flexibility in Document Model

### Schemaless vs Schema-on-Read vs Schema-on-Write

**Schemaless claim**: Misleading — code reading data assumes structure
- More accurate: **Schema-on-read** (implicit, interpreted on read)

**Comparison**:
- **Schema-on-write** (relational): Explicit schema, database enforces conformity on write
- **Schema-on-read** (document): Implicit schema, no enforcement

**Analogy**: Static (compile-time) vs dynamic (runtime) type checking

### Schema Evolution: Adding first_name Column

**Document database approach** (schema-on-read):
```javascript
if (user && user.name && !user.first_name) {
  user.first_name = user.name.split(" ")[0];
}
```
- Start writing new documents with new fields
- Code handles old/new formats
- Downside: Every reader needs to handle old formats

**Relational database approach** (schema-on-write):
```sql
ALTER TABLE users ADD COLUMN first_name text DEFAULT NULL;
UPDATE users SET first_name = split_part(name, ' ', 1);
```
- Add column with default (fast, even on large tables)
- Run UPDATE statement (slow — every row rewritten)
- Other schema changes (datatype changes) require table copy

**Mitigation for relational**:
- Tools allow background schema changes without downtime
- Large database migrations remain operationally challenging
- Workaround: Add column with NULL default (fast), fill at read time (like document DB)

### When Schema-on-Read Advantageous

**Best when**:
- Collection items don't all have same structure (heterogeneous)
- Many types of objects (impractical to put each type in separate table)
- Data structure determined by external systems (no control, may change)

**Best when Schema-on-Write Advantageous**:
- All records expected to have same structure
- Schemas useful for documenting and enforcing structure

---

## Data Locality for Reads and Writes

### Storage Format

**Document storage**: Single continuous string (JSON, XML, binary variant like BSON)

**Locality advantage**:
- If application often needs entire document (render on webpage)
- Performance advantage over split data across multiple tables
- Multiple index lookups required if split → more disk seeks, more time

**Limitations**:
- Advantage only if need large parts of document simultaneously
- Database typically loads entire document (wasteful if need only small part)
- Updates usually rewrite entire document
- Recommendation: Keep documents small, avoid frequent small updates

### Locality Not Limited to Document Model

**Other approaches with locality**:
- Google Spanner: Relational model with interleaved (nested) rows
- Oracle: Multi-table index cluster tables
- Wide-column model (Bigtable, HBase, Accumulo): Column families for locality management

---

## Query Languages for Documents

### Variations in Document Databases

**Range of query support**:
- Some allow only key-value access by primary key
- Some offer secondary indexes
- Some provide rich query languages

### XML Query Languages

**XQuery and XPath**: Designed for complex queries including cross-document joins, formatted as XML

### JSON Query Tools

**JSON Pointer and JSONPath**: Equivalent to XPath for JSON

### MongoDB Aggregation Pipeline

**Example**: Cypher-like joins with `$lookup` operator

```javascript
db.users.aggregate([
  { $match: { _id: 251 } },
  { $lookup: {
    from: "regions",
    localField: "region_id",
    foreignField: "_id",
    as: "region"
  } }
])
```

### SQL vs MongoDB Example

**PostgreSQL query** (shark observation count by month):
```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month,
  sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks'
GROUP BY observation_month;
```

**MongoDB aggregation pipeline** (same query):
```javascript
db.observations.aggregate([
  { $match: { family: "Sharks" } },
  { $group: {
    _id: {
      year: { $year: "$observationTimestamp" },
      month: { $month: "$observationTimestamp" }
    },
    totalAnimals: { $sum: "$numAnimals" }
  } }
]);
```

**Difference**: MongoDB JSON-based syntax vs SQL's English-like syntax (matter of taste)

---

## Convergence of Document and Relational Databases

### Historical Divergence → Convergence

**Started**: Very different approaches

**Recent trend**: Growing more similar

**Relational changes**: Added JSON types, query operators, indexing inside documents

**Document changes**: MongoDB, Couchbase, RethinkDB added joins, secondary indexes, declarative query languages

### Hybrid Advantages

**Good news**: Can combine both in same database
- Many document DBs need relational-style references
- Many relational DBs benefit from schema flexibility

**Hybrid strength**: Relational-document combination powerful

### Historical Note

Codd's original relational model (1970) allowed "nonsimple domains" — nested relations (tables) as values
- Like JSON/XML support added 30+ years later

---

## Graph-Like Data Models

### When Graphs Are Appropriate

**Relational model**: Good for one-to-many (tree-structured), few other relationships

**Graph model**: When many-to-many relationships very common

### Graph Basics

**Components**:
- **Vertices** (nodes/entities): Represent objects
- **Edges** (relationships/arcs): Connect vertices

**Typical examples**:
- Social graphs: People connected by friendships
- Web graph: Pages connected by links
- Road/rail networks: Junctions connected by roads/railways

**Well-known algorithms**:
- Shortest path (navigation)
- PageRank (web ranking)

### Graph Representations

**Adjacency list**: Each vertex stores IDs of neighbor vertices one edge away
- Good for graph traversals

**Adjacency matrix**: 2D array, row/column = vertex, value = 1 (edge) or 0 (no edge)
- Good for machine learning

### Homogeneous vs Heterogeneous Graphs

**Homogeneous**: All vertices same type (people, pages, junctions)

**Heterogeneous**: Different vertex types in single database
- **Facebook**: Vertices (people, locations, events, check-ins, comments); edges (friendship, location, authored)
- **Search engines**: Knowledge graphs (organizations, people, places)

### Graph Advantages for Complex Data

**Example from Figure 3-6**: Two people (Lucy, Alain), locations (France, US, London)
- Different regional structures (France: départements/régions; US: counties/states)
- Country within country
- Varying granularity
- Easy to extend with more facts (allergies, foods)
- Graphs evolvable: Add features by extending graph

---

## Property Graphs

### Model Definition

**Vertex consists of**:
- Unique identifier
- Label (describes object type)
- Outgoing edges
- Incoming edges
- Properties (key-value pairs)

**Edge consists of**:
- Unique identifier
- Tail vertex (start)
- Head vertex (end)
- Label (describes relationship type)
- Properties (key-value pairs)

### Relational Implementation

**Can represent as two tables** (Example 3-3):

```sql
CREATE TABLE vertices (
  vertex_id integer PRIMARY KEY,
  label text,
  properties jsonb
);

CREATE TABLE edges (
  edge_id integer PRIMARY KEY,
  tail_vertex integer REFERENCES vertices,
  head_vertex integer REFERENCES vertices,
  label text,
  properties jsonb
);

CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

### Key Features

- Any vertex can edge to any other (no schema restrictions)
- Efficiently find incoming/outgoing edges (traversal forward/backward)
- Different labels for different vertex/relationship types
- Flexibility: Store several kinds of info in single graph, maintain clean model

### Limitations

**Edge constraint**: Edge can associate only two vertices
- Relational join table can represent three-way+ relationships via multiple FKs
- Workaround: Create additional vertex corresponding to join table row, edges to/from it
- Alternative: Hypergraph

### Database Examples

**Implementations**: Neo4j, Memgraph, KùzuDB, Amazon Neptune (both property graph and triple store)

---

## The Cypher Query Language

### Overview

**Purpose**: Query language for property graphs

**Origin**: Created for Neo4j, now open standard (openCypher)

**Support**: Neo4j, Memgraph, KùzuDB, Amazon Neptune, Apache AGE (on PostgreSQL), others

**Name origin**: Character in The Matrix film (not related to cryptography)

### Cypher Example: Data Insertion

**Example 3-4** (inserting portion of Figure 3-6):

```cypher
CREATE
(namerica :Location {name:'North America', type:'continent'}),
(usa :Location {name:'United States', type:'country'}),
(idaho :Location {name:'Idaho', type:'state'}),
(lucy :Person {name:'Lucy'}),
(idaho) -[:WITHIN]-> (usa) -[:WITHIN]-> (namerica),
(lucy) -[:BORN_IN]-> (idaho)
```

**Syntax**:
- Vertex represented with name in parentheses: `(idaho)`
- Label after colon: `:Location`
- Properties in braces: `{name:'Idaho', type:'state'}`
- Arrow notation for edges: `(idaho) -[:WITHIN]-> (usa)`
- Arrow direction: Tail → head

### Cypher Example: Complex Query

**Query**: Find people who emigrated from US to Europe

**Example 3-5**:

```cypher
MATCH
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (:Location {name:'United States'}),
(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (:Location {name:'Europe'})
RETURN person.name
```

**Query logic**:
1. Find person with BORN_IN edge to vertex
2. From that vertex, follow chain of outgoing WITHIN edges until reach Location vertex named "United States"
3. Same person also has LIVES_IN edge
4. Following that, chain of WITHIN edges reaches Location vertex named "Europe"
5. Return person's name

**Syntax breakdown**:
- `MATCH`: Find patterns in graph
- `(person) -[:BORN_IN]-> ()`: Tail bound to person, head unnamed
- `:WITHIN*0..`: Follow WITHIN edge zero or more times (like regex `*`)
- `(:Location {name:'United States'})`: Match Location with specific name

**Execution flexibility**: Can start from locations and work backward (if name indexed)

**Comparison**: 4-line Cypher query vs 31 lines in SQL (recursive CTEs) shows power of right data model/language

---

## Graph Queries in SQL

### The Challenge

**Problem**: Graph queries traverse variable number of edges; SQL queries usually know joins in advance

**Example**: LIVES_IN edge may point to street, city, district, region, or state
- City may be WITHIN region, region within state, state within country
- Multiple levels away in hierarchy

**SQL solution**: Recursive common table expressions (WITH RECURSIVE)

### SQL Complexity vs Cypher

**Cypher conciseness**: `:WITHIN*0..` expresses variable-length paths

**SQL equivalent** (Example 3-6): 31 lines with recursive CTEs
- Define in_usa (all locations within US)
- Define in_europe (all locations within Europe)
- Define born_in_usa (people born in US locations)
- Define lives_in_europe (people living in Europe locations)
- Join to find people in both sets

**Length difference**: 4 lines Cypher vs 31 lines SQL (shows difference of right query language)

### Standards Development

**GQL ISO standard**: Published 2024, based on Cypher
- Should improve uniformity among graph databases
- Not yet widely adopted

### Alternative Graph Query Languages

**Examples**: TigerGraph's GSQL, Property Graph Query Language (PGQL), Gremlin

---

## Triple Stores and SPARQL

### Triple Store Model

**Equivalence**: Mostly equivalent to property graph (different terminology)

### Triple Basics

**Three-part statement**: (subject, predicate, object)

**Example**: (Jim, likes, bananas)
- Jim = subject
- likes = predicate (verb)
- bananas = object

**Note**: Databases often store additional metadata
- AWS Neptune: Quads (add graph ID)
- Datomic: 5-tuples (add transaction ID and deletion Boolean)
- Still called "triple stores" for simplicity

### Subject and Object in Triples

**Subject**: Equivalent to vertex

**Object** (two possibilities):

1. **Primitive datatype value** (string, number)
   - Predicate + object = property on subject vertex
   - Example: (lucy, birthYear, 1989) = vertex lucy with {"birthYear": 1989}

2. **Another vertex**
   - Predicate = edge
   - Subject = tail vertex
   - Object = head vertex
   - Example: (lucy, marriedTo, alain) = edge between two vertices

### Turtle Format (RDF Encoding)

**Example 3-7** (LinkedIn profile data):

```turtle
@prefix : <urn:example:>.
_:lucy a :Person.
_:lucy :name "Lucy".
_:lucy :bornIn _:idaho.
_:idaho a :Location.
_:idaho :name "Idaho".
_:idaho :type "state".
_:idaho :within _:usa.
```

**Syntax**:
- `_:someName`: Vertex (name only meaningful within file)
- `a`: Shorthand for "is a" (type)
- `:predicate`: Property or edge
- String literal: Object value (in quotes)
- Semicolon: Multiple predicates about same subject (concise format)

### Concise Turtle Format

**Example 3-8** (more concise):

```turtle
_:lucy a :Person; :name "Lucy"; :bornIn _:idaho.
_:idaho a :Location; :name "Idaho"; :type "state"; :within _:usa.
```

### The Semantic Web Context

**Motivation**: Early 2000s effort for internet-wide data exchange (machine-readable format)

**Original vision**: Didn't fully succeed as envisioned

**Legacy**: Lives on in
- Linked data standards (JSON-LD)
- Biomedical science ontologies
- Facebook's Open Graph protocol
- Knowledge graphs (Wikidata)
- Structured data vocabularies (Schema.org)

**Current use**: Triples effective internal data model even outside Semantic Web context

### RDF Data Model

**Resource Description Framework**: Designed for Semantic Web

**Encoding options**:
- Turtle (N3 subset)
- XML (verbose)
- Other formats

**Tools**: Apache Jena automatically converts between encodings

### RDF Quirks

**URIs as identifiers**: Subject, predicate, object often URIs (not just plain names)

**Example**: Predicate might be `<http://my-company.com/namespace#within>` instead of just WITHIN

**Reasoning**: Enable combining data from different sources without conflicts
- Your `#within` different from someone else's `#within` (different URIs)

**Namespace shorthand**: Specify prefix once, reuse throughout file
- Example: `@prefix : <urn:example:>.` then use `:within`

---

## SPARQL Query Language

### Overview

**Name**: SPARQL Protocol and RDF Query Language (recursive acronym)
- Pronounced "sparkle"

**Timeline**: Predates Cypher; Cypher pattern matching borrowed from SPARQL

**Similarity to Cypher**: Similar expressiveness and conciseness

### SPARQL vs Cypher Comparison

**Query**: People who moved from US to Europe

**SPARQL equivalent** (Example 3-10):

```sparql
PREFIX : <urn:example:>
SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

**Comparison**:
- Variables start with `?` in SPARQL
- `/` denotes path traversal (equivalent to Cypher's `-[edge]->`)
- `*` means zero or more (regex-like)
- Equivalent expressions:
  - Cypher: `(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location)`
  - SPARQL: `?person :bornIn / :within* ?location`

**Property matching** (both similar):
- Cypher: `(usa {name:'United States'})`
- SPARQL: `?usa :name "United States".`

### SPARQL Database Support

Supported by: Amazon Neptune, AllegroGraph, Blazegraph, OpenLink Virtuoso, Apache Jena, others

---

## Datalog: Recursive Relational Queries

### Overview

**Age**: Much older than SPARQL/Cypher (academic research, 1980s)

**Adoption**: Less well-known among engineers, limited mainstream database support

**Value**: Very expressive, especially powerful for complex queries

**Implementations**: Datomic, LogicBlox, CozoDB, LinkedIn's LIquid

**Foundation**: Based on relational data model (not graph), but recursive queries especially strong

### Datalog Basics

**Data representation**: Facts (correspond to rows in relational table)

**Syntax**: `table(val1, val2, ...)`

**Example** (Figure 3-6 data):

```datalog
location(1, "North America", "continent").
location(2, "United States", "country").
location(3, "Idaho", "state").
within(2, 1).  /* US in North America */
within(3, 2).  /* Idaho in US */
person(100, "Lucy").
born_in(100, 3).  /* Lucy born in Idaho */
```

### Datalog Query Example

**Query**: People who emigrated from US to Europe

**Example 3-12**:

```datalog
within_recursive(LocID, PlaceName) :- location(LocID, PlaceName, _).
within_recursive(LocID, PlaceName) :- within(LocID, ViaID), within_recursive(ViaID, PlaceName).

migrated(PName, BornIn, LivingIn) :- person(PersonID, PName), born_in(PersonID, BornID),
  within_recursive(BornID, BornIn), lives_in(PersonID, LivingID), within_recursive(LivingID, LivingIn).

us_to_europe(Person) :- migrated(Person, "United States", "Europe").
```

### How Datalog Works

**Rules**: Define virtual tables (like SQL views)

**Structure**: `head :- body`
- Before `:-`: Virtual table name and columns
- After `:-`: Patterns to match in underlying facts

**Example**: `migrated(PName, BornIn, LivingIn)` virtual table with three columns

**Execution** (iterative application of rules):

1. `location(1, "North America", "continent")` exists → Rule 1 generates `within_recursive(1, "North America")`
2. `within(2, 1)` exists + previous step → Rule 2 generates `within_recursive(2, "North America")`
3. `within(3, 2)` exists + previous step → Rule 2 generates `within_recursive(3, "North America")`
4. By repeated application, `within_recursive` table contains all locations in North America

**Recursion**: Rule 2 invokes itself (like recursive functions)

### Why Datalog Is Powerful

**Advantage over others**: Expressiveness for complex queries

**Similarity to programming**: Break complex queries into rules calling other rules (like functions)

**Graph traversals**: Recursive queries handle well (like Rule 2 example)

**Different thinking**: Requires different mental model than SQL/Cypher

---

## GraphQL

### Overview

**Purpose**: OLTP query language for client software (mobile apps, web frontends)

**Intentional limitation**: Much more restrictive than other query languages

**Goal**: Allow clients to request JSON document with specific structure for UI rendering

### GraphQL Design Principles

**Client flexibility**: Rapidly change queries without server-side API changes

**Cost**: Organizations need tooling to convert to internal services (REST, gRPC)

**Challenges**: Authorization, rate limiting, performance

### Safety Through Restriction

**Reason for limitations**: GraphQL queries from untrusted sources

**Restrictions**:
- No expensive execution (prevent DoS via expensive queries)
- No recursive queries (unlike Cypher, SPARQL, SQL, Datalog)
- No arbitrary search conditions (unless explicitly offered)

**Trade-off**: Simpler, safer but less expressive

### GraphQL Use Case: Chat Application

**Example 3-13** (Discord/Slack-like group chat):

```graphql
query ChatApp {
  channels {
    name
    recentMessages(latest: 50) {
      timestamp
      content
      sender {
        fullName
        imageUrl
      }
      replyTo {
        content
        sender {
          fullName
        }
      }
    }
  }
}
```

**Query logic**:
- Get all channels user has access to
- For each: channel name + 50 most recent messages
- For each message: timestamp, content, sender (name + profile pic)
- If reply: included message content and sender name (provides context)

### GraphQL Response Structure

**Example 3-14** (possible response):

```json
{
  "data": {
    "channels": [
      {
        "name": "#general",
        "recentMessages": [
          {
            "timestamp": 1693143014,
            "content": "Hey! How are y'all doing?",
            "sender": {"fullName": "Aaliyah", "imageUrl": "https://..."},
            "replyTo": null
          }
        ]
      }
    ]
  }
}
```

### GraphQL Design Choices

**Response mirrors query structure**: Exact fields requested, no more/no less

**Advantage**: Server doesn't need to know which attributes client requires
- UI changes → client adds requested attributes, no server changes

**Denormalization choice**: Sender info embedded directly in message
- Duplication if same user sends multiple messages
- Simpler for UI rendering
- Trade-off: Larger response size vs simpler data handling

**Alternative**: Return message IDs, require additional requests
- Datalog approach more normalized
- GraphQL accepts duplication for simplicity

### GraphQL Implementation

**Key point**: GraphQL can run on any database type
- Relational, document, graph
- Server performs necessary joins; client requests only join paths declared in schema

---

## Event Sourcing and CQRS

### The Problem Being Solved

**Challenge**: Single data representation can't satisfy all query and presentation needs

**Example reference**: "Systems of Record and Derived Data" (Chapter 1), ETL (Chapter 1)

### Event Sourcing Concept

### What If Optimized Only for Writing?

**Simplest, fastest, most expressive write format**: Event log

**How it works**:
- Every data write encoded as self-contained string (often JSON)
- Includes timestamp
- Appended to sequence of events
- Events immutable (never change/delete, only append)
- Event can contain arbitrary properties

### Conference Management Example (Figure 3-8)

**Complex domain**: Attendee registration, corporate bulk orders, invoicing, seat assignment, reservation cancellations, capacity changes

**Calculating available seats**: Challenging query (many state changes)

**Solution**: Use event log as source of truth
- Every state change (registration open, attendees register/cancel, organizer changes capacity) = event
- Append to log
- Multiple materialized views derived from events:
  - View 1: Booking status info
  - View 2: Organizer dashboard charts
  - View 3: Badge files for printer

### Event Sourcing and CQRS Definitions

**Event sourcing**: Using events as source of truth, expressing every state change as event

**CQRS (Command Query Responsibility Segregation)**: Maintaining separate read-optimized representations derived from write-optimized representation

**Origin**: Domain-driven design (DDD) community

**Similar patterns**: State machine replication (Chapter 10)

### Event Flow

**User request**:
1. Called a "command"
2. Must be validated first
3. Once validated and executed (e.g., enough seats for reservation)
4. Becomes fact → corresponding event added to log
5. Event log contains only valid events
6. Materialized view consumers can't reject events

### Event Naming Convention

**Best practice**: Name events in past tense
- Example: "the seats were booked"
- Reason: Event = record of fact that happened
- Even if user later cancels, fact remains true (separate event added later)

### Comparison to Star Schema

**Similarity**: Both are collections of past events

**Differences**:
- Fact table: All rows have same columns
- Event sourcing: Many event types, different properties each
- Fact table: Unordered collection
- Event sourcing: Order of events important (booking then cancellation doesn't make sense reversed)

### Event Sourcing Advantages

1. **Intent communication**
   - Better communicates why something happened
   - Example: "booking was canceled" clearer than row modifications
   - Reason for updates explicit

2. **Reproducible materialized views**
   - Derive views from event log reproducibly
   - Delete materialized view, recompute from same events, same code
   - Easier debugging (rerun view code repeatedly, inspect)
   - Fix bugs: delete view, recompute with new code

3. **Multiple optimized views**
   - Views optimized for particular queries
   - Any data model
   - Can denormalize for fast reads
   - Can keep in-memory only (recompute on restart)

4. **Easy new views**
   - New way to present info → new materialized view from existing log
   - Support new features: Add new event types or event properties
   - Old events unchanged
   - Chain new behaviors off existing events (e.g., offer seat to waitlist)

5. **Handle errors gracefully**
   - Erroneous write → subsequent deletion event reverses it
   - Downstream views automatically incorporate deletion
   - Direct data updates hard to reverse
   - Event sourcing reduces irreversible actions → easier to change

6. **Audit log**
   - Event log serves as audit log
   - Valuable in regulated industries (auditability required)

7. **High write throughput**
   - Event logs handle higher write throughput than databases
   - Sequential access patterns
   - Temporary event bursts absorbed
   - Downstream views catch up at own pace (not overwhelmed)

### Event Sourcing Downsides

1. **External information handling**
   - Event contains price in one currency
   - Need to convert to another currency for view
   - Exchange rates fluctuate
   - Problem: Fetching external rate when processing event → different result if recomputed later
   - Solution: Include exchange rate in event OR query historical rate at timestamp (deterministic)

2. **GDPR/personal data deletion**
   - Immutable events problematic for GDPR right to erasure
   - If event log per-user: delete whole log (works)
   - Multi-user log: harder to delete specific user data
   - Solutions:
     - Store personal data outside event log
     - Encrypt with key that can later be deleted (crypto-shredding)
   - Trade-off: Harder to recompute derived state

3. **Side effects from reprocessing**
   - Events have externally visible side effects (send confirmation email)
   - Problem: Resending email when rebuilding materialized view
   - Need careful handling

### Event Sourcing Implementations

**Purpose-built systems**: EventStoreDB, MartenDB (PostgreSQL-based), Axon Framework

**Alternative**: Message brokers (Apache Kafka) store event log
- Stream processors keep materialized views up-to-date (Chapter 12)

**Key requirement**: Event storage must guarantee all views process events in exactly same order as log
- Not always easy in distributed systems (Chapter 10)

---

## DataFrames, Matrices, and Arrays

### Context

**Data models covered so far**: Generally used for both transaction processing (OLTP) and analytics (OLAP)

**Other models**: Primarily analytical/scientific, rarely in OLTP systems

### DataFrame Basics

**Support**: R language, Pandas library (Python), Apache Spark, ArcticDB, Dask, others

**Use cases**:
- Data scientists preparing data for ML training
- Data exploration, statistical analysis, data visualization

### DataFrame vs Relational

**Similarity**: Similar to database table or spreadsheet

**Operations**: Relational-like operators
- Apply function to all rows
- Filter rows by condition
- Group by columns, aggregate others
- Join (called "merge" in DataFrames)

**Key difference**:
- Relational: Declarative SQL
- DataFrames: Series of commands modifying structure/content
- Matches data scientist workflow: incrementally "wrangle" data into analysis form

**Scope**: Usually on scientist's local copy, may be shared

### DataFrame Capabilities Beyond Relational

**Operations far beyond relational databases**:
- Transform relational representation into matrix/multidimensional array
- Form many ML algorithms expect

**Workflow**: Incrementally evolve data from relational form to matrix

### Matrix Transformation Example (Figure 3-9)

**Input**: Relational table (users rate movies 1-5)

**Output**: Matrix (rows=users, columns=movies, values=ratings)

**Sparse**: No data for many user-movie combinations

**Properties**:
- Thousands of columns
- Not practical in relational DB
- DataFrames and sparse array libraries (NumPy) handle easily

### Converting Non-Numerical Data to Numbers

**Techniques**:

1. **Dates**: Scale to floating-point in suitable range

2. **Fixed set categories**: One-hot encoding
   - Create column per possible value (comedy, drama, horror)
   - Put 1 for matching value, 0 for others
   - Generalizes to items fitting multiple categories

### From Matrix to ML

**Once numerical matrix**: Amenable to linear algebra operations
- Foundation of ML algorithms

**Example**: Movie recommendation system
- Use user-movie rating matrix
- Apply ML to find user preferences
- Recommend unwatched movies

### DataFrame Flexibility

**Advantage**: Control evolution from relational to matrix representation
- Suits data analysis/model training goals

### Array Databases

**Specialized systems**: TileDB, others

**Purpose**: Store large multidimensional arrays of numbers

**Use cases**:
- Scientific datasets: Geospatial measurements (raster data, regularly-spaced grid)
- Medical imaging
- Astronomical telescope observations

### Time-Series Data

**Financial applications**: Use DataFrames for time-series (asset prices, trades over time)

**Popularity**: Data scientists' preference drove adoption in batch frameworks
- Apache Spark, Flink (Chapter 11)

---

## Summary

### Broad Overview Covered

**Models discussed**:
- Relational
- Document
- Graph-based
- Event sourcing
- DataFrames

### Key Model Characteristics

**Relational model**:
- Dominant for 50+ years
- Critical for data warehousing, business analytics
- Star/snowflake schemas, SQL ubiquitous

**Document model**:
- Targets self-contained JSON documents
- Rare relationships between documents
- Flexible schema

**Graph models**:
- Opposite direction: Many-to-many relationships common
- Query potentially needs multiple hops
- Recursive queries (Cypher, SPARQL, Datalog) natural

**DataFrames**:
- Generalize relational data to large column numbers
- Bridge databases and multidimensional arrays
- Foundation of ML, statistical analysis, scientific computing

### Cross-Model Representation

**Possible**: One model can emulate another (graph in relational)
- Result often awkward (e.g., SQL recursive queries)

### Database Specialization

**Trend**: Specialist databases for each model
- Query languages and storage engines optimized for model
- But databases expanding into neighboring areas:
  - Relational: JSON column support
  - Document: Relational-like joins
  - SQL: Gradual graph query support improvement

### Event Sourcing

**Different approach**: Append-only event log as source of truth
- Translated to read-optimized materialized views via CQRS
- Good for modeling complex business domain activities

### Schema Considerations

**Non-relational consistency**: Typically don't enforce schema
- Application still assumes structure
- Question: Explicit schema (schema-on-write) or implicit (schema-on-read)?

### Unmentioned Models

**Specialized niches**:
- **Genome databases**: Sequence similarity searches (GenBank, others)
- **Ledger systems**: Double-entry accounting model (TigerBeetle, blockchains/cryptocurrencies use distributed ledgers)
- **Full-text search**: Information retrieval specialty (discussed in Chapter 4, search indexes and vector search)

### Next Chapter

**Transition**: Chapter 4 discusses trade-offs in implementing data models
