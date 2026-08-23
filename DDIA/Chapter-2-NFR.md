# Chapter 2: Defining Nonfunctional Requirements

## Overview

### Functional vs Nonfunctional Requirements

**Functional requirements**: What the application must do—screens, buttons, operations to fulfill software purpose.

**Nonfunctional requirements**: How the application must perform—fast, reliable, secure, legally compliant, maintainable. Often implicit but just as important as functionality.

---

## Case Study: Social Network Home Timelines

### Scenario Assumptions

- **Post volume**: 500 million posts per day = ~5,800 posts/second average
- **Peak rate**: Can spike to 150,000 posts/second
- **Social graph**: Average user follows 200 people and has 200 followers
  - Wide range: most have handful of followers; celebrities have 100M+ followers
- **Main operation**: Home timeline — display recent posts by people user is following

### Initial Naive Approach: Polling

**SQL Query Example**:
```sql
SELECT posts.*, users.* FROM posts
JOIN follows ON posts.sender_id = follows.followee_id
JOIN users ON posts.sender_id = users.id
WHERE follows.follower_id = current_user
ORDER BY posts.timestamp DESC
LIMIT 1000
```

**Issues**:
- Assume 10 million concurrent users; each polls every 5 seconds = 2 million queries/second
- Each query fetches recent posts from 200 followed accounts = 400 million lookups/second
- Users following tens of thousands of accounts face very expensive queries

### Improved Approach: Materialized Timelines

**Strategy**:
1. Precompute home timeline for each user (cache recent posts from accounts they follow)
2. When user makes a post, push it to all followers' timelines (like mailbox delivery)
3. User logs in → receive precomputed timeline instantly
4. Client subscribes to stream of new posts for real-time updates

**Trade-off**: More work on writes to speed up reads

**Fan-out**: Factor by which number of requests increases from initial request
- 5,800 posts/second × 200 followers average = ~1.1 million home timeline writes/second
- Much better than 400 million lookups/second

**Handling Spikes**:
- During load spikes, enqueue timeline deliveries
- Timelines remain fast since served from cache
- Slight delay acceptable for posts to appear

**Extreme Cases**:

1. **User following huge number of accounts**: High write rate to their materialized timeline
   - Solution: Drop some writes, show only sample of posts from accounts followed

2. **Celebrity account with millions of followers**: Massive work inserting post into millions of timelines
   - Solution: Store celebrity posts separately, merge with timeline on read (no fan-out cost)

---

## Describing Performance

### Two Main Performance Metrics

**Response Time**
- Elapsed time from user request to receiving answer
- Measured in: seconds, milliseconds, microseconds
- What users directly experience

**Throughput**
- Number of requests per second (or data volume per second) system processes
- For given hardware, there is maximum throughput
- Determines required computing resources and cost
- Measured in: "things per second"

### Response Time vs Throughput Relationship

- **Low load**: Low response times
- **Increasing load**: Response times increase due to queueing
  - CPU likely already processing earlier request
  - Incoming request must wait
- **Approaching maximum capacity**: Response times increase sharply due to queueing delays
- **Near overload**: System can enter vicious cycle

### When Overloaded System Won't Recover

**Metastable Failure**: System enters vicious cycle and won't recover without reboot/reset

**Mechanism**:
1. Long queue of waiting requests → response times increase
2. Client timeouts → clients resend requests (retry storm)
3. Increased request rate → problem gets worse
4. System remains overloaded even after load reduction

**Prevention Techniques**:

*Client-side*:
- **Exponential backoff**: Increase and randomize time between retries
- **Circuit breaker**: Temporarily stop sending requests to service returning errors/timeouts
- **Token bucket algorithm**: Rate limit outgoing requests

*Server-side*:
- **Load shedding**: Proactively reject requests when approaching overload
- **Backpressure**: Send responses asking clients to slow down
- **Queueing/load balancing algorithms**: Choice matters for performance

### Response Time Focus

