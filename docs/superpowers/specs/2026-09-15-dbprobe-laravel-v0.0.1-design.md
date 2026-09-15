# DBProbe for Laravel v0.0.1 — Schema Scan Design

**Status:** Approved design contract

**Date:** 2026-09-15

**Repository:** `kefyusuf/dbprobe-laravel`

## 1. Purpose

`dbprobe-laravel` is a local-first, read-only database diagnostics package for Laravel applications.

The v0.0.1 milestone proves one narrow capability:

> Inspect the configured MySQL schema and produce deterministic, evidence-backed structural findings without reading application row data or modifying the database.

The first release is not a general database optimizer, query profiler, monitoring product, or AI agent. It establishes the collection, normalization, finding, evidence, reporting, privacy, and test contracts required for later capabilities.

## 2. Product Positioning

```text
Repository:         kefyusuf/dbprobe-laravel
Composer package:   kefyusuf/dbprobe-laravel
Display name:       DBProbe for Laravel
PHP namespace:      DbProbe\Laravel
Artisan prefix:     dbprobe:
Artifact directory: .dbprobe/
License:            MIT
```

Short positioning:

> Local-first, read-only database diagnostics for Laravel applications.

The package is a Laravel-specific companion to the separate `dbprobe` database-intelligence runtime. v0.0.1 has no runtime dependency on that Go-based project.

## 3. v0.0.1 Scope

### 3.1 Included

- Laravel package auto-discovery.
- `php artisan dbprobe:scan`.
- Optional `--connection=<name>` selection.
- Laravel 12 and Laravel 13 support.
- PHP 8.2+ within the supported Laravel matrix.
- MySQL 8.0 and MySQL 8.4 support.
- Read-only schema metadata collection.
- Base-table, column, and index inventory.
- Canonical normalized schema snapshot.
- Stable SHA-256 structural fingerprint.
- Three deterministic findings:
  - `mysql.duplicate_index`
  - `mysql.redundant_index`
  - `mysql.missing_primary_key`
- Findings-first terminal output.
- Versioned JSON report at `.dbprobe/report.json`.
- Structured caveats when analysis coverage is limited.
- Contract, unit, package feature, and real-MySQL integration tests.

### 3.2 Explicitly excluded

- Migration parsing or execution.
- Docker orchestration by the package.
- Mock or synthetic data generation.
- Eloquent model/source-code analysis.
- Query capture or workload analysis.
- N+1 detection.
- `EXPLAIN` or `EXPLAIN ANALYZE`.
- Workload-derived missing/composite-index recommendations.
- Automatic migration generation.
- Automatic index creation or removal.
- LLM or agent integration.
- Main `dbprobe` binary integration.
- PostgreSQL, MariaDB, SQLite, or other database support.
- Production monitoring or daemon mode.
- SaaS, telemetry, or remote upload.
- CI failure based on findings.
- User-publishable package configuration.

## 4. Design Principles

### 4.1 Deterministic first

The same normalized schema snapshot and ruleset must produce the same findings in the same canonical order. LLMs and agents are not diagnostic authorities and are not part of v0.0.1.

### 4.2 Read-only by construction

Package scan code must not execute DDL or DML. No application rows are sampled. No schema changes are performed. No session mutation is required for normal operation.

### 4.3 Conservative findings

The package prefers a documented limitation over an unsupported assertion. When metadata is ambiguous, a rule skips the ambiguous case and emits a structured caveat where appropriate. A finding is never an automatic remediation instruction.

### 4.4 Evidence and uncertainty are first-class

Each finding carries confidence, exactness, an object reference, and structured evidence.

### 4.5 Minimal public surface

v0.0.1 supports the CLI, its documented option, the JSON report contract, and finding IDs. Internal PHP interfaces are architectural seams, not a third-party plugin API.

### 4.6 Bounded cost

Schema collection uses a fixed number of bulk metadata queries rather than per-table or per-index query loops.

## 5. Compatibility

```text
PHP:       ^8.2
Laravel:   ^12.0 || ^13.0
Database:  MySQL 8.0 / 8.4
```

