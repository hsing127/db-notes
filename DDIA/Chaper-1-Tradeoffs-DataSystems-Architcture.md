# Chapter 1: Trade-Offs in Data Systems Architecture

## Core Concept: Data-Intensive Applications

**Definition**: An application where data management is one of the primary challenges in development (as opposed to compute-intensive systems where the challenge is parallelizing large computations).

**Key concerns**:
- Storing/processing large data volumes
- Managing data changes
- Ensuring consistency during failures and concurrency
- Ensuring high availability

**Why it matters**: You need to combine multiple storage/processing systems with different capabilities as data volume and query rates grow.

---

## Common Building Blocks for Data-Intensive Applications

- **Databases**: Store data so it can be found later
- **Caches**: Remember expensive operation results to speed up reads
- **Search indexes**: Allow filtering and keyword searching
- **Stream processing**: Handle events and data changes as they occur
- **Batch processing**: Periodically crunch accumulated data

---

## Operational vs Analytical Systems

### Operational Systems (OLTP - Online Transaction Processing)

| Aspect | Details |
|--------|---------|
| **Characteristic queries** | Point queries — fetch individual records by key |
| **Characteristic writes** | Create, update, delete individual records |
| **Typical users** | End users via web/mobile apps; backend engineers |
| **Query type** | Fixed, predefined by application |
| **Volume** | Lots of small queries |
| **Data represents** | Latest state (current point in time) |
| **Dataset size** | Gigabytes to terabytes |
| **Key constraint** | Users cannot run custom SQL (security risk + performance impact); only fixed queries baked into application code |

### Analytical Systems (OLAP - Online Analytical Processing)

| Aspect | Details |
|--------|---------|
| **Characteristic queries** | Aggregate queries — scan huge numbers of records, calculate statistics (count, sum, average) |
| **Characteristic writes** | Bulk import (ETL) or event stream |
| **Typical users** | Business analysts (BI), data scientists, internal decision-makers |
| **Query type** | Arbitrary, ad-hoc exploration; users write SQL by hand or use visualization tools (Tableau, Looker, Power BI) |
| **Volume** | Few queries, each complex |
| **Data represents** | History of events over time |
| **Dataset size** | Terabytes to petabytes |
| **Key advantage** | Freedom to write custom SQL; optimized for high-throughput processing |

### Specialized Analytics Roles Emerging

- **Data engineers**: Integrate operational and analytical systems; manage organization's data infrastructure
- **Analytics engineers**: Model and transform data to make it useful for business analysts/data scientists

---

## Data Warehouses

### Why Separate OLTP and Analytics?

- Analytical queries are expensive to run; would impact OLTP performance for regular users
- Data of interest is spread across multiple OLTP systems (**data silos problem**)
- Schemas optimized for OLTP are not ideal for analytics
- OLTP systems often in separate networks for security/compliance reasons

### What It Is

- A separate database containing a **read-only copy** of data from all OLTP systems
- Analysts can query freely without affecting operational performance
- Data flows in via **ETL (Extract-Transform-Load)**:
  1. Extract from operational systems
  2. Transform into analysis-friendly schema
  3. Load into warehouse
- **Alternative**: ELT (Extract-Load-Transform) — transform happens after loading

### Data Sources

- Can include external SaaS products (CRM, email marketing, payment systems) via APIs
- Specialist services (Fivetran, Singer, Airbyte) help bring external system data into warehouse

### HTAP (Hybrid Transactional/Analytical Processing)

- Aims to enable both OLTP and analytics in single system without ETL
- Often internally still has separate OLTP + analytical system hidden behind common interface
- **Does NOT replace data warehouses** — enterprise still has one warehouse for combining data across many operational systems

---

## Data Lakes

### Why They Emerged

- Data scientists need data in forms not suited to relational databases (not just rows/columns)
- Feature engineering (transforming data for ML models) requires custom code, hard to express in SQL
- NLP and computer vision tasks on text/images require flexible storage