- Most important to users
- Throughput determines required computing resources (cost)
- Scalability: Ability to increase maximum throughput by adding resources

---

## Latency and Response Time

### Terminology (specific definitions)

**Response Time**
- What client sees; includes ALL delays in system

**Service Time**
- Duration service actively processes client request

**Queueing Delay**
- Time request waits for resources to become available
- Can occur at multiple points:
  - Waiting for CPU availability
  - Response packet buffered before network transmission

**Latency**
- Catchall term for time request not actively processed (request is latent)
- Network latency/delay: time request/response travels through network

### Sources of Response Time Variation

Response times vary significantly request-to-request (same request repeated). Random delays from:
- Context switches to background processes
- Network packet loss and TCP retransmission
- Garbage collection pauses
- Page faults (disk reads)
- Mechanical vibrations in server rack

**Head-of-line Blocking**: Small number of slow requests hold up processing of subsequent requests
- Even fast service times appear slow due to waiting for prior request
- **Important**: Measure response times on client side (includes queueing delay)

---

## Average, Median, and Percentiles

### Why Not Use Average?

- Average (arithmetic mean) sums response times and divides by count
- Useful for estimating throughput limits
- **Poor for "typical" response time**: Doesn't tell how many users actually experienced that delay
- Heavily affected by outliers

### Percentiles (Better Metric)

**Median (50th percentile / p50)**
- Halfway point when sorting response times fastest to slowest
- Example: p50 = 200ms means half requests return in <200ms, half take longer
- Good indicator of typical user experience

**Higher Percentiles**
- p95: 95 of 100 requests < 1.5s; 5 of 100 requests ≥ 1.5s
- p99 and p999: Common for SLOs
- Identify outliers and worst-case scenarios

### Tail Latencies (High Percentiles)

- Directly affect user experience
- Amazon uses p999 SLOs for internal services
- Often customers with most data (highest value) experience slowest requests
- Optimizing very high percentiles (p99.99) diminishing returns
  - Hard to control (random external events)
  - Minimal benefit-cost ratio

### Percentile Issues

**Beware**: Averaging percentiles mathematically meaningless (reduces time resolution or combines machines)
- Correct aggregation: Add histograms, then calculate percentiles

---

## User Impact of Response Times

### Research Findings (Inconsistent/Unreliable Data)

- 2006 Google: 400ms → 900ms slowdown = 20% traffic/revenue drop
- 2009 Google: 400ms increase = only 0.6% fewer searches/day
- 2009 Bing: 2 second load time increase = 4.3% ad revenue drop
- 2010s Akamai: 100ms increase = up to 7% conversion rate reduction (but study methodology flawed — fastest pages often error pages)
- 2011 Yahoo: 1.25+ second difference = 20-30% more clicks on fast searches (better controlled study)

**Conclusion**: Hard to quantify exact impact; clear that speed matters but inconsistent data

---

## High Percentile Metrics in Practice

### Tail Latency Amplification

**Scenario**: Backend service requires multiple parallel calls to serve single end-user request

**Problem**: Just ONE slow backend call makes entire end-user request slow
- Even if calls made in parallel, must wait for slowest call
- If small percentage of backend calls slow, likelihood of slow end-user request increases significantly

**Impact**: Higher proportion of end-user requests slow due to probability

### SLOs and SLAs

**SLO (Service Level Objective)**
- Target performance and availability for service
- Example: Median response time <200ms, p99 <1 second, 99.9% non-error responses
- Internal commitment

**SLA (Service Level Agreement)**
- Contract specifying what happens if SLO not met
- Example: Customers entitled to refund if SLO breached
- Legal/financial commitment

**Defining Good SLOs/SLAs**: Not straightforward in practice

### Computing Percentiles

**Simplest implementation**:
- Keep list of response times in time window
- Sort each period (e.g., every minute)
- Calculate percentiles

**More efficient**:
- Open source percentile estimation libraries
- Minimal CPU/memory cost while approximating percentiles
- Examples: HdrHistogram, t-digest, OpenHistogram, DDSketch