The package itself remains PHP 8.2 compatible so Laravel 12 is not excluded. Laravel 13 is exercised on PHP 8.3+.

MariaDB and Percona-specific behavior are outside the v0.0.1 compatibility promise.

## 6. User Interface

Primary command:

```bash
php artisan dbprobe:scan
```

Explicit connection:

```bash
php artisan dbprobe:scan --connection=mysql
```

Connection resolution order:

1. `--connection` when provided.
2. Laravel `database.default` otherwise.

The report path is intentionally fixed in v0.0.1:

```text
.dbprobe/report.json
```

Consumer projects should ignore the artifact directory:

```gitignore
/.dbprobe/
```

The package does not modify the consuming project's `.gitignore` automatically.

## 7. Internal Architecture

The selected architecture is intentionally minimal:

```text
ScanCommand
    |
    v
ScanDatabase
    |
    +--> resolve/validate Laravel connection
    |
    v
MySqlSchemaCollector
    |
    v
SchemaSnapshot
    |
    v
RuleEngine
    |
    +--> DuplicateIndexRule
    +--> RedundantIndexRule
    +--> MissingPrimaryKeyRule
    |
    v
ScanResult
    |
    +--> TerminalReportRenderer
    +--> JsonReportWriter
```

### 7.1 `ScanCommand`

- reads CLI options;
- invokes `ScanDatabase`;
- renders the terminal result;
- writes the JSON report;
- returns the documented exit code;
- contains no SQL or rule logic.

### 7.2 `ScanDatabase`

- resolves the selected Laravel connection;
- validates that the target is supported MySQL;
- invokes metadata collection;
- executes deterministic rules;
- returns a `ScanResult`;
- does not format terminal output or serialize JSON.

### 7.3 `MySqlSchemaCollector`

- owns MySQL metadata reads;
- maps raw metadata rows into immutable schema objects;
- does not evaluate findings.

### 7.4 `SchemaSnapshot`

- contains the normalized structural model;
- is immutable;
- produces a canonical structural fingerprint.

### 7.5 `RuleEngine`

- evaluates built-in rules against a snapshot;
- canonicalizes finding order;
- does not access Laravel, PDO, the filesystem, or remote services.

### 7.6 Reporting

Reporters receive `ScanResult` only and never query the database.

## 8. Repository Structure

Target structure after implementation:

```text
dbprobe-laravel/
├── .github/
│   └── workflows/
│       └── tests.yml
├── docs/
│   └── superpowers/
│       ├── specs/
│       └── plans/
├── resources/
│   └── schemas/
│       └── schema-scan-report-v0.1.schema.json
├── src/
│   ├── Commands/
│   │   └── ScanCommand.php
│   ├── Exceptions/
│   ├── Findings/
│   │   ├── Contracts/
│   │   ├── Rules/
│   │   ├── Confidence.php
│   │   ├── Evidence.php
│   │   ├── Exactness.php
│   │   ├── Finding.php
│   │   ├── ObjectReference.php
│   │   ├── RuleEngine.php
│   │   └── Severity.php
│   ├── Reporting/
│   │   ├── JsonReportWriter.php
│   │   └── TerminalReportRenderer.php
│   ├── Scan/
│   │   ├── ScanDatabase.php
│   │   └── ScanResult.php
│   ├── Schema/
│   │   ├── Contracts/
│   │   │   └── SchemaCollector.php
│   │   ├── ColumnDefinition.php
│   │   ├── DatabaseIdentity.php
│   │   ├── IndexDefinition.php
│   │   ├── IndexPart.php
│   │   ├── MySqlSchemaCollector.php
│   │   ├── SchemaSnapshot.php
│   │   └── TableDefinition.php
│   └── DbProbeServiceProvider.php
├── tests/
│   ├── Contract/
│   ├── Feature/
│   ├── Fixtures/
│   │   ├── MySql/
│   │   └── Reports/
│   ├── Integration/
│   ├── Unit/
│   └── TestCase.php
├── .editorconfig
├── .gitattributes
├── .gitignore
├── CHANGELOG.md
├── LICENSE.md
├── README.md
├── composer.json
├── phpstan.neon.dist
└── phpunit.xml.dist
```

