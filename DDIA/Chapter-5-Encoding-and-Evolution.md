# Chapter 5: Encoding, Evolution, and Compatibility

## Overview

### Core Problem

**Data lives longer than code**:
- Application changes (deploy new versions, fix bugs)
- Database contains data written years ago
- Data format evolution inevitable
- Must support reading old formats

### Questions This Chapter Answers

- How to encode data (binary vs text)?
- How to maintain compatibility when formats change?
- How to evolve schemas without breaking applications?

### Why It Matters

**Forward/backward compatibility**: 
- Old code reads new data format
- New code reads old data format
- Deployed systems mix old/new versions (rolling deploys, multiple services)
- Essential for reliable system evolution

---

## Formats for Encoding Data

### Encoding Terminology

**Encoding (serialization, marshalling)**: Convert in-memory representation to sequence of bytes

**Decoding (deserialization, unmarshalling)**: Reconstruct in-memory object from bytes

**Challenge**: Different programming languages, different in-memory representations
- Python dict vs Java HashMap vs C struct
- Size, endianness, floating-point precision differ

---

## Language-Specific Formats

### Built-in Serialization

**Many languages have native serialization** (Java serialization, Python pickle, Ruby Marshal)

**Advantages**:
- Easy (one function call)
- Preserves complete object detail

**Disadvantages**:
- Deeply tied to language
- Security risk (deserialize arbitrary code execution)
- Versioning problems (language changes break format)
- Other language code can't access data
- Performance/efficiency concerns

### Security Risk Example

**Java serialization**:
```java
byte[] serialized = objectMapper.writeValueAsBytes(userObject);
// Later, in another service...
UserObject user = (UserObject) ois.readObject(); // DANGER!
```

**Problem**: Malicious serialized data can execute arbitrary code during deserialization
- Gadget chains: Combine library classes to trigger commands
- CVE-2015-4852 (Oracle WebLogic)
- CVE-2017-5638 (Apache Struts)

**Recommendation**: Avoid language-specific serialization for untrusted data

---

## JSON, XML, and Binary Variants

### JSON (JavaScript Object Notation)

**Advantages**:
- Human-readable text format
- Universal support (every language has JSON library)
- Good enough for most use cases
- Schema optional (flexible)

**Disadvantages**:
- Verbose (more bytes than necessary)
- No built-in schema (ambiguity in types)
- Doesn't distinguish integer vs float vs decimal
- No binary string support (must base64 encode)

### XML (Extensible Markup Language)

**Similar to JSON**:
- Text-based, human-readable
- Universal support
- Optional schema (XML Schema, DTD)

**Differences**:
- More verbose than JSON (more metadata/tags)
- Powerful schema support
- Namespace support

**Trend**: JSON replaced XML for most use cases (less verbose)

### JSON Example

```json
{
  "userName": "Martin",
  "favoriteNumber": 1337,
  "interests": ["surfing", "philosophy"]
}
```

**Issue**: Reader must assume "favoriteNumber": 1337 is integer (not string "1337")

### Binary Variants

**Motivation**: Text formats verbose, want efficiency of binary

**Examples**: MessagePack, BSON, BJSON, Avro, Protocol Buffers, Thrift

**MessagePack example** (compact binary variant of JSON):
- Encoded: `83 a8 75 73 65 72 4e 61 6d 65 a6 4d 61 72 74 69 6e a8 66 61 76 6f 72 69 74 65 4e 75 6d 62 65 72 cd 05 39 a9 69 6e 74 65 72 65 73 74 73 93 a7 73 75 72 66 69 6e 67 a9 70 68 69 6c 6f 73 6f 70 68 79`
- Slightly smaller than JSON, more complex to read

**Advantage**: 30% smaller than JSON, can be faster to parse (fewer special characters)

**Disadvantage**: Not human-readable, less tooling support than JSON

**Tradeoff**: Space vs simplicity (often JSON wins)

---

## Thrift and Protocol Buffers

### Motivation