---

## Reliability and Fault Tolerance

### What Reliability Means

**Reliable system expectations**:
- Application performs function user expected
- Tolerates user mistakes and unexpected usage
- Good performance under expected load/data volume
- Prevents unauthorized access and abuse

**Reliability definition**: Continuing to work correctly, even when things go wrong

### Faults vs Failures

**Fault**
- Part of system stops working correctly
- Example: Hard drive malfunction, machine crash, external service outage
- Often tolerable if system designed properly

**Failure**
- System as a whole stops providing required service (doesn't meet SLO)
- Example: System outage visible to users

**Key distinction**: Same thing at different levels
- Hard drive fails → hard drive failure
- Hard drive is only system → system failure
- Hard drive in multi-drive system → system fault (if other copies exist)

### Fault Tolerance

**Definition**: System continues providing service despite certain faults occurring

**Single Point of Failure (SPOF)**: Part where fault escalates to cause whole system failure

**Limits to Fault Tolerance**:
- Always limited to certain types/numbers of faults
- Example: System tolerates max 2 disk failures, 1 node crash out of 3
- Can't tolerate all possible faults (impossible to prevent "Earth swallowed by black hole")

### Fault Injection

**Concept**: Deliberately trigger faults to test fault tolerance
- Randomly kill processes without warning
- Ensures fault-tolerance machinery exercised and tested
- Increases confidence handling real failures correctly

**Chaos Engineering**: Discipline improving fault-tolerance confidence through deliberate fault injection

### Prevention vs Tolerance

**General principle**: Prefer tolerating faults over preventing faults

**Exception**: Security — if attacker compromises system, event cannot be undone
- Prevention better than cure in security context
- Book focuses on faults that can be cured

---

## Hardware and Software Faults

### Hardware Fault Rates

**Hard drives**:
- 2-5% fail per year
- 10,000-disk cluster = expect 1 failure/day average
- Getting more reliable but rates remain significant

**SSDs**:
- 0.5-1% fail per year
- Automatic correction of small bit errors
- Uncorrectable errors: ~once/year per drive (even newer drives)
- Higher error rate than magnetic drives

**Other components**:
- Power supplies, RAID controllers, memory modules: Less frequent than drives
- Still occur regularly in large systems

**CPU errors**:
- ~1 in 1,000 machines have CPU core occasionally computing wrong result
- Likely manufacturing defects
- May cause crash or silently return wrong result

**RAM errors**:
- Corrupted by random events (cosmic rays) or physical defects
- Even with ECC memory: >1% machines get uncorrectable error/year
- Typically causes machine crash, memory module replacement needed
- Pathological access patterns can flip bits with high probability

**Datacenter failures**:
- Entire datacenter unavailable (power outage, network misconfiguration)
- Permanent destruction possible (fire, flood, earthquake)
- Solar storms can damage power grids and undersea cables
- Large-scale failures rare but catastrophic if service can't tolerate

### Rare but Common at Scale

- Small systems: Often ignore hardware faults (easy to replace hardware)
- Large-scale systems: Hardware faults happen frequently = normal operation

### Tolerating Hardware Faults Through Redundancy

**Redundancy techniques**:
- RAID: Spread data across multiple disks in same machine (failed disk ≠ data loss)
- Dual power supplies
- Hot-swappable CPUs
- Datacenter batteries and diesel generators
- Result: Single machine can run uninterrupted for years

**Assumption**: Faults are independent

**Reality**: Significant correlations between failures
- Entire server rack or datacenter unavailability common
- More frequent than expected

### Cloud Approach: Software-Level Fault Tolerance

**Traditional focus**: Reliability of individual machines (hardware redundancy)

**Cloud focus**: High availability by tolerating faulty nodes at software level

**Availability zones**: Identify physically co-located resources
- Same location resources more likely fail together
- Geographically separated resources more resilient

**Fault-tolerance techniques** (Chapters 6, 10, others):
- Tolerate loss of entire machines, racks, availability zones
- Machine in one datacenter takes over when another fails

**Operational advantages of multi-node fault tolerance**:
- Single-node system: Requires downtime for reboots (OS patches)
- Multi-node system: Rolling upgrades (restart one node at time, no user impact)

### Software Faults

**Characteristics**:
- Often highly correlated (same software, same bugs on many nodes)
- Harder to anticipate than hardware faults
- Cause more system failures than uncorrelated hardware faults

**Examples**:

*Systematic bugs*:
- Bug causing every node fail in particular circumstances
- June 30, 2012 leap second: Linux kernel bug crashed Java apps, took down internet services
- Firmware bug: All SSDs of certain models fail after exactly 32,768 hours (~4 years)

*Resource exhaustion*:
- Runaway process consuming shared resource (CPU, memory, disk, bandwidth, threads)
- Large request kills process; client library bug causes unexpected load

*Service degradation*:
- Depended-on service slows down, becomes unresponsive, returns corrupted responses

*Emergent behavior*:
- Interaction between systems produces unexpected behavior not seen in isolation

*Cascading failures*:
- Problem in one component → another overloaded/slowed → brings down third component
- Chain reaction effect

### Root Cause: Dormant Assumptions

- Bugs often dormant for long time until triggered
- Revealed software makes assumptions about environment
- Usually true, eventually stops being true for some reason

### Solutions (No Quick Fix)

- Carefully think about assumptions and interactions
- Thorough testing
- Process isolation
- Allow crashes and restarts
- Avoid feedback loops (retry storms)
- Measure, monitor, analyze production behavior

---

## Humans and Reliability

### Human Error as Failure Cause

**Study finding**: Configuration changes by operators = leading cause of outages
- Hardware faults accounted for only 10-25% of cases

**Common approach**: Label as "human error," control behavior through procedures

**Problem**: Counterproductive. "Human error" is symptom, not root cause

### Sociotechnical System View

**Concept**: "Human error" not really cause but symptom of sociotechnical system problem

- Humans don't just follow rules; creative and adaptive
- This strength leads to unpredictability and mistakes
- Complex systems have emergent behavior
- Unexpected interactions between components cause failures

### Technical Mitigations for Human Mistakes

- Thorough testing (handwritten + property testing on random inputs)
- Rollback mechanisms (quickly revert config changes)
- Gradual rollouts (new code rolled out slowly)
- Detailed monitoring and observability tools
- Well-designed interfaces (encourage right thing, discourage wrong)

**Cost**: Requires time and money investment

**Reality**: Organizations often prioritize revenue-generating features over resilience

### Blameless Postmortems

**Principle**: After incident, encourage sharing full details without fear of punishment

**Goal**: Learn how to prevent similar problems in future

**Benefits**:
- Organizational learning
- May uncover need to change business priorities
- May reveal need for investment in neglected areas
- May surface systemic issues to management

### Investigating Incidents

**Avoid simplistic answers**:
- "Bob should have been more careful" — not productive
- "Rewrite backend in Haskell" — also not productive

**Better approach**:
- Learn details of how sociotechnical system works
- Get perspective from people working with it daily
- Improve system based on feedback
- Suspicious of surface-level blame

---

## How Important Is Reliability?

### Reliability Beyond Critical Systems

Not just for nuclear power stations and air traffic control

**Business applications**:
- Bugs cause lost productivity and legal risks (incorrect figures)
- Ecommerce outages: lost revenue and reputation damage

**Consumer applications**:
- Temporary outages (minutes/hours): Often tolerable
- Permanent data loss/corruption: Catastrophic

**Example**: Parent storing all photos/videos of children in photo app
- How would they feel if database corrupted?
- Do they know how to restore from backup?

### Real-World Consequence: Post Office Horizon Scandal

**Timeline**: 1999-2019

**What happened**:
- Hundreds of Post Office branch managers convicted of theft/fraud
- Accounting software showed shortfall in accounts
- Many shortfalls caused by software bugs
- Many convictions overturned

**Impact**: Probably largest miscarriage of justice in British history

**Root cause**: English law assumption that computers operate correctly
- Evidence from computers assumed reliable unless proven otherwise
- Software engineers know software has bugs; victims didn't

**Consequences**:
- Wrongful imprisonment
- Bankruptcy
- Suicide among wrongfully convicted

### When to Accept Lower Reliability

**Situations**:
- Developing prototype for unproven market
- Want to reduce development cost

**Critical**: Be conscious of trade-off and keep potential consequences in mind

---

## Scalability

### Definition and Context

**Scalability**: Ability to cope with increased load

**Common misconception**: "You're not Google/Amazon. Stop worrying about scale."
- Depends on application type

**Early-stage startups**:
- Goal: Keep simple and flexible
- Easy to modify features as learn about customers
- Worry about hypothetical future scale: Wasted effort (at best) or inflexible design (at worst)
- Scale when you become popular and hit bottlenecks

### Scalability is Multidimensional

**Meaningless**: "X is scalable" or "Y doesn't scale"

**Right questions**:
- If system grows in particular way, options for coping?
- How to add computing resources for additional load?
- When will current architecture hit limits?

### Relationship Between Load and Architecture

**Pattern**: Architecture appropriate for one load level unlikely to handle 10× that load

**Implication**: Rethink architecture at every order-of-magnitude load increase

**Planning**: Usually not worth planning scalability needs >1 order of magnitude in advance

---

## Understanding Load

### Load Metrics

**Throughput metrics**:
- Requests per second to service
- Gigabytes new data arriving per day
- Shopping cart checkouts per hour

**Peak metrics**:
- Simultaneously online users

### Load Characteristics Affecting Access Patterns

- Ratio of reads to writes in database
- Cache hit rate
- Number of data items per user (followers, etc.)
- Whether average or extreme cases dominate bottleneck

### Investigating Load Impact

Two perspectives:

1. **Keep resources constant, increase load**: How does performance change?
2. **Keep performance constant, increase load**: How much more resources needed?

### Scalability Goals

**Usually**: Keep performance within SLA requirements while minimizing cost

**Cost scales with resources**:
- Larger hardware = higher cost
- Different hardware types = different cost-effectiveness
- Cost-effectiveness changes as new hardware available

### Linear Scalability

**Linear scalability**: Doubling resources = handles double the load with same performance
- Considered good
- Difficult to achieve

**Better than linear**: Occasionally possible
- Economies of scale
- Better peak load distribution

**More likely**: Sublinear (cost grows faster than linearly)
- More data → more work per request
- Other inefficiencies

---

## Shared-Memory, Shared-Disk, and Shared-Nothing Architectures

### Vertical Scaling (Scale Up)

**Approach**: Move to more powerful machine
- More CPU cores, RAM, disk space
- Called "vertical scaling" or "scaling up"

**Parallelism on single machine**:
- Multiple processes/threads
- Shared-memory architecture: All threads in same process share RAM

**Problem**: Cost grows faster than linearly
- Machine with 2× resources typically costs significantly more than 2×
- Bottlenecks prevent actually handling 2× load

### Shared-Disk Architecture

**Approach**: Multiple machines, independent CPUs/RAM, shared disk array
- Connected via fast network (NAS or SAN)

**Historical use**: On-premises data warehousing

**Limitations**: Contention and locking overhead limit scalability

### Shared-Nothing Architecture (Preferred Modern Approach)

**Also called**: Horizontal scaling or scaling out

**Architecture**: Distributed system with multiple nodes
- Each node has own CPUs, RAM, disks
- Coordination at software level via conventional network

**Advantages**:
- Potential for linear scalability
- Use best price/performance ratio hardware (especially cloud)
- Easy to adjust hardware as load changes
- Greater fault tolerance (distributed across datacenters/regions)

**Disadvantages**:
- Requires explicit sharding
- All complexity of distributed systems

### Cloud Native Hybrid Approach

**Emerging pattern**: Separate storage and compute services
- Multiple compute nodes share access to same storage service
- Similar to shared-disk but avoids older system scalability problems
- Storage service offers specialized API (not filesystem/block device abstraction)

---

## Principles for Scalability

### No Magic Scaling Sauce

**Fact**: Architecture at large scale highly specific to application
- No generic one-size-fits-all scalable architecture
- Example: 100,000 requests/second (1KB each) vs 3 requests/minute (2GB each)
  - Same throughput (100MB/s) but vastly different architectures

### Rethinking Architecture at Scale Increases

- Architecture for load X unlikely suitable for 10X
- Need new architecture with each order-of-magnitude increase
- Requirements likely evolve → not worth planning beyond 1 order of magnitude ahead

### Decomposition Principle

**Core principle**: Break system into smaller components operating largely independently

**Applies to**:
- Microservices
- Sharding
- Stream processing
- Shared-nothing architectures

**Challenge**: Knowing where to draw boundaries (together vs apart)

### Pragmatism Over Premature Optimization

**Principles**:
- If single-machine database works → preferable to complicated distributed setup
- Autoscaling cool but manually scaled system may have fewer surprises if load predictable
- 5 services simpler than 50
- Good architectures = pragmatic mixture of approaches

---

## Maintainability

### Why Maintenance Matters

**Software lifecycle**:
- Doesn't wear out or suffer material fatigue
- Requirements constantly evolve
- Environment changes (dependencies, platforms)
- Bugs need fixing

**Cost reality**: Majority of software cost in ongoing maintenance, not initial development

**Maintenance tasks**:
- Fixing bugs
- Keeping systems operational
- Investigating failures
- Adapting to new platforms
- Modifying for new use cases
- Repaying technical debt
- Adding new features

### Legacy System Challenges

**Complexity**:
- Long-running systems use outdated technologies (mainframes, COBOL)
- Institutional knowledge lost as people leave
- Need to fix others' mistakes
- Systems intertwined with human organizations

**People problem as much as technical problem**

### Three Maintenance Principles

**Operability**
- Make it easy for organization to keep system running smoothly

**Simplicity**
- Make it easy for new engineers to understand system
- Use well-understood, consistent patterns and structures
- Avoid unnecessary complexity

**Evolvability**
- Make it easy for engineers to make changes
- Adapt and extend for unanticipated use cases
- Support changing requirements

---

## Operability: Making Life Easy for Operations

### Operations' Role

**Goals**:
- Reliably deliver services to users
- Configure infrastructure, deploy applications
- Maintain stable production environment
- Monitor, diagnose problems affecting reliability

### Automation Tradeoffs

**Self-hosted systems**:
- Manual maintenance expensive at scale (thousands of machines)
- Automation essential
- Automation two-edged sword: Harder to troubleshoot when auto system fails

**When automation helps**:
- Handles routine cases
- Frees operations for high-value activities

**When automation hurts**:
- Complex edge cases need manual intervention
- Rare failure scenarios often most complex
- Broken automated system harder to troubleshoot than manual system

**Sweet spot**: Depends on specific application and organization
- More automation not always better
- Some amount important for scalability

### Good Operability Practices

Data systems can help by:

- **Monitoring and observability**:
  - Allow tools to check key metrics
  - Observability tools provide runtime behavior insights
  - Commercial and open source tools available

- **Avoiding single points of failure**:
  - Machines can be taken down for maintenance
  - System as whole continues running

- **Documentation and clear operational model**:
  - "If I do X, Y will happen"
  - Easy to understand

- **Good defaults with override capability**:
  - Sensible defaults
  - Administrators can override when needed

- **Self-healing with manual control**:
  - Automatic recovery appropriate
  - Administrators maintain manual control when needed

- **Predictable behavior**:
  - Minimize surprises

---

## Simplicity: Managing Complexity

### Complexity Problem

**Reality**: Small projects simple and expressive; large projects become complex and difficult

**Consequences**:
- Slows everyone working on system
- Increases maintenance cost
- Risk of introducing bugs when making changes
- Hidden assumptions, unintended consequences, unexpected interactions overlooked
- Described as "big ball of mud"

### Benefits of Simplicity

**Simplicity drives maintainability**:
- Easier to understand → lower cost
- Reduces bug risk
- Key goal for systems we build

### Challenge of Defining Simplicity

**Subjectivity**: No objective simplicity standard
- One system: Complex implementation, simple interface
- Another system: Simple implementation, exposed internals
- Which is simpler?

### Essential vs Accidental Complexity

**Attempt to categorize complexity**:

**Essential complexity**: Inherent in problem domain

**Accidental complexity**: From tooling limitations

**Problem**: Distinction flawed — boundary shifts as tooling evolves

### Abstraction for Managing Complexity

**Tool**: Abstraction hides implementation detail behind clean facade

**Benefits**:
- Used across wide range of applications
- Reuse more efficient than reimplementing
- Quality improvements benefit all apps using abstraction

**Examples**:
- High-level programming languages: Hide machine code, registers, system calls
- SQL: Hide on-disk/in-memory structures, concurrent requests, crash inconsistencies
- Design patterns, domain-driven design (DDD): Application-specific abstractions

**This book covers**: General-purpose abstractions (transactions, indexes, event logs)

---

## Evolvability: Making Change Easy

### Inevitable Change

**Reality**:
- System requirements never stay unchanged
- Constant flux: new facts, unanticipated use cases, business priority changes
- User requests, new platforms, legal/regulatory changes
- Growth forces architectural changes

### Organizational Framework: Agile

**Agile working patterns**: Framework for adapting to change

**Technical tools and processes**:
- Test-driven development (TDD)
- Refactoring

**System-level agility**: Different term used for data systems level

### Evolvability Definition

**Evolvability**: Ease modifying data system and adapting to changing requirements

**Key factor**: Linked to simplicity and abstractions

**Design**: Loosely coupled, simple systems easier to modify than tightly coupled, complex ones

### Irreversibility Problem

**Challenge**: Some actions irreversible (difficult to undo)

**Example**: Migrating from one database to another
- Can't switch back if problems with new system
- Stakes much higher than reversible changes
- Requires careful planning

**Principle**: Minimize irreversibility improves flexibility

---

## Summary

### Topics Covered

**Nonfunctional requirements examined**:
- Performance
- Reliability
- Scalability
- Maintainability

**Key ideas**:
- Metrics and terminology relevant to rest of book
- Social network case study illustrated scalability challenges

### Performance

- **Metrics**: Response time percentiles, throughput
- **Relationship**: Response time increases as load approaches capacity (queueing)
- **Measurement**: Use percentiles (p50, p95, p99, p999) not averages
- **SLOs/SLAs**: Define expected performance and availability
- **Practical**: Tail latency amplification when multiple backend calls needed

### Reliability

- **Definition**: Continuing to work correctly despite faults
- **Fault tolerance**: System tolerates certain faults via redundancy and software techniques
- **Hardware faults**: Common at scale; tolerate via redundancy
- **Software faults**: Highly correlated; harder to handle
- **Human failures**: Result of sociotechnical system issues; blameless postmortems help learning
- **Importance**: Reliability failures have serious consequences

### Scalability

- **Definition**: Ability to increase throughput by adding resources
- **Load understanding**: Measure and understand current load first
- **Architectures**: Shared-nothing (horizontal scaling) preferred for distributed systems
- **Decomposition**: Break into independent components

### Maintainability

- **Operability**: Make it easy to keep system running; support monitoring and observability
- **Simplicity**: Reduce complexity through abstractions
- **Evolvability**: Make changes easy; minimize irreversibility
- **Principle**: Well-understood building blocks provide useful abstractions