No empty future-facing scaffold is committed merely to match this tree. Files are added with working behavior and tests.

## 9. No Package Configuration Surface in v0.0.1

There is no publishable `config/dbprobe.php` in v0.0.1.

The package deliberately keeps configuration out of the first public contract:

```text
connection = --connection when supplied, otherwise Laravel database.default
report path = .dbprobe/report.json
rules = all three built-in rules
finding policy = informational only; successful scan exits 0
```

Configurable report paths, rule enable/disable controls, severity thresholds, suppressions, CI policies, telemetry, AI providers, and concurrency controls require separate future design decisions.

## 10. Metadata Collection Contract

Primary MySQL metadata sources:

```text
information_schema.tables
information_schema.columns
information_schema.statistics
```

Selected server/session metadata may additionally be read when required to interpret schema metadata safely.

### 10.1 Query budget

Normal collection must use at most five read-only metadata `SELECT` statements, independent of table count.

Expected shape:

```text
1 × server/database identity
1 × information_schema.tables
1 × information_schema.columns
1 × information_schema.statistics
0/1 × GIPK visibility metadata
```

Disallowed patterns:

```text
- table-by-table SHOW INDEX
- table-by-table SHOW COLUMNS
- table-by-table SHOW CREATE TABLE
- one metadata query per index
- application-row sampling
```

### 10.2 Safe filtering

Database/schema names used in metadata filters are passed as bound values. User-controlled values are not interpolated into SQL strings.

### 10.3 Metadata collected

Tables:

```text
TABLE_NAME
TABLE_TYPE
ENGINE
```

Columns:

```text
TABLE_NAME
COLUMN_NAME
ORDINAL_POSITION
DATA_TYPE
COLUMN_TYPE
IS_NULLABLE
EXTRA
```

Indexes:

```text
TABLE_NAME
INDEX_NAME
NON_UNIQUE
SEQ_IN_INDEX
COLUMN_NAME
COLLATION
SUB_PART
INDEX_TYPE
IS_VISIBLE
EXPRESSION
```

### 10.4 Intentionally not collected

- row/document values;
- query bindings;
- raw application SQL;
- host;
- port;
- username;
- password;
- DSN/connection URL;
- absolute project path;
- comments;
- default values;
- table row estimates;
- index cardinality;
- data size;
- index size;
- migration/model source code.

## 11. Normalized Schema Model

### 11.1 Database identity

Report identity may include:

```text
connection name
driver
database name
server product
server version
```

It must not include credentials or endpoint details.

### 11.2 Table model

Each base table contains:

```text
schema/table identity
table type
storage engine
ordered columns
ordered indexes
```

Views are inventoried as skipped scope and are not analyzed by v0.0.1 rules.

### 11.3 Column model

Each column records:

```text
name
ordinal position
data type
full column type
nullable
generated/invisible-related structural flags from EXTRA where relevant
```

### 11.4 Index model

Each index records:

```text
name
primary
unique
index type
visibility
ordered key parts
```

Each key part records:

```text
column or expression identity
sequence
prefix length
sort direction
```

Normalization:

```text
COLLATION=A    -> asc
COLLATION=D    -> desc
COLLATION=NULL -> unknown

COLUMN_NAME present -> kind=column
EXPRESSION present  -> kind=expression
```

## 12. Structural Fingerprint

Format:

```text
sha256:<hex>
```

The fingerprint represents the canonical `mysql.schema.v1` structural projection, not complete DDL.

Included:

- exact table name;
- table type;
- storage engine;
- exact column name;
- ordinal position;
- data/full column type;
- nullability;
- relevant structural flags;
- exact index name;
- primary/unique flags;
- index type;
- visibility;
- ordered explicit key parts;
- expression/column identity;
- prefix length;
- sort direction.

Excluded:

- scan timestamp;
- package version;
- report format version;
- connection name;
- database name;
- server version;
- findings;
- approximate cardinality/row counts;
- comments/defaults/sizes.

Canonicalization rules:

- tables sorted by exact identifier;
- columns sorted by ordinal position;
- indexes sorted by exact index name;
- key parts sorted by `SEQ_IN_INDEX`;
- booleans/enums use canonical string forms;
- identifier case is preserved;
- volatile values are removed before hashing.

## 13. Finding Model

The finding model follows the conceptual shape used by the main `dbprobe` project:

```text
ID
Title
Severity
Category
Impact
Confidence
Exactness
Object
Evidence
Summary
Guidance
```

Initial enum values:

```text
Severity:   warning | info
Confidence: high | medium
Exactness:  exact | inferred
```

Machine consumers use stable fields and structured evidence. They must not parse prose fields such as `title`, `summary`, or `guidance`.

## 14. Rule Contract

Internal seam:

```php
interface Rule
{
    public function id(): string;

    /** @return list<Finding> */
    public function evaluate(SchemaSnapshot $snapshot): array;
}
```

Rules:

- receive normalized snapshots only;
- do not query databases;
- do not access Laravel's container;
- do not read files;
- do not invoke remote services;
- do not invoke LLMs;
- do not depend on another rule's output.

The interface is not a supported third-party plugin API in v0.0.1.

## 15. Rule: `mysql.duplicate_index`

Meaning:

> Multiple observable secondary indexes on the same table have the same conservative BTREE signature.

### 15.1 Eligible indexes

Only conservative, column-based, observable secondary BTREE indexes are evaluated.

Comparison signature:

- uniqueness;
- index type;
- visibility;
- ordered key parts;
- key-part kind;
- column identity;
- prefix length;
- sort direction.

Index name is not part of the equivalence signature.

### 15.2 Exclusions

- functional indexes;
- multi-valued indexes;
- FULLTEXT;
- SPATIAL;
- HASH/non-BTREE types;
- definitions with unreliable sort-direction metadata;
- primary-versus-secondary comparisons.

Skipped supported-but-unanalysed structures create caveats where useful.

### 15.3 Grouping

Equivalent indexes are grouped into one finding rather than pairwise findings. Three equivalent indexes produce one finding with evidence for all three.

### 15.4 Output semantics

```text
severity:   warning
confidence: high
exactness:  exact
```

Guidance says to review before removal; it never claims an index is safe to drop.

## 16. Rule: `mysql.redundant_index`

Meaning:

> A non-unique secondary index is a strict leftmost prefix of another visible column-based BTREE secondary index and is therefore a workload-validation candidate.

It does not mean the shorter index is proven unnecessary.

### 16.1 Candidate requirements

Short index:

- secondary;
- non-unique;
- visible;
- BTREE;
- column-based;
- at least one key part.

Long index:

- secondary;
- visible;
- BTREE;
- column-based;
- more key parts than the short index.

Matching prefix key parts must have identical:

- column identity;
- prefix length;
- sort direction.

### 16.2 Exclusions

No finding when:

- the short index is unique;
- the relation is not a leftmost prefix;
- the long index is the primary key;
- either relevant index is invisible;
- prefix lengths differ;
- sort directions differ;
- either relevant key contains a functional/expression part;
- the short signature is already part of an exact-duplicate group.

### 16.3 Output semantics

```text
severity:   info
confidence: medium
exactness:  inferred
```

Guidance requires workload/index-usage validation before removal.

## 17. Rule: `mysql.missing_primary_key`

Meaning:

> No observable `PRIMARY` index was found for a base table.

The rule does not claim that InnoDB has no internal clustered index.

### 17.1 Explicit primary key

When an observable `PRIMARY` index exists, no finding is emitted.

### 17.2 No primary and no suitable fallback

When no explicit primary exists and no full-column `UNIQUE NOT NULL` candidate exists:

```text
severity:   warning
confidence: high
exactness:  exact
```

### 17.3 `UNIQUE NOT NULL` fallback

When no explicit primary exists but an eligible full-column `UNIQUE NOT NULL` index may serve as an InnoDB clustered-key fallback:

```text
severity:   info
confidence: high
exactness:  exact
```

Fallback eligibility requires:

- unique index;
- all key parts column-based;
- all indexed columns NOT NULL;
- no prefix indexing;
- no functional key part.

### 17.4 Generated Invisible Primary Keys

