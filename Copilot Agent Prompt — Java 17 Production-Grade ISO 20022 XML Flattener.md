# Copilot Agent Instructions — Production-Grade ISO 20022 XML Flattener

You are implementing a production-grade, generic, configuration-driven XML flattening library for ISO 20022 payment messages.

The library must be designed for banking/financial workloads and must be suitable for high-throughput production environments.

---

# 1. ABSOLUTE JAVA VERSION REQUIREMENT

## Java 17 ONLY

The entire project MUST be fully compatible with **Java 17**.

Java 17 is the minimum AND maximum language/API baseline for this project.

### Mandatory requirements

- Compile using JDK 17.
- Target Java 17 bytecode.
- Maven compiler configuration MUST use:

```xml
<release>17</release>
```

- All modules must compile with Java 17.
- All tests must execute successfully on Java 17.
- CI must build and test using Java 17.
- Production deployment must support Java 17.
- Do NOT use Java 18, 19, 20, 21, 22, 23, 24, 25+ language features or APIs.
- Do NOT introduce Java 21 APIs even if they appear convenient.
- Do NOT use preview features.
- Do NOT use incubator APIs.
- Do NOT use virtual threads.
- Do NOT use `StructuredTaskScope`.
- Do NOT use Java 21 sequenced collections.
- Do NOT use pattern matching features that require Java versions greater than 17.
- Do NOT use record patterns.
- Do NOT use string templates.
- Do NOT use unnamed classes.
- Do NOT use Java 21 collection/API additions.
- Do NOT configure the project to compile against Java 21.

The project must remain executable on a standard Java 17 runtime.

### Java 17 is a hard architectural constraint

Do not create a separate Java 21 module.

Do not create Java 21-specific optimizations.

Do not design an abstraction whose implementation requires Java 21.

Performance must be achieved using Java 17-compatible techniques.

---

# 2. PROJECT OBJECTIVE

Build a generic XML flattening engine that converts ISO 20022 XML documents into flat records according to external YAML configuration.

Example:

```text
ISO 20022 XML
      |
      v
Secure Streaming XML Parser
      |
      v
Compiled Mapping
      |
      v
Path Matcher
      |
      v
Context Manager
      |
      v
Field Extraction
      |
      v
Type Conversion
      |
      v
Transformations
      |
      v
FlatRecord
      |
      v
RecordConsumer
```

The engine MUST NOT contain hardcoded handlers such as:

```java
Pain001Handler
Pacs008Handler
Camt053Handler
```

The same engine must support different ISO 20022 message types through configuration.

Examples:

- pain.001
- pain.002
- pacs.008
- pacs.002
- camt.053
- camt.054
- future ISO 20022 message types

---

# 3. PERFORMANCE TARGET

The target workload is:

- XML size: approximately 500 KB average
- XML size: potentially 1 MB+
- XML complexity: approximately 1,000–2,000 lines average
- Output: approximately 20 columns
- Target throughput: **1,000 XML documents/minute**
- Equivalent throughput:

```text
16.7 documents/second
```

At 500 KB/document:

```text
~8.3 MB/sec input throughput
```

The implementation MUST be designed to scale horizontally.

Do NOT claim that the target is achieved until it is demonstrated through benchmarks.

---

# 4. MEMORY REQUIREMENTS

Memory efficiency is critical.

DO NOT:

- load the complete XML into a String
- convert InputStream into byte[]
- use DOM
- build a complete XML object tree
- store every record in a List
- store every extracted field in a large Map
- duplicate XML content unnecessarily
- accumulate complete documents in memory

Prefer:

```text
InputStream
   ↓
StAX
   ↓
current XML event
   ↓
current record state
   ↓
FlatRecord
   ↓
RecordConsumer
```

Only the current document state and current record state should be retained.

Records must be emitted immediately.

---

# 5. XML PARSING

Use a streaming XML parser.

Preferred technology:

```text
StAX
+
Woodstox
```

Do NOT use:

```text
DOM
SAX + large custom buffering
XPath.evaluate() for every field
JAXB object graph for the flattening hot path
```

The parser must process XML incrementally.

Example API:

```java
public interface FlattenEngine {

    void flatten(
        InputStream input,
        RecordConsumer consumer
    );
}
```

---

# 6. XML SECURITY