**Problem**: Binary formats with schema definition
- More compact than JSON
- Schema defines structure (prevents ambiguity)
- Code generation (compile schema → language-specific classes)

### Thrift (Facebook)

**Schema example**:
```thrift
struct Person {
  1: required string userName,
  2: optional i32 favoriteNumber,
  3: optional list<string> interests
}
```

**Compilation**:
- Run Thrift compiler
- Generates Java/Python/C++ code
- Read/write code handles encoding/decoding
- Developers use generated classes

**Encoding example** (binary):
```
Field 1 (string): 0x0c 0x08 0x4d 0x61 0x72 0x74 0x69 0x6e
Field 2 (i32): 0x04 0x05 0x39
Field 3 (list): 0x0c 0x09 0x0c 0x00 0x02 0x07 0x73 0x75 0x72 0x66 0x69 0x6e 0x67 0x09 0x70 0x68 0x69 0x6c 0x6f 0x73 0x6f 0x70 0x68 0x79
```

**Field tags**:
- Each field has numeric tag (1, 2, 3)
- Encoder writes: tag + value
- Decoder reads: tag → knows field type
- Allows fields to be omitted, renamed, reordered

### Protocol Buffers (Google)

**Schema example**:
```protobuf
syntax = "proto3";

message Person {
  string user_name = 1;
  int32 favorite_number = 2;
  repeated string interests = 3;
}
```

**Similar to Thrift**:
- Numeric field tags
- Schema compilation
- More compact binary format
- Better adoption in industry (Google uses internally)

**proto3 changes**:
- Backward compatibility: Messages allow optional fields
- Fields can be added/removed
- Unknown fields safely ignored by old decoders

### Field Tag Benefits

**Forward compatibility** (new code reads old data):
- Old data missing field 4
- New code expects field 4
- If optional: Use default value
- If required: Error (breaking change)

**Backward compatibility** (old code reads new data):
- New data includes field 4
- Old decoder sees unknown tag 4
- Old code ignores unknown fields
- Works fine

**Schema evolution enabled**: Add/remove/rename fields while maintaining compatibility

### Binary Format Compactness

**Comparison** (encoding "favoriteNumber": 1337):
- JSON: `"favoriteNumber":1337` = 20 bytes (including quotes, colon, spaces)
- Protobuf: `0a 05 39` = 3 bytes (field tag 2 + type/length + value)

**Scaling**:
- Document: Few fields → JSON overhead small
- Document: 100+ fields → Binary format saves significant space
- Millions of requests/day → Space savings → Cost savings

---

## Avro (Apache Avro)

### Different Approach from Thrift/Protobuf

**Key difference**: Avro schema passed separately (not embedded)

**Schema example**:
```json
{
  "type": "record",
  "name": "Person",
  "fields": [
    {"name": "userName", "type": "string"},
    {"name": "favoriteNumber", "type": ["null", "int32"]},
    {"name": "interests", "type": {"type": "array", "items": "string"}}
  ]
}
```

**Encoding** (binary, no field tags):
```
0x0c 0x4d 0x61 0x72 0x74 0x69 0x6e 0x01 0x4a 0x02 0x0e 0x73 0x75 0x72 0x66 0x69 0x6e 0x67 0x12 0x70 0x68 0x69 0x6c 0x6f 0x73 0x6f 0x70 0x68 0x79
```

**Key observation**:
- No field tags in encoding
- Encoder/decoder reference schema separately
- Both must have same schema version
- Saves bytes (no tags) but requires schema agreement

### Schema Evolution Challenge

**Problem**: Encoder uses schema v1, decoder uses schema v2
- How does decoder know which schema was used for encoding?

**Solution**: Store schema with data
- Write: Encode data, write schema reference (hash/ID)
- Database stores: (schema_id, encoded_data)
- Read: Look up schema_id, fetch schema, decode with correct schema

**Kafka example**:
- Messages include schema ID in header
- Kafka broker routes to schema registry
- Fetch full schema, decode correctly