For MySQL versions supporting generated invisible primary keys, the collector attempts to determine whether GIPK metadata is visible through the current session.

If GIPK metadata is hidden or visibility cannot be determined reliably:

- ambiguous InnoDB tables do not receive `mysql.missing_primary_key` findings;
- analysis coverage becomes `limited`;
- a structured caveat is emitted.

False-positive avoidance is preferred over pretending analysis is complete.

## 18. Caveats and Coverage

Successful scans expose:

```text
analysis_coverage: complete | limited
```

Initial caveat codes may include:

```text
mysql.functional_indexes_skipped
mysql.non_btree_indexes_skipped
mysql.unknown_index_order_skipped
mysql.gipk_metadata_hidden
mysql.gipk_visibility_unknown
```

Views are outside v0.0.1 rule scope and represented as an inventory count rather than a caveat.

`0 findings + limited coverage` is not rendered as equivalent to `0 findings + complete coverage`.

## 19. Scan Failure Model

### 19.1 Findings are not failures

```text
exit 0 — scan completed, regardless of finding count
```

### 19.2 Operational failures

```text
exit 1 — runtime, metadata collection, or report-write failure
exit 2 — invalid usage or unsupported connection/driver/server family
```

Fail-closed examples:

- connection cannot be resolved;
- database cannot be reached;
- no database is selected;
- core metadata queries fail;
- unsupported database driver/server family;
- metadata cannot be mapped safely;
- report replacement cannot be completed safely.

A failed scan does not produce a new successful-looking report.

### 19.3 Error redaction

Raw exceptions are not allowed to leak credentials, DSNs, connection URLs, or endpoint secrets into terminal or JSON output.

## 20. JSON Report Contract

Top-level shape:

```json
{
  "format": "dbprobe.laravel.schema-scan",
  "format_version": "0.1.0",
  "tool": {},
  "scan": {},
  "target": {},
  "scope": {},
  "snapshot": {},
  "ruleset": {},
  "inventory": {},
  "summary": {},
  "findings": [],
  "caveats": []
}
```

### 20.1 Scope declaration

Example:

```json
{
  "scope": {
    "base_tables": true,
    "views": false,
    "row_data": false,
    "query_workload": false,
    "source_code": false
  }
}
```

### 20.2 Format versioning

The first report contract is `0.1.0`. During the pre-1.0 period, a breaking report-contract change advances the minor version. `1.0.0` is reserved until the format is exercised by a real external consumer such as the main `dbprobe` runtime or CI integration.

### 20.3 JSON Schema

Repository file:

```text
resources/schemas/schema-scan-report-v0.1.schema.json
```

Schema identifier:

```text
urn:dbprobe:laravel:schema-scan-report:v0.1
```

No runtime JSON-Schema validator is required. The schema is used by tests, fixtures, documentation, and future consumers.

## 21. Report Write Safety

The package does not intentionally expose a partially written JSON report and does not discard a previously valid report before the new report is ready.

Process:

1. create a temporary file in the target directory;
2. serialize and write the complete report;
3. validate that temporary content is valid JSON;
4. replace the target using the safest platform-supported operation;
5. preserve the previous valid report if replacement fails;
6. clean up temporary artifacts.

The design does not claim identical filesystem-level atomic-replace semantics across all operating systems.

## 22. Terminal Report

Example:

```text
DBProbe for Laravel

Connection : mysql
Driver     : mysql
Database   : example_app
MySQL      : 8.4.x
Snapshot   : sha256:...

Inventory
────────────────────────
Tables     42
Columns    388
Indexes    117

Findings
────────────────────────
WARNING  mysql.duplicate_index
         orders

INFO     mysql.redundant_index
         order_items

Summary
────────────────────────
2 findings
1 warning, 1 info

No database changes were made.
Report: .dbprobe/report.json
```

Terminal output distinguishes complete and limited coverage.

## 23. Test Strategy

Four layers are required.

### 23.1 Unit

No Laravel bootstrap and no real MySQL.

Covers:

- snapshot normalization;
- fingerprint determinism;
- finding ordering;
- duplicate rule positive/negative cases;
- redundant rule positive/negative cases;
- missing-primary-key positive/negative cases;
- caveat logic testable from normalized metadata.