The XML parser MUST be securely configured.

Prevent:

- XXE
- external entities
- external DTDs
- entity expansion attacks
- Billion Laughs
- external resource access

Do not allow untrusted XML to resolve external resources.

Security tests MUST include malicious XML examples.

---

# 7. CONFIGURATION-DRIVEN DESIGN

Mappings must be provided externally.

Example YAML:

```yaml
schemaVersion: "1.0"

name: pain001_transaction

message:
  type: pain.001
  version: 001.001.13

namespace:
  prefix: iso
  uri: urn:iso:std:iso:20022:tech:xsd:pain.001.001.13

record:
  path: /Document/CstmrCdtTrfInitn/PmtInf/CdtTrfTxInf

fields:

  - name: messageId
    scope: DOCUMENT
    path: GrpHdr/MsgId
    type: STRING

  - name: paymentInformationId
    scope: PAYMENT_INFO
    path: PmtInfId
    type: STRING

  - name: instructionId
    scope: TRANSACTION
    path: PmtId/InstrId
    type: STRING

  - name: endToEndId
    scope: TRANSACTION
    path: PmtId/EndToEndId
    type: STRING

  - name: creditorName
    scope: TRANSACTION
    path: Cdtr/Nm
    type: STRING

  - name: amount
    scope: TRANSACTION
    path: Amt/InstdAmt
    type: DECIMAL

  - name: currency
    scope: TRANSACTION
    path: Amt/InstdAmt/@Ccy
    type: STRING
```

YAML is an application-level mapping language.

Do not treat YAML itself as an ISO 20022 standard.

---

# 8. PATH LANGUAGE

Implement a controlled XML path language.

Example:

```text
GrpHdr/MsgId
PmtInfId
PmtId/EndToEndId
Cdtr/Nm
Amt/InstdAmt
Amt/InstdAmt/@Ccy
```

Do NOT implement full XPath in version 1.

Do NOT execute XPath expressions for every field.

Compile paths during configuration loading.

For example:

```text
"Cdtr/Nm"
```

should become an internal representation similar to:

```text
[
    "Cdtr",
    "Nm"
]
```

The compiled representation should be reused for every document.

---

# 9. NAMESPACE HANDLING

Namespace handling must be based on:

```text
namespace URI
+
local element name
```

Do NOT rely only on XML prefixes.

These should be treated as equivalent when they resolve to the same namespace:

```xml
<iso:Document>
```

and

```xml
<Document xmlns="urn:iso:std:iso:20022:tech:xsd:...">
```

The engine must correctly handle namespace-aware StAX events.

---

# 10. RECORD MODEL

The configured record path determines when a record is produced.

Example:

```text
/Document/CstmrCdtTrfInitn/PmtInf/CdtTrfTxInf
```

Every occurrence of:

```text
CdtTrfTxInf
```

produces one:

```text
FlatRecord
```

Do not store all records.

Use:

```java
public final class FlatRecord {

    private final Object[] values;

    // appropriate accessors
}
```

Field order must correspond to the compiled column definition.

Avoid a `Map<String, Object>` in the hot path.

A Map-based representation may exist as an optional convenience API.

---

# 11. RECORD CONSUMER

Use streaming output.

```java
@FunctionalInterface
public interface RecordConsumer {

    void accept(FlatRecord record);
}
```

Example:

```java
engine.flatten(inputStream, record -> {
    databaseWriter.write(record);
});
```

The engine must not decide where records are stored.

The consumer could write to:

- database
- Kafka
- file
- another service
- batch processor
- memory
- analytics pipeline

This separation is mandatory.

---

# 12. HIERARCHICAL CONTEXT

ISO 20022 documents contain hierarchical information.

For example:

```text
Document
 └── GrpHdr
      └── PmtInf
           └── CdtTrfTxInf
```

A transaction record may need values from parent levels.

Example:

```yaml
- name: messageId
  scope: DOCUMENT
  path: GrpHdr/MsgId

- name: paymentInformationId
  scope: PAYMENT_INFO
  path: PmtInfId

- name: endToEndId
  scope: TRANSACTION
  path: PmtId/EndToEndId
```

When processing a transaction:

```text
messageId
paymentInformationId
endToEndId
```

must be available.

Implement a lightweight context model.