### Schema Compatibility Checking

**Avro tools check**:
- Old writer schema vs new reader schema
- Can new reader schema correctly interpret old data?
- Examples:

**Compatible changes**:
- Add field with default value (old data uses default)
- Remove field without default (old data ignored)
- Rename field by updating reader schema

**Incompatible changes**:
- Add required field with no default (old data can't provide value)
- Remove field with default → old data becomes invalid
- Change field type (int32 → string breaks decoding)

### Avro Flexibility

**Union types**: Field can be multiple types
```json
{"name": "favoriteNumber", "type": ["null", "int32"]}
```
- Means: Either null or int32
- Encodes which type was used
- More flexible than single type

**Avro strengths**:
- Very compact binary format
- Schema evolution built-in
- Dynamic typing support
- Good for streaming (Kafka, Pulsar)

---

## Modes of Dataflow

### Where Data Flows Between Processes

**Fundamental question**: Encoder writes data, decoder reads data

**Three scenarios**:
1. Via database (sender writes, receiver reads later)
2. Via network (sender sends over network, receiver gets)
3. Via files (sender writes file, receiver processes file)

---

## Dataflow via Databases

### Write-Read Gap Problem

**Scenario**:
- Application v1 writes data to database
- Later, application v2 deployed (new code)
- v2 reads data written by v1

**Compatibility required**:
- v2 must decode v1's format
- Schema evolution needed

### In-Database Evolution Example

**Schema**:
```sql
ALTER TABLE users ADD COLUMN age INT DEFAULT NULL;
```

**Migration**:
- Old rows lack age column
- New rows have age column (possibly NULL)
- Application must handle both cases

**Time span**: Could be years between v1 write and v2 read
- Database stores data indefinitely
- Forward/backward compatibility critical

### Data Dump Problem

**Scenario**: Export database to another system
- System A: Exports users table as binary format or JSON
- System B: Imports into different database

**Compatibility**: Both systems must understand same format
- Document format + schema version important
- Evolution path crucial (system B might use older/newer schema)

---

## Dataflow via Network: RPC and REST

### Remote Procedure Call (RPC)

**Goal**: Call function in remote process as if local

**How it works**:
- Client serializes arguments → send over network
- Server deserializes → runs function
- Server serializes result → send back
- Client deserializes result

**Encoding example** (gRPC with Protobuf):
```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  string user_name = 1;
  int32 age = 2;
}
```

**gRPC uses HTTP/2**: Streaming, multiplexing, compression built-in

### Problems with RPC

**Illusion of transparency**: Network call ≠ local function call

**Differences**:
- Network calls can fail (timeout, packet loss)
- Unpredictable latency (same call takes 1ms or 10s)
- Retries: Network call → automatic retry → might run twice
- Local call: Deterministic (always same latency, succeed/fail consistently)

**Implications**:
- Can't pretend network call is local
- Must handle failures explicitly
- Idempotency important (safe to retry)

### REST over HTTP

**Alternative to RPC**: HTTP verbs + JSON

**Example**:
```
GET /users/123 HTTP/1.1
Host: api.example.com

HTTP/1.1 200 OK
Content-Type: application/json
{"user_id": "123", "user_name": "Martin", "age": 42}
```

**Advantages**:
- Simpler than RPC frameworks
- Works across firewalls/proxies
- Caching via HTTP semantics
- Browser-friendly (can test in browser)

**Disadvantages**:
- HTTP overhead (headers, text encoding)
- One request-response per operation
- Can't stream multiple results efficiently

### REST Characteristics

**Stateless**: Server maintains no session state
- Each request contains all needed context
- Simpler scaling (any server can handle any request)
- Easier deployment

**Evolution**: Add optional fields to JSON responses
- Old clients ignore new fields
- New clients provide old format
- gradual migration possible

---

## Dataflow via Network: RPC-like Frameworks

### Examples

**SOAP**: XML-based RPC (complex, verbose, declined)

**Thrift and Protocol Buffers**: Not limited to serialization
- Also define RPC services
- Code generation: Client stub + server skeleton
- Framework handles network communication

### Code Generation Benefits

**Type safety**:
- Compiler checks argument types match
- Compile-time errors (better than runtime)

**Documentation**:
- Service definition = contract between client/server
- Clear inputs/outputs
- IDE autocomplete support

### Service Versioning

**Service interface change**: New parameter, different response

**Approach 1**: Include version in URL/parameter
```
GET /v1/users/123
GET /v2/users/123  # Different schema
```

**Approach 2**: New service endpoint
```
POST /getUserV2  # New version
POST /getUser    # Old version kept for compatibility
```

**Both support**:
- Old clients → old endpoint (works)
- New clients → new endpoint (works)
- Gradual migration path

**Deprecation**: Mark old endpoint as deprecated, give migration time

---

## Dataflow via Message Brokers (Asynchronous)

### Publish-Subscribe Pattern

**Setup**:
- Producer: Sends message to broker (topic/queue)
- Broker: Stores message
- Consumer: Reads from broker (different time, possibly different process)

**Example** (Apache Kafka):
```
Producer A writes to topic "user-events":
{
  "type": "user_created",
  "user_id": 123,
  "username": "alice",
  "email": "alice@example.com"
}

Consumer B reads later from "user-events":
{
  "type": "user_created",
  "user_id": 123,
  "username": "alice",
  "email": "alice@example.com"
}
```

### Advantages Over Direct Network

**Decoupling**:
- Producer doesn't need to know consumers
- Consumers don't need to be online when message sent
- Systems can be deployed independently

**Buffering**: Message broker stores messages
- Producer fast (write to broker immediately)
- Consumer can lag (reads at own pace)
- Handles traffic spikes gracefully

**Reliability**: Broker persists to disk
- Message not lost if consumer crashes
- Consumer can replay from checkpoint

### Encoding and Evolution

**Schema stored**:
- Message includes version info or schema reference
- Stored in message header (Kafka, Pulsar)
- Separate schema registry (common pattern)

**Example** (Kafka + Schema Registry):
```
Kafka message:
{
  "schema_id": 42,     # Reference to schema version 42
  "payload": <binary data>
}

Schema Registry contains schema 42:
{
  "type": "user_created",
  "fields": ["user_id", "username", "email"]
}
```

**Evolution scenario**:
- Old producer (schema v1): Sends user_id, username
- New producer (schema v2): Sends user_id, username, email
- Old consumer (schema v1): Reads v2 data, ignores email field
- New consumer (schema v2): Reads v2 data, gets all fields

### Message Broker vs Direct Network

**Direct RPC**: Tightly coupled, synchronous, fails if service down

**Message broker**: Loosely coupled, asynchronous, producer succeeds even if consumer down
- Better for: Microservices, event-driven systems, cross-team integration

---

## Dataflow Through Services: Event Sourcing and Derived Data

### Event Sourcing (Chapter 3 Review)

**Concept**: Immutable log of events = source of truth

**Example**:
```
Event 1: booking_created (seats: 10, user_id: 101)
Event 2: booking_confirmed (booking_id: 1, payment_confirmed: true)
Event 3: booking_cancelled (booking_id: 1, reason: "user request")
```

**Encoding events**: Often Avro or Protobuf
- Schema per event type
- Versioning per event type
- Easy to evolve independently

**Reading events**: Consumer must handle all versions it encounters
- Must support old event versions
- New event types added → old consumers ignore

### Derived Data Systems

**Materialized views updated**: Event consumers update read-optimized stores

**Example**:
```
Event stream:
event_1 (type=order_placed, item_id=5, qty=2)
event_2 (type=item_shipped, item_id=5)
event_3 (type=item_cancelled, item_id=5)

Materialized view (current orders):
Before event_1: {}
After event_1: {5: pending}
After event_2: {5: shipped}
After event_3: {}
```

**Consumer logic**:
- Stateful: Maintains state from events
- Idempotent: Same event processed multiple times = same result
- Handles event versioning: Accept old and new formats

### Event Versioning Strategy

**Per-event-type schema**:
```
order_placed_v1: {order_id, customer_id, items, timestamp}
order_placed_v2: {order_id, customer_id, items, timestamp, source} # Added source field
```

**Consumer strategy**:
- Rename: order_placed_v2 replaces order_placed_v1
- Old events still published as order_placed_v1
- New events published as order_placed_v2
- Consumer handles both (use default for source if v1)

**Or migration event**:
```
Event: order_placed_v1_to_v2_migration
Contains: Mapping of old orders to new schema
Allows retroactive upgrade
```

---

## Dataflow Through Databases: Adding Fields

### Migration Challenges

**Schema change**: Add column to table

**Problem**: Old rows lack new column, new rows have it

**Solution strategies**:

**1. Default values**:
```sql
ALTER TABLE users ADD COLUMN favorite_color VARCHAR(50) DEFAULT NULL;
```
- Old rows: favorite_color = NULL
- New rows: favorite_color = provided value or NULL
- Application handles NULL (show default color or ask user)

**2. Lazy updates**:
```sql
ALTER TABLE users ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
```
- Old rows: created_at = column add time (not actual creation time, but acceptable)
- New rows: created_at = actual creation time
- Application accepts approximation for old records

**3. Fill-in background job**:
```sql
ALTER TABLE users ADD COLUMN age INT;
-- Later, in background:
UPDATE users SET age = calculate_age(birth_date) WHERE age IS NULL;
```
- Add column with NULL
- Background job calculates values
- No blocking, flexible timing
- Application handles NULL initially

**4. Backfill + deploy cycles**:
- Deployment 1: Add column with NULL default, deploy code to handle NULL
- Background: Backfill values
- Deployment 2: Add NOT NULL constraint (once backfill complete)

### Read/Write Format Separation

**Distinction**:
- Storage format: How data stored on disk
- Application format: In-memory representation

**Example**:
```python
# Storage format (database schema):
CREATE TABLE users (id INT, name VARCHAR, email VARCHAR);

# Application format (Python object):
class User:
  def __init__(self, id, name, email):
    self.id = id
    self.name = name
    self.email = email

# Conversion (ORM handles):
db_record = {id: 1, name: "Alice", email: "alice@example.com"}
user_object = User(**db_record)
```

**Evolution impact**:
- Database schema evolves (add column)
- Application code evolves (handle new field)
- Conversion layer (ORM, mapper) handles mismatch
- Old rows have NULL → Application uses default

---

## Summary: How Machines and Humans Interact

### Core Concepts Reviewed

**Encoding/decoding**: Fundamental to moving data between processes

**Formats**:
- Language-specific (avoid for inter-service use)
- Text: JSON, XML (human-readable)
- Binary: Protobuf, Avro, Thrift (compact, schematized)

### Compatibility Requirements

**Forward compatibility**: New code reads old data
- Optional/default fields
- Unknown fields ignored
- Field tags/schema IDs enable this

**Backward compatibility**: Old code reads new data
- Can skip unknown fields
- Known fields maintain meaning
- Evolution slower (old code constraints)

### Dataflow Scenarios

**Database**: Write (v1) → Read (v2)
- Long time gap, forward compatibility critical

**Network (RPC/REST)**: Request/response immediate
- Both sides usually same team, closely versioned
- Still need multiple versions during rollout

**Message broker**: Producer/consumer decoupled
- Possibly long gap, any time offset
- Schema registry enables version matching
- Similar to database scenario

**Event sourcing**: Immutable event log
- Per-event-type versioning
- Consumer must handle all versions seen
- New event types naturally added

### Evolution Philosophy

**Default approach**: Preserve old data alongside new
- Don't delete old data immediately
- Support reading all versions
- Migrate gradually, test thoroughly
- Avoid irreversible changes if possible