### 23.2 Package feature

Uses Orchestra Testbench without real MySQL for most tests.

Covers:

- package auto-discovery/bootstrap;
- service-provider registration;
- `dbprobe:scan` registration;
- `--connection` propagation;
- fallback to Laravel `database.default`;
- exit codes;
- terminal behavior;
- report writer behavior;
- failure redaction.

### 23.3 MySQL integration

Runs against disposable real MySQL 8.0 and 8.4 services.

Fixture schemas cover at least:

```text
healthy_records
duplicate_index_records
redundant_index_records
missing_primary_key_records
unique_fallback_records
view_records
```

Tests verify actual `information_schema` mapping and rule outcomes.

### 23.4 Contract

Covers:

- JSON Schema conformance;
- golden report fixtures;
- required top-level fields;
- supported format version;
- stable finding/evidence ordering;
- absence of credential-like report fields;
- UTC RFC 3339 timestamps;
- `sha256:` fingerprint shape;
- safe report replacement semantics.

## 24. Read-Only Verification

Integration tests observe SQL executed by package scan code after fixture/bootstrap setup.

Allowed package scan statements:

```text
SELECT
```

Forbidden package scan statements:

```text
INSERT
UPDATE
DELETE
CREATE
ALTER
DROP
SET
EXPLAIN ANALYZE
```

Fixture setup may use DDL before the DBProbe scan starts; fixture DDL is test-harness behavior, not package scan behavior.

## 25. CI Matrix

### 25.1 Unit/feature/contract

```text
PHP 8.2 + Laravel 12
PHP 8.3 + Laravel 13
PHP 8.5 + Laravel 13
```

### 25.2 MySQL integration

```text
PHP 8.2 + Laravel 12 + MySQL 8.0
PHP 8.3 + Laravel 13 + MySQL 8.4
```

### 25.3 Windows smoke

```text
Windows + PHP 8.3 + Laravel 13
```

Windows smoke covers package bootstrap, path handling, UTF-8 JSON, temporary-file behavior, and safe report replacement without requiring a real MySQL service.

## 26. Quality Gates

Expected verification commands:

```bash
composer validate --strict
vendor/bin/pint --test
vendor/bin/phpstan analyse
vendor/bin/phpunit --testsuite=Unit
vendor/bin/phpunit --testsuite=Feature
vendor/bin/phpunit --testsuite=Contract
vendor/bin/phpunit --testsuite=Integration
```

PHPStan starts at level 8.

No percentage coverage gate is required for v0.0.1. Behavioral coverage of rules, failure paths, metadata-query budget, read-only posture, contract stability, and determinism is more important than a raw percentage target.

## 27. Composer and Runtime Dependencies

Runtime requirements are limited to the Illuminate components actually required, expected to include:

```text
illuminate/console
illuminate/database
illuminate/support
```

The package does not require the complete `laravel/framework` solely for convenience.

Development dependencies may include Orchestra Testbench, PHPUnit, PHPStan/Larastan as appropriate, and Pint.

Runtime must not add:

- DBProbe-specific HTTP clients;
- LLM SDKs;
- telemetry SDKs;
- queue/cache packages;
- JSON Schema runtime validators;
- remote logging clients.

`composer.lock` is not committed for this reusable library. `composer.json` contains no fixed `version` field; release versions come from Git tags/package metadata.

## 28. Public API Boundary

Supported v0.0.1 external contract:

1. `php artisan dbprobe:scan`;
2. `--connection`;
3. fixed artifact path `.dbprobe/report.json`;
4. JSON report format `dbprobe.laravel.schema-scan`;
5. finding IDs.

Internal PHP DTOs, rule interfaces, collector contracts, container bindings, and orchestration services may evolve during the 0.x series and are not third-party extension APIs.

## 29. README Contract

Initial README documents:

1. project status;
2. what DBProbe for Laravel does;
3. what it does not do;
4. installation;
5. running the schema scan;
6. example output;
7. findings;
8. privacy and read-only behavior;
9. supported versions;
10. current limitations;
11. relationship with the main `dbprobe` project;
12. development/testing;
13. license.