Do not retain unnecessary XML objects.

---

# 13. ATTRIBUTES

Attributes must be supported.

Example:

```xml
<InstdAmt Ccy="EUR">100.50</InstdAmt>
```

Configuration:

```yaml
- name: amount
  path: Amt/InstdAmt
  type: DECIMAL

- name: currency
  path: Amt/InstdAmt/@Ccy
  type: STRING
```

The path compiler must distinguish:

```text
element
```

from:

```text
attribute
```

---

# 14. FALLBACK PATHS

Support fallback extraction.

Example:

```yaml
- name: accountId
  firstAvailable:
    - DbtrAcct/Id/IBAN
    - DbtrAcct/Id/Othr/Id
```

The engine should use the first available value.

This functionality must be generic.

---

# 15. DEFAULT VALUES

Support defaults.

Example:

```yaml
- name: country
  path: Cdtr/PstlAdr/Ctry
  type: STRING
  default: UNKNOWN
```

Do not silently convert errors into defaults.

A default should apply only when the configured value is absent, according to documented semantics.

---

# 16. TYPE SYSTEM

At minimum support:

```text
STRING
INTEGER
LONG
DECIMAL
BOOLEAN
LOCAL_DATE
OFFSET_DATE_TIME
```

For financial values:

```java
BigDecimal
```

MUST be used.

Never use:

```java
double
float
```

for monetary values.

Conversions must be deterministic and testable.

---

# 17. TRANSFORMATIONS

Support transformations such as:

```text
TRIM
EMPTY_TO_NULL
UPPERCASE
LOWERCASE
DATE_FORMAT
DECIMAL_SCALE
DEFAULT_VALUE
CONCAT
```

Use an extensible abstraction:

```java
public interface Transformer {

    Object transform(
        Object value,
        TransformationContext context
    );
}
```

Transformers should be registered through a registry.

Example:

```java
transformerRegistry.register(
    "TRIM",
    new TrimTransformer()
);
```

Avoid large `if/else` or `switch` blocks containing all transformation logic.

---

# 18. SOLID PRINCIPLES

Strictly follow SOLID.

### Single Responsibility

Separate:

- XML parsing
- path matching
- configuration
- type conversion
- transformation
- context management
- record creation
- error handling

### Open/Closed

New transformations should be addable without modifying core engine logic.

### Liskov Substitution

Implementations must respect their interfaces.

### Interface Segregation

Avoid giant interfaces.

### Dependency Inversion

Core modules must depend on abstractions rather than infrastructure.

---

# 19. MODULE STRUCTURE

Use Maven multi-module architecture.

Recommended:

```text
iso-flattener/
│
├── pom.xml
│
├── iso-flattener-core/
│
├── iso-flattener-xml/
│
├── iso-flattener-config/
│
├── iso-flattener-transform/
│
├── iso-flattener-validation/
│
├── iso-flattener-spring/
│
├── iso-flattener-benchmarks/
│
└── README.md
```

Every module MUST remain Java 17 compatible.

---

# 20. DEPENDENCY DIRECTION

Prefer:

```text
core
 ↑
xml
 ↑
config
 ↑
transform
 ↑
validation
 ↑
spring
```

But avoid unnecessary coupling.

The core module MUST NOT depend on Spring.

The core module MUST NOT depend on:

- Spring Boot
- Spring Framework
- Micrometer
- Kafka
- database libraries
- HTTP frameworks
- application-specific code

The Spring module is an adapter only.

---

# 21. JAVA 17 COMPATIBILITY VERIFICATION

The build MUST verify Java 17 compatibility.

Maven compiler:

```xml
<release>17</release>
```

The project must successfully run:

```bash
mvn clean test
```

using JDK 17.

The agent MUST NOT change the project to:

```xml
<release>21</release>
```

or any newer version.

If a dependency requires Java 21+, do NOT blindly upgrade it.

Find a Java 17-compatible version instead.

Before adding any dependency, verify its Java runtime requirement.

---

# 22. THREAD SAFETY

Compiled mappings should be immutable.

The engine should be reusable across threads.

Example:

```text
CompiledMapping
      |
      +---- Thread 1
      |
      +---- Thread 2
      |
      +---- Thread 3
```

Do not maintain mutable processing state inside singleton engine objects.