### Key Differences from Data Warehouse

- Simply stores **files without imposing format, data model, or schema**
- Can hold:
  - Database records (Avro/Parquet files)
  - Text, images, videos
  - Sensor readings
  - Sparse matrices
  - Feature vectors
  - Genome sequences
- **Often cheaper**: Uses commoditized file storage (object stores like S3)
- Follows **"sushi principle"**: raw data is better — each consumer can transform raw data into form they need

### Data Pipelines & ETL Generalization

- ETL processes expanded to "data pipelines"
- Data lake often intermediate stop:
  - Raw operational data → Data lake → Data warehouse (for BI)
  - OR Data lake → Direct use by data scientists

---

## Beyond Data Lakes: Modern Analytics Architecture

- **DataOps Manifesto**: Increased focus on management and operations of analytical systems
- **Privacy/Compliance**: GDPR and CCPA compliance required; impacts system design
- **Stream processing**: Analytics increasingly available as event streams, not just files/tables — enables response in seconds vs periodic daily rerun
- **Reverse ETL**: Analytical system outputs (e.g., ML models trained on analytics data) deployed to operational systems for real-time recommendations

---

## Systems of Record vs Derived Data

### System of Record (Source of Truth)

- Holds **authoritative/canonical version** of data
- New data written here first
- Each fact represented exactly once (**normalized**)
- If discrepancy exists elsewhere, system of record is correct by definition

### Derived Data Systems

- Result of transforming/processing existing data from another system
- If lost, can be **re-created from original source**
- Examples:
  - Caches
  - Denormalized values
  - Indexes
  - Materialized views
  - Trained ML models
- **Technically redundant** but essential for read performance
- Can derive multiple datasets from single source for different viewpoints

### Key Insight

- Distinction depends on **how you use the tool**, not the tool itself
- When data in one system derives from another, you need a process to update derived data when source changes
- Many databases assume single-database use, making multi-system integration difficult

---

## Cloud vs Self-Hosting

### The Trade-Off Framework

- **Build in-house**: Things that are core competencies or competitive advantages
- **Buy/outsource**: Non-core, routine, commonplace things

### Spectrum of Options

1. **Bespoke software** you write and run in-house
2. **Off-the-shelf software** (open source/commercial) you self-host on your own hardware or VM
3. **Cloud services** (SaaS) operated by external vendor, accessed via API/web interface

### Pros of Cloud Services

- Saves time and money (in many situations)
- Easier/quicker if you don't know how to deploy/operate the system
- Saves on hiring/training operations staff
- Outsourcing can yield better service (provider gains expertise from many customers)
- **Excellent for variable load**: Easy to scale resources up/down; pay for what you use

### Cons of Cloud Services

- **No control**: Can't implement missing features yourself
- **Downtime**: Only option is wait for recovery
- **Debugging difficulty**: Limited access to performance metrics, server logs, system internals
- **Vendor lock-in**: Hard to switch if service becomes expensive, shuts down, or changes unfavorably; limited standard APIs
- **Geopolitical risk**: If provider in different country, sanctions could lock you out
- **Security trust**: Must trust vendor with data security; complicates compliance

### When Self-Hosting May Be Cheaper