README states clearly:

```text
- No row data is collected.
- No query workload is captured in v0.0.1.
- No schema changes are made.
- Findings are deterministic.
- Redundant-index findings are not drop recommendations.
- LLM usage is neither required nor included in v0.0.1.
```

## 30. Implementation Slices

Implementation proceeds as test-first vertical slices.

### Slice 1 — Package boot

- `composer.json`;
- Testbench;
- service provider;
- command registration;
- first feature test.

### Slice 2 — Schema domain and fingerprint

- database/schema DTOs;
- canonicalization;
- fingerprint tests.

### Slice 3 — MySQL collector

- server-family validation;
- bulk metadata queries;
- row mapping;
- GIPK handling;
- real MySQL integration tests;
- fixed metadata-query-budget test.

### Slice 4 — Deterministic rules

- duplicate index;
- redundant index candidate;
- missing primary key;
- caveats;
- rule fixtures/tests.

### Slice 5 — Orchestration and reporting

- `ScanDatabase`;
- `ScanResult`;
- terminal renderer;
- JSON writer;
- exit codes;
- error redaction;
- JSON contract tests.

### Slice 6 — CI and release documentation

- Linux CI matrix;
- MySQL integration jobs;
- Windows smoke;
- README;
- CHANGELOG;
- license;
- release checklist.

No empty placeholder classes or speculative subsystems are created ahead of a slice.

## 31. Git Strategy

Initial branch model:

```text
main
└── feat/schema-scan-v0.0.1
```

No `develop` branch is needed.

The approved design is the first repository artifact on `main`. Implementation starts only after the design has been reviewed and an implementation plan has been approved.

Implementation commits should be small and behavior-oriented, for example:

```text
test: define package boot contract
feat: add canonical mysql schema snapshot
feat: collect mysql schema metadata
feat: add deterministic schema findings
feat: emit terminal and json reports
ci: verify supported runtime matrix
docs: document schema scan usage
```

The implementation branch is merged only after required quality gates pass. `v0.0.1` is tagged only after all release criteria are satisfied.

## 32. Completion Criteria

v0.0.1 is complete only when all of the following are true:

1. Laravel 12 and 13 package auto-discovery works.
2. `php artisan dbprobe:scan` is registered and usable.
3. Only the promised MySQL 8.0/8.4 target is accepted.
4. Unsupported drivers/server families fail safely.
5. Metadata collection uses no more than five read-only metadata SELECTs in the normal path.
6. Metadata query count does not grow with table count.
7. No application row data is read.
8. The three deterministic rules behave according to this design.
9. False-positive boundaries are protected by unit tests.
10. Ambiguous GIPK visibility does not produce unsupported missing-primary-key findings.
11. The same normalized schema produces the same fingerprint and finding ordering.
12. JSON output conforms to the versioned contract.
13. Failed scans do not replace an existing valid report with a partial/invalid one.
14. Credentials, DSNs, and endpoint secrets do not appear in reports.
15. Findings do not cause non-zero exit status in v0.0.1.
16. Operational failures return the documented non-zero status.
17. Linux compatibility jobs pass.
18. Windows report-writer/package smoke passes.
19. MySQL 8.0 and 8.4 integration jobs pass.
20. README accurately documents the read-only, deterministic, LLM-free scope.

## 33. Locked Decisions