Per-document state must be local to the processing invocation.

Avoid:

```java
static mutable fields
```

Avoid shared:

```java
StringBuilder
HashMap
parser state
record state
context state
```

unless properly isolated.

---

# 23. MULTI-POD DEPLOYMENT

The library must be stateless.

Do NOT introduce:

- pod identity
- distributed locks
- in-memory distributed coordination
- shared mutable static state
- pod-local deduplication assumptions

Multiple application pods must be able to process documents independently.

Exactly-once processing and idempotency belong to the host application.

Potential identifiers:

```text
messageId
transactionId
endToEndId
```

The library must not implement distributed idempotency itself.

---

# 24. CONCURRENCY

Use bounded concurrency.

Do NOT create:

```java
Executors.newFixedThreadPool(10000)
```

or unbounded queues.

The library should not create an executor unless explicitly required by its API.

Prefer synchronous streaming processing and let the hosting application control concurrency.

If an executor abstraction becomes necessary, it MUST remain fully Java 17 compatible.

---

# 25. ERROR HANDLING

Create meaningful exceptions:

```text
ConfigurationException
XmlParsingException
PathResolutionException
ConversionException
TransformationException
FlatteningException
```

Do not silently swallow exceptions.

Support configurable policies:

```text
FAIL_FAST
SKIP_RECORD
COLLECT_ERRORS
```

Error behavior must be deterministic and documented.

---

# 26. LOGGING

Never log complete payment XML.

Never log sensitive values such as:

- IBAN
- account number
- customer information
- payment amount
- credentials
- tokens
- complete XML

Use sanitized metadata.

Example:

```text
correlationId
messageType
processingTime
recordCount
errorType
```

Do not introduce high-cardinality metric labels such as:

```text
messageId
transactionId
accountNumber
```

---

# 27. OBSERVABILITY

The Spring adapter should support Micrometer metrics.

Recommended metrics:

```text
documents_processed_total
documents_failed_total
records_generated_total
records_skipped_total
processing_time
xml_parse_time
flatten_time
conversion_errors
configuration_errors
```

Keep metric labels low-cardinality.

---

# 28. CONFIGURATION COMPILATION

Configuration must NOT be interpreted repeatedly for every XML element.

The lifecycle should be:

```text
YAML
 |
 v
Parse
 |
 v
Validate
 |
 v
Resolve namespaces
 |
 v
Compile paths
 |
 v
Compile transformations
 |
 v
CompiledMapping
 |
 v
Reusable engine
```

Compile once.

Reuse many times.

---

# 29. CONFIGURATION VALIDATION

Validate configuration at startup.

Detect:

- missing mapping name
- invalid message type
- invalid namespace
- duplicate field names
- invalid paths
- invalid datatype
- invalid transformation
- invalid record path
- malformed fallback configuration
- unsupported configuration version

Fail early.

Do not discover configuration errors after processing thousands of documents.

---

# 30. TEST-DRIVEN DEVELOPMENT

Use strict TDD.

For every feature:

```text
RED
 ↓
Write failing test
 ↓
GREEN
 ↓
Minimal implementation
 ↓
REFACTOR
 ↓
Run complete test suite
```

Do not implement large features without tests.

---

# 31. UNIT TESTS

Create tests for:

### Configuration

- valid YAML
- invalid YAML
- duplicate fields
- invalid datatype
- invalid transformation
- invalid path

### XML paths

- simple paths
- nested paths
- record paths
- attributes
- missing values
- fallback paths

### Namespaces

- default namespace
- prefixed namespace
- multiple namespaces
- namespace mismatch

### Types

- String
- Integer
- Long
- BigDecimal
- Boolean
- LocalDate
- OffsetDateTime

### Transformations

- trim
- empty-to-null
- uppercase
- lowercase
- date formatting
- decimal scaling
- default
- concat

### Context

- document scope
- parent scope
- transaction scope
- context reset between records

### Security

- XXE
- external DTD
- entity expansion
- malicious XML

---

# 32. INTEGRATION TESTS

Include realistic XML fixtures for:

```text
pain.001
pacs.008
pacs.002
pain.002
camt.053
camt.054
```

Also include a generic custom XML fixture.

Tests must verify that the same engine works with different mappings without changing Java code.

---