- You already have experience operating the system
- Your load is predictable (doesn't fluctuate wildly)
- You have skilled operations staff

### When Cloud Services Make Sense

- You need a system you don't know how to operate
- Load varies significantly (analytical systems especially)
- You value ease/speed of setup over customization

---

## Cloud Native System Architecture

### Definition

- Architecture designed from ground up to take advantage of cloud services
- Shows advantages: better performance, faster failure recovery, quick resource scaling, larger dataset support

### Key Architectural Pattern: Layering Services

- Cloud native systems build upon lower-level cloud services to create higher-level services
- **Object storage** (S3, Azure Blob Storage): Store large files with automatic distribution across machines; hides underlying physical infrastructure
- Many other services built on top of object storage (e.g., Snowflake built on S3)

### Separation of Storage and Compute

- **Traditional**: Disk storage and computation on same machine; RAID for redundancy
- **Cloud native**: Treats local disks as ephemeral cache (lost if instance fails/resizes)
- Uses cloud services for true durability: object storage (S3) for long-term storage
- **Disaggregation**: S3 only stores; analysis code runs separately, transferring data over network
- Databases manage small values separately, store large data blocks in object stores

### Multitenancy

- Multiple customers' data/computation handled on same shared hardware
- Enables:
  - Better hardware utilization
  - Easier scalability
  - Easier management
- Requires: Careful engineering so one customer's activity doesn't affect others' performance/security

---

## Operations in the Cloud Era

### Traditional Role (DBAs/Sysadmins)

- Significant work at individual machine level
- Capacity planning (disk space, etc.)
- Provisioning, moving services, OS patches

### Modern Role (DevOps/SRE)

- Shifted from individual machines to services
- **Emphasis on**:
  - Automation
  - Ephemeral VMs
  - Frequent updates
  - Learning from incidents
  - Preserving organizational knowledge
- Cloud services hide individual machines behind APIs (no capacity planning for storage, metered billing)

### Bifurcation of Operations

- **Infrastructure companies' ops teams**: Specialize in reliable service delivery to many customers
- **Cloud service customers' ops teams**: Focus on choosing appropriate services, integrating them, migrating between them

### Financial Planning Replaces Capacity Planning

- Need to track resources used for which purpose (avoid wasting money)
- Must know cloud service quotas/limits before hitting them
- Performance optimization becomes cost optimization

### What Still Can't Be Outsourced

- Maintaining application security and library updates
- Managing interactions between your services
- Monitoring service load
- Troubleshooting problems (performance degradation, outages)

---

## Distributed vs Single-Node Systems

### Why Use Distributed Systems?

**Inherent distribution**
- Multi-user apps with multiple devices must communicate via network

**Requests between cloud services**
- Data in one service, processed in another — requires network transfer

**Fault tolerance/high availability**
- Multiple machines provide redundancy; one can fail without system going down

**Scalability**
- Data volume/computing needs exceed single machine capacity; spread load across multiple

**Latency**
- Geographically distributed users served from nearby servers (avoid worldwide packet travel)

**Elasticity**
- Scale up/down to match demand; pay only for resources used (cloud benefit)

**Specialized hardware**
- Different workloads use different hardware:
  - Object store: many disks/few CPUs
  - Analysis: many CPU/RAM
  - ML: GPUs

**Legal compliance**
- Data residency laws require data stored/processed within specific countries

**Sustainability**
- Flexibility on where/when to run jobs allows use of renewable electricity

### Problems with Distributed Systems

- **Network failure risk**: Every network request may timeout; don't know if request was received; retrying may not be safe
- **Network latency**: Network call vastly slower than in-process function call; for large data, better to move computation to data than move data to computation
- **Not always faster**: Simple single-threaded program may outperform 100+ CPU core cluster in some cases
- **Troubleshooting difficulty**: Hard to diagnose slow responses; requires **observability** tools (OpenTelemetry, Zipkin, Jaeger for tracing)
- **Consistency challenge**: Each service with own database makes maintaining cross-service data consistency an application problem (not database's)
- **Distributed transactions**: Rarely used in microservices (counter to goal of service independence; many databases don't support them)

### Single-Node Systems Revival

- Modern CPUs, memory, disks are larger/faster/more reliable
- Single-node databases (DuckDB, SQLite, KùzuDB) enable many workloads to run on single machine
- Often simpler and cheaper than distributed setup

---

## Microservices and Serverless

### Microservices Architecture

- Service has **one well-defined purpose**; exposes API for network access; one team maintains it
- Complex app decomposed into multiple interacting services, each managed separately

**Advantages**:
- Independent updates (less team coordination)
- Hardware resources allocated per service need
- Implementation details hidden; can change without affecting clients
- Common for each service to have own database (avoids shared database becoming API, prevents query interference)

**Disadvantages**:
- Adds complexity; testing requires running all dependencies
- Each service needs infrastructure: deployment, resource adjustment, logging, monitoring, alerting
- Orchestration frameworks (Kubernetes) needed
- Microservice APIs hard to evolve; adding/removing fields breaks clients; failures often discovered late

**Primary purpose**: Technical solution to people problem — enable independent team progress without coordination (valuable in large companies; overhead in small ones)

### Serverless/FaaS (Function as a Service)

- Cloud vendor manages infrastructure automatically
- Allocates/frees resources based on incoming requests
- **Billing model**: Metered — pay only for execution time, not provisioned resources
- **Trade-offs**:
  - Time limits on execution
  - Limited runtimes
  - Potential slow start times on first invocation
- **Terminology note**: "Serverless" misleading — code still runs on servers; different execution may use different server

---

## Cloud Computing vs Supercomputing (HPC)

### Key Differences

| Aspect | Supercomputers (HPC) | Cloud Computing |
|--------|---------------------|-----------------|
| **Workloads** | Computationally intensive scientific (weather, climate, molecular dynamics, optimization) | Online services, business data systems, user-facing systems needing high availability |
| **Failure handling** | Checkpoint state; stop entire cluster, repair, restart from checkpoint | Must stay available; stopping entire cluster undesirable |
| **Networking** | Shared memory + RDMA (high bandwidth/low latency, high trust assumption) | IP/Ethernet (shared by untrusting organizations; require isolation, encryption, authentication) |
| **Network topology** | Specialized topologies (multidimensional meshes, toruses) for known HPC communication patterns | Clos topologies for high bisection bandwidth |
| **Geography** | Nodes close together | Nodes across multiple regions |

---

## Data Systems, Law, and Society

### Regulatory Landscape

- **GDPR** (2018, EU): Residents have greater control/legal rights over personal data
- **Similar regulations**: CCPA and others adopted worldwide
- **AI regulations**: EU AI Act restricts how personal data used

### Ethical Concerns

- Automated systems make consequential decisions (loans, insurance, job interviews, crime suspicion)
- Social media influences news consumption, political opinions, election outcomes
- Engineers share responsibility for ethical impact and legal compliance

### Legal Implications for System Design

- **Right to be forgotten (GDPR)**: Individuals can request data erasure
- **Challenge**: Many systems use immutable/append-only logs; how to delete from immutable files?
- **Derived data deletion**: How to delete data from ML training datasets?
- **No prescriptive technology mandates**: GDPR sets high-level principles (interpretation required); no clear "GDPR-compliant" tech list

### Data Minimization (Datensparsamkeit)

- Only collect data for specified, explicit purpose
- Cannot reuse for other purposes
- Must not keep longer than necessary
- **Principle**: Counters "big data" philosophy of storing speculatively
- **Practical reasoning**: Some data simply not worth storing — risks of liability, reputational damage, legal costs/fines outweigh storage benefits

### Real Safety Risks

- Data revealing criminalized behaviors (homosexuality in some countries, abortion seeking) creates safety risks
- Location data or IP logs easily reveal sensitive movements/intentions

### Compliance Standards

- **PCI DSS**: Payment Card Industry standards for payment processing
- **SOC Type 2**: Service Organization Control standards increasingly required by software vendors' buyers
- Both require third-party audits for verification

### Business vs User Needs Balance

- Important to balance business needs against rights/needs of people whose data collected/processed
- All engineers share responsibility for this balance

---

## Key Terminology

**Frontend**
- Client-side code running in web browser (or mobile app)
- Manages single user's data

**Backend**
- Server-side code handling user requests
- Manages data for all users via databases/data infrastructure

**Application code**
- Often stateless; forgets request details after handling
- Persistent info stored in client or server-side infrastructure