- **DLR-001:** v0.0.1 is a Laravel 12/13, MySQL-only, read-only Schema Scan package.
- **DLR-002:** The package operates without the main `dbprobe` runtime.
- **DLR-003:** Finding IDs use the engine-namespaced `mysql.*` form.
- **DLR-004:** Initial findings are `mysql.duplicate_index`, `mysql.redundant_index`, and `mysql.missing_primary_key` only.
- **DLR-005:** LLMs, agents, query analysis, and auto-remediation are outside v0.0.1.
- **DLR-006:** Core flow is Command -> Scan service -> Collector -> Snapshot -> Rule engine -> Reporters.
- **DLR-007:** `SchemaCollector` and `Rule` are internal seams, not public plugin contracts.
- **DLR-008:** Metadata uses bounded bulk queries rather than per-object inspection.
- **DLR-009:** Rules are pure with respect to database/filesystem/Laravel/remote access.
- **DLR-010:** `SchemaSnapshot` is immutable and canonicalized.
- **DLR-011:** Each snapshot carries a stable SHA-256 structural fingerprint.
- **DLR-012:** Finding fields remain conceptually compatible with the main `dbprobe` project.
- **DLR-013:** Successful scans exit 0 even when findings exist.
- **DLR-014:** Reports are staged completely before replacing the previous valid report.
- **DLR-015:** No facade, adapter registry, event system, persistence store, plugin loader, or AI client is included.
- **DLR-016:** JSON report format starts at `0.1.0`.
- **DLR-017:** Machine consumers use IDs/enums/evidence rather than prose parsing.
- **DLR-018:** No rows, application SQL, credentials, or connection endpoints are collected.
- **DLR-019:** No standalone full-schema manifest artifact exists in v0.0.1.
- **DLR-020:** Volatile and approximate fields are excluded from the fingerprint.
- **DLR-021:** Exact duplicate findings are limited to safely comparable column-based BTREE secondary indexes.
- **DLR-022:** Equivalent indexes are reported as one grouped finding.
- **DLR-023:** Left-prefix redundancy is a workload-validation candidate, never a drop authorization.
- **DLR-024:** Unique, primary-covering, invisible, functional, prefix-mismatched, and order-mismatched cases are conservatively excluded from redundant findings.
- **DLR-025:** Missing-primary-key analysis distinguishes explicit primary keys from eligible `UNIQUE NOT NULL` InnoDB fallback candidates.
- **DLR-026:** Ambiguous/hidden generated-primary-key metadata causes a skip/caveat rather than a false-positive finding.
- **DLR-027:** Missing core metadata prevents a new successful report.
- **DLR-028:** Limited analysis coverage is explicit and machine-readable.
- **DLR-029:** JSON contract is protected by a versioned JSON Schema and golden tests.
- **DLR-030:** Report format reaches `1.0.0` only after real external-consumer validation.
- **DLR-031:** Cross-platform report guarantees focus on no partial-success artifact and preservation of the previous valid report, not identical filesystem atomicity.
- **DLR-032:** Unit, package feature, MySQL integration, and contract tests are mandatory.
- **DLR-033:** Normal metadata collection uses at most five read-only SELECTs independent of table count.
- **DLR-034:** Package scan behavior executes no DDL, DML, or session-mutating statement.
- **DLR-035:** CI verifies representative support boundaries rather than every cross-product.
- **DLR-036:** Windows verifies package bootstrap and report/path semantics.
- **DLR-037:** Composer validation, Pint, PHPStan level 8, PHPUnit suites, and real MySQL integration form the initial quality gate.
- **DLR-038:** Runtime dependencies are limited to necessary Illuminate components.
- **DLR-039:** No committed `composer.lock` and no fixed Composer `version` field.
- **DLR-040:** CLI, `--connection`, fixed report path, JSON format, and finding IDs are the supported external surface; internal PHP types are not frozen APIs.
- **DLR-041:** Implementation proceeds by working test-first vertical slices without speculative scaffolding.
- **DLR-042:** Use `main` plus short-lived feature branches; no `develop` branch.
- **DLR-043:** The approved design document precedes implementation.
- **DLR-044:** `v0.0.1` is tagged only when completion criteria and CI gates pass.
- **DLR-045:** v0.0.1 has no publishable package config; connection selection is CLI-or-Laravel-default and report location is fixed.

## 34. Deferred Evolution

The architecture leaves room for separately designed later milestones such as:

```text
- query workload capture through Laravel runtime/test execution;
- Eloquent source mapping;
- N+1 and repeated-query findings;
- plan inspection;
- query-scoped synthetic datasets;
- before/after index verification;
- CI regression policies;
- schema/history snapshots;
- optional local or remote LLM explanation;
- remediation agents;
- interoperability with the main dbprobe runtime.
```

None of these deferred ideas may expand v0.0.1 without a new design decision.