# 33. CONCURRENCY TESTS

Verify that the same compiled mapping and engine can safely process documents concurrently.

Example:

```text
Thread 1 → document A
Thread 2 → document B
Thread 3 → document C
Thread 4 → document D
```

Results must never leak between documents.

---

# 34. PERFORMANCE TESTING

Create JMH benchmarks.

Benchmark at least:

```text
500 KB XML
1 MB XML
2 MB XML
5 MB XML
```

Measure:

```text
documents/sec
records/sec
MB/sec
p50 latency
p95 latency
p99 latency
heap usage
allocation rate
GC activity
CPU usage
```

The benchmark must represent realistic XML structures rather than trivial XML.

Do not optimize based solely on theoretical assumptions.

---

# 35. PERFORMANCE PRINCIPLES

Prefer:

```text
streaming
immutable configuration
precompiled paths
primitive/simple internal structures where appropriate
minimal object allocation
reusable compiled mapping
sequential parsing
immediate record emission
```

Avoid:

```text
DOM
reflection in hot paths
repeated YAML parsing
repeated path parsing
XPath evaluation per field
large intermediate object graphs
unbounded queues
unnecessary Maps
unnecessary String creation
```

Do not sacrifice correctness or security merely for benchmark numbers.

---

# 36. MEMORY BENCHMARK

The benchmark should prove that memory usage does not scale linearly with the number of records in the XML.

For example:

```text
100 transactions
1000 transactions
5000 transactions
```

The engine should emit records progressively rather than storing all records.

---

# 37. SPRING BOOT INTEGRATION

Spring support must remain optional.

Core usage:

```java
FlattenEngine engine = ...;
engine.flatten(inputStream, consumer);
```

Spring usage may provide:

- configuration loading
- dependency injection
- lifecycle management
- metrics
- bean creation

But the core library must work without Spring.

---

# 38. OPTIONAL ISO 20022 LIBRARIES

Do not make Prowide or another ISO library mandatory unless there is a clear requirement.

The generic flattening engine must remain independent of generated ISO Java models.

If an ISO 20022 library is used, isolate it behind an adapter.

Do not create compile-time coupling between the core engine and a vendor-specific ISO library without justification.

---

# 39. CODE QUALITY

Code must be:

- readable
- maintainable
- testable
- documented where necessary
- production-grade

Avoid:

```text
God classes
God methods
large switch statements
large if/else chains
static mutable state
magic strings
magic numbers
duplicate logic
unnecessary abstractions
```

Prefer composition over inheritance where appropriate.

---

# 40. DOCUMENTATION

Create documentation covering:

```text
Architecture
Configuration format
YAML examples
Supported datatypes
Supported transformations
Path syntax
Namespace handling
Error handling
Thread safety
Performance
Security
Spring integration
Java 17 requirements
Adding new transformations
Adding new message mappings
```

Include architecture diagrams in README/documentation using Mermaid where useful.

---

# 41. JAVA 17 ENFORCEMENT CHECKLIST

Before completing any task, verify:

```text
[ ] JDK 17 builds successfully
[ ] Maven release = 17
[ ] Java 17 bytecode generated
[ ] No Java 18+ APIs
[ ] No Java 19+ APIs
[ ] No Java 20+ APIs
[ ] No Java 21+ APIs
[ ] No preview features
[ ] No incubator APIs
[ ] No virtual threads
[ ] No Java 21-specific collections/API
[ ] No Java 21-specific language syntax
[ ] All tests pass on Java 17
```

If uncertain whether an API exists in Java 17, assume it is NOT allowed until verified.

---

# 42. AGENT BEHAVIOR

Before modifying code:

1. Inspect the repository.
2. Understand the existing architecture.
3. Check the current Java version.
4. Check the Maven structure.
5. Check existing tests.
6. Check existing dependencies.
7. Identify whether this is a new project or existing project.
8. Explain assumptions before making major architectural changes.

Do not rewrite existing code unnecessarily.

Do not introduce unnecessary dependencies.

Do not create speculative abstractions.

Prefer the simplest architecture that satisfies the requirements.

---

# 43. IMPLEMENTATION PHASES

Implement incrementally.

## Phase 1 — Architecture

Create:

- Maven parent
- modules
- package structure
- core interfaces
- initial domain models
- test structure
- Java 17 build configuration

Do NOT implement the XML parser yet.

Stop and report.

---

## Phase 2 — Configuration

Implement:

- YAML parsing
- configuration model
- validation
- immutable compiled configuration

Write tests first.

---

## Phase 3 — XML Streaming

Implement:

- secure StAX configuration
- Woodstox integration
- streaming event processing
- namespace-aware element matching

Write security tests.

---

## Phase 4 — Path Engine

Implement:

- path tokenizer
- path compiler
- compiled path matcher
- element extraction
- attribute extraction

Do not use XPath evaluation in the hot path.

---

## Phase 5 — Context Engine

Implement:

- document context
- parent context
- transaction context
- context inheritance
- context reset

---

## Phase 6 — Record Generation

Implement:

```text
FlatRecord
RecordConsumer
```

Records must be emitted immediately.

---

## Phase 7 — Conversion and Transformations

Implement:

- datatype conversion
- BigDecimal handling
- transformations
- fallback values
- defaults

---

## Phase 8 — Validation and Error Policies

Implement:

```text
FAIL_FAST
SKIP_RECORD
COLLECT_ERRORS
```

and exception hierarchy.

---

## Phase 9 — ISO 20022 Fixtures

Add:

```text
pain.001
pain.002
pacs.008
pacs.002
camt.053
camt.054
```

using external configuration.

No message-specific Java handlers.

---

## Phase 10 — Concurrency and Production Hardening

Implement and test:

- thread safety
- concurrent processing
- immutable mappings
- stateless operation
- bounded resource usage
- security
- logging
- metrics

---

## Phase 11 — Performance

Create JMH benchmarks.

Benchmark:

```text
500 KB
1 MB
2 MB
5 MB
```

Measure:

```text
throughput
latency
allocation
GC
heap
CPU
```

Optimize only based on benchmark evidence.

---

## Phase 12 — Documentation

Complete:

- README
- architecture documentation
- YAML reference
- API documentation
- security documentation
- performance documentation
- Java 17 compatibility documentation

---

# 44. MOST IMPORTANT ARCHITECTURAL RULES

The following rules are NON-NEGOTIABLE:

### Rule 1

Java 17 only.

### Rule 2

No Java 21 compatibility layer.

### Rule 3

No Java 21 APIs.

### Rule 4

No DOM.

### Rule 5

Streaming XML processing.

### Rule 6

Configuration-driven mappings.

### Rule 7

No message-specific Java handlers.

### Rule 8

Compile mappings once.

### Rule 9

Do not store all records.

### Rule 10

Use BigDecimal for financial values.

### Rule 11

Namespace-aware XML processing.

### Rule 12

Secure XML parser.

### Rule 13

Immutable compiled configuration.

### Rule 14

Thread-safe engine.

### Rule 15

Stateless multi-pod operation.

### Rule 16

Strict TDD.

### Rule 17

Benchmark before claiming performance.

### Rule 18

Do not trade correctness/security for performance.

---

# 45. FIRST TASK

Start with **Phase 1 only**.

First inspect the existing repository.

Then report:

1. Current repository structure
2. Existing Java version
3. Existing Maven structure
4. Existing dependencies
5. Existing tests
6. Proposed module structure
7. Proposed package structure
8. Dependency direction
9. Core interfaces
10. Configuration model
11. FlatRecord design
12. Java 17 compatibility strategy
13. TDD strategy
14. Performance strategy
15. Security strategy
16. Any assumptions

Before creating significant code, explain the proposed architecture.

Then create only the Phase 1 skeleton.

Run:

```bash
mvn clean test
```

using Java 17.

Verify that:

```text
Java 17 compilation succeeds
Java 17 tests succeed
No Java 21 APIs are present
No Java 21-specific language features are present
```

Do NOT implement the XML parser yet.

Do NOT implement transformations yet.

Do NOT implement the full path engine yet.

Do NOT implement ISO-specific handlers.

Stop after Phase 1.

Return:

```text
PHASE 1 COMPLETE

Files created:
...

Dependencies:
...

Tests:
...

Build:
...

Java version:
...

Java 17 compatibility:
PASS/FAIL

Architecture decisions:
...

Remaining Phase 2 work:
...
```

Wait for further instructions before starting Phase 2.