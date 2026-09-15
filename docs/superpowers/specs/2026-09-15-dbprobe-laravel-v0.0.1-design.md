# DBProbe for Laravel v0.0.1 — Schema Scan Design

**Status:** Approved design contract

**Date:** 2026-09-15

**Repository:** `kefyusuf/dbprobe-laravel`

## 1. Purpose

`dbprobe-laravel` is a local-first, read-only database diagnostics package for Laravel applications.

The v0.0.1 milestone deliberately proves one narrow capability:

> Can the package inspect the configured MySQL schema and produce deterministic, evidence-backed structural findings without reading application row data or modifying the database?

The first release is not a general database optimizer, query profiler, monitoring product, or AI agent. It establishes the collection, normalization, finding, evidence, reporting, privacy, and test contracts required for later capabilities.

## 2. Product Positioning

Package identity:

```text
Repository:        kefyusuf/dbprobe-laravel
Composer package:  kefyusuf/dbprobe-laravel
Display name:      DBProbe for Laravel
PHP namespace:     DbProbe\Laravel
Artisan prefix:    dbprobe:
Artifact directory:.dbprobe/
License:           MIT
```

Short positioning:

> Local-first, read-only database diagnostics for Laravel applications.

The package is a Laravel-specific companion to the separate `dbprobe` database-intelligence runtime. v0.0.1 has no runtime dependency on the Go-based `dbprobe` project.

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
- Versioned JSON report at `.dbprobe/report.json` by default.
- Structured caveats when analysis coverage is limited.
- Contract, unit, package feature, and real-MySQL integration tests.

### 3.2 Explicitly excluded

- Migration parsing or execution.
- Docker orchestration by the package.
- Mock or synthetic data generation.
- Eloquent model/source-code analysis.
- Query capture or query workload analysis.
- N+1 detection.
- `EXPLAIN` or `EXPLAIN ANALYZE`.
- Missing/composite index recommendations derived from workloads.
- Automatic migration generation.
- Automatic index creation or removal.
- LLM or agent integration.
- Main `dbprobe` binary integration.
- PostgreSQL, MariaDB, SQLite, or other database support.
- Production monitoring or daemon mode.
- SaaS, telemetry, or remote upload.
- CI failure based on findings.

## 4. Design Principles

### 4.1 Deterministic first

The same normalized schema snapshot and ruleset must produce the same findings in the same canonical order.

LLMs and agents are not diagnostic authorities and are not part of v0.0.1.

### 4.2 Read-only by construction

Package scan code must not execute DDL or DML. It reads metadata only.

No application rows are sampled. No schema changes are performed. No session mutation is required for normal operation.

### 4.3 Conservative findings

The package prefers a documented limitation over an unsupported assertion. When metadata is ambiguous, a rule skips the ambiguous case and emits a structured caveat where appropriate.

A finding is not an automatic remediation instruction.

### 4.4 Evidence and uncertainty are first-class

Each finding carries confidence, exactness, an object reference, and structured evidence.

### 4.5 Minimal public surface

v0.0.1 supports the CLI, its documented option, configuration keys, JSON report contract, and finding IDs. Internal PHP interfaces are architectural seams, not a third-party plugin API.

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
2. `dbprobe.connection` when configured.
3. Laravel `database.default`.

Default report path:

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

### 7.1 Dependency responsibilities

`ScanCommand`:

- reads CLI options;
- invokes `ScanDatabase`;
- renders the terminal result;
- writes the JSON report;
- returns the documented exit code.

It contains no SQL or rule logic.

`ScanDatabase`:

- resolves the selected Laravel database connection;
- validates that the target is supported MySQL;
- invokes metadata collection;
- executes deterministic rules;
- returns a `ScanResult`.

It does not format terminal output or serialize JSON.

`MySqlSchemaCollector`:

- owns MySQL metadata reads;
- maps raw metadata rows into immutable schema objects;
- does not evaluate findings.

`SchemaSnapshot`:

- contains the normalized structural model;
- is immutable;
- produces a canonical structural fingerprint.

`RuleEngine`:

- evaluates all built-in rules against a snapshot;
- canonicalizes finding order;
- does not access Laravel, PDO, the filesystem, or remote services.

`Reporting`:

- receives `ScanResult` only;
- does not query the database.

## 8. Repository Structure

Target structure after implementation:

```text
dbprobe-laravel/
├── .github/
│   └── workflows/
│       └── tests.yml
├── config/
│   └── dbprobe.php
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

No empty future-facing scaffold should be committed merely to match this tree. Files are added with working behavior and tests.

## 9. Configuration

Initial configuration is deliberately small:

```php
return [
    'connection' => null,

    'report' => [
        'path' => base_path('.dbprobe/report.json'),
    ],
];
```

v0.0.1 does not provide configurable rule selection, severity thresholds, suppression lists, fail-on-finding behavior, telemetry, LLM configuration, or concurrency controls.

## 10. Metadata Collection Contract

The collector uses MySQL metadata sources such as:

```text
information_schema.tables
information_schema.columns
information_schema.statistics
```

It may additionally read selected server/session variables required to interpret metadata safely.

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

Disallowed implementation patterns:

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

Stored report identity may include:

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

Normalization examples:

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

- tables are sorted by exact identifier;
- columns are sorted by ordinal position;
- indexes are sorted by exact index name;
- key parts are sorted by `SEQ_IN_INDEX`;
- booleans/enums have canonical string forms;
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

Severity:

```text
warning
info
```

Confidence:

```text
high
medium
```

Exactness:

```text
exact
inferred
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

The comparison signature includes:

- uniqueness;
- index type;
- visibility;
- ordered key parts;
- key-part kind;
- column identity;
- expression identity where represented, although expression indexes are excluded in v0.0.1;
- prefix length;
- sort direction.

Index name is not part of the equivalence signature.

### 15.2 Excluded from exact duplicate analysis

- functional indexes;
- multi-valued indexes;
- FULLTEXT;
- SPATIAL;
- HASH/non-BTREE types;
- definitions with unreliable sort-direction metadata;
- primary-versus-secondary comparisons.

Skipped supported-but-unanalysed structures create caveats where useful.

### 15.3 Grouping

Equivalent indexes are grouped into one finding rather than pairwise findings.

Three equivalent indexes produce one duplicate-index finding with evidence for all three.

### 15.4 Output semantics

```text
severity:   warning
confidence: high
exactness:  exact
```

Guidance must say to review before removal. It must not claim an index is safe to drop.

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
- the covering relation is not leftmost-prefix;
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

Guidance must require workload/index-usage validation before removal.

## 17. Rule: `mysql.missing_primary_key`

Meaning:

> No observable `PRIMARY` index was found for a base table.

The rule must not claim that InnoDB has no internal clustered index.

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

Views are declared outside v0.0.1 analysis scope and represented as an inventory count rather than a caveat.

`0 findings + limited coverage` must never be rendered as equivalent to `0 findings + complete coverage`.

## 19. Scan Failure Model

### 19.1 Findings are not failures

Finding presence does not fail the command in v0.0.1.

```text
exit 0 — scan completed, regardless of finding count
```

### 19.2 Operational failures

```text
exit 1 — runtime, metadata collection, or report-write failure
exit 2 — invalid usage or unsupported connection/driver/server family
```

Examples that fail closed:

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

The report explicitly states that v0.0.1 does not inspect row data, query workload, source code, or views as rule targets.

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

The first format is `0.1.0`, not `1.0.0`.

Within the pre-1.0 period, breaking changes may advance the minor version. `1.0.0` is reserved until the contract is exercised by at least one external consumer, such as the main `dbprobe` runtime or CI integration.

Consumers:

- reject unknown major versions after 1.0 stabilization;
- ignore unknown compatible fields;
- use structured IDs/evidence rather than prose.

### 20.3 JSON Schema

The repository contains:

```text
resources/schemas/schema-scan-report-v0.1.schema.json
```

Schema identifier:

```text
urn:dbprobe:laravel:schema-scan-report:v0.1
```

No runtime JSON-Schema validation dependency is required. The schema is used by tests, fixtures, documentation, and future consumers.

## 21. Report Write Safety

The package guarantees that it does not intentionally expose a partially written JSON report and does not discard a previously valid report before the new report is ready.

Process:

1. create a temporary file in the target directory;
2. serialize and write the complete new report;
3. validate that the temporary content is valid JSON;
4. replace the target using the safest platform-supported operation;
5. preserve the previous valid report if replacement fails;
6. clean up temporary artifacts.

The design does not claim identical filesystem-level atomic-replace semantics across all operating systems.

## 22. Terminal Report

Human-readable output is findings-first and concise:

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

Terminal output must distinguish complete and limited coverage.

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
- caveat logic that can be tested from normalized metadata.

### 23.2 Package feature

Uses Orchestra Testbench without requiring real MySQL for most tests.

Covers:

- package auto-discovery/bootstrap;
- service-provider registration;
- `dbprobe:scan` registration;
- connection option propagation;
- exit codes;
- terminal behavior;
- report writer behavior;
- failure redaction;
- configuration precedence.

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

Integration tests verify actual `information_schema` mapping and rule outcomes.

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

Integration tests observe SQL executed by package scan code.

Allowed:

```text
SELECT
```

Forbidden:

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

Fixture setup may use DDL before the package scan begins; fixture DDL is not package behavior.

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

Expected verification commands include:

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

No percentage coverage gate is required for v0.0.1. Behavioral coverage of rules, failure paths, metadata query budget, read-only posture, contract stability, and determinism is more important than a raw percentage target.

## 27. Composer and Runtime Dependencies

Runtime requirements should be limited to the Illuminate components actually required, expected to include:

```text
illuminate/console
illuminate/database
illuminate/support
```

The package should not require the complete `laravel/framework` package solely for convenience.

Development dependencies may include Orchestra Testbench, PHPUnit, PHPStan/Larastan as appropriate, and Pint.

The runtime must not add:

- HTTP clients solely for DBProbe;
- LLM SDKs;
- telemetry SDKs;
- queue/cache packages;
- JSON Schema runtime validators;
- remote logging clients.

`composer.lock` is not committed for this reusable library.

`composer.json` does not contain a fixed `version` field; release versions come from Git tags/package metadata.

## 28. Public API Boundary

Supported v0.0.1 external contract:

1. `php artisan dbprobe:scan`;
2. `--connection`;
3. documented `config/dbprobe.php` keys;
4. JSON report format `dbprobe.laravel.schema-scan`;
5. finding IDs.

Internal PHP DTOs, rule interfaces, collector contracts, container bindings, and orchestration services may evolve during the 0.x series and are not yet third-party extension APIs.

## 29. README Contract

Initial README should document:

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

README must state clearly:

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

The approved design is the first repository commit on `main`. Implementation starts only after the design has been reviewed and an implementation plan has been approved.

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

The implementation branch is merged only after its required quality gates pass. `v0.0.1` is tagged only after all release criteria are satisfied.

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

### DLR-001 — Product scope

v0.0.1 is a Laravel 12/13, MySQL-only, read-only Schema Scan package.

### DLR-002 — Independent package

The package operates without the main `dbprobe` runtime.

### DLR-003 — Finding namespace

Finding IDs use the engine-namespaced `mysql.*` form compatible with the main `dbprobe` concepts.

### DLR-004 — Initial findings

Only `mysql.duplicate_index`, `mysql.redundant_index`, and `mysql.missing_primary_key` are included.

### DLR-005 — No AI dependency

LLMs, agents, query analysis, and auto-remediation are outside v0.0.1.

### DLR-006 — Internal flow

The core flow is Command -> Scan service -> Collector -> Snapshot -> Rule engine -> Reporters.

### DLR-007 — Internal seams

`SchemaCollector` and `Rule` are internal architectural seams, not public plugin contracts.

### DLR-008 — Bulk metadata collection

Metadata is collected through a bounded number of bulk queries rather than table-by-table inspection.

### DLR-009 — Pure rules

Rules evaluate snapshots without database/filesystem/Laravel/remote access.

### DLR-010 — Immutable snapshot

The normalized schema snapshot is immutable and canonicalized.

### DLR-011 — Structural fingerprint

Each snapshot carries a stable SHA-256 structural fingerprint.

### DLR-012 — Main-project conceptual compatibility

Finding fields remain conceptually compatible with the main `dbprobe` finding model.

### DLR-013 — Findings do not fail CI

v0.0.1 returns zero for successful scans even when findings exist.

### DLR-014 — Safe report replacement

Reports are written through a temporary complete file and safely replace the previous valid report.

### DLR-015 — YAGNI

No facade, adapter registry, event system, persistence store, plugin loader, or AI client is included.

### DLR-016 — Pre-1.0 report contract

JSON report format starts at `0.1.0`.

### DLR-017 — Structured machine contract

Consumers use IDs/enums/evidence, not prose parsing.

### DLR-018 — Structural data only

No rows, application SQL, credentials, or connection endpoints are collected.

### DLR-019 — No full schema manifest artifact yet

The report contains inventory, fingerprint, findings, evidence, and caveats, not a standalone full schema export.

### DLR-020 — Canonical fingerprint projection

Volatile and approximate fields are excluded from the fingerprint.

### DLR-021 — Conservative duplicate analysis

Exact duplicate findings are limited to safely comparable column-based BTREE secondary indexes.

### DLR-022 — Duplicate grouping

Equivalent indexes are reported as one grouped finding.

### DLR-023 — Redundancy means candidate

Left-prefix redundancy is a workload-validation candidate, never a drop authorization.

### DLR-024 — Redundancy exclusions

Unique, primary-covering, invisible, functional, prefix-mismatched, and order-mismatched cases are conservatively excluded.

### DLR-025 — Primary-key semantics

Missing-primary-key analysis distinguishes explicit primary keys from eligible `UNIQUE NOT NULL` InnoDB fallback candidates.

### DLR-026 — GIPK ambiguity

Ambiguous/hidden generated primary-key metadata causes a skip/caveat rather than a false-positive finding.

### DLR-027 — Fail closed on core metadata failure

Missing core metadata prevents a new successful report.

### DLR-028 — Limited coverage is explicit

Unsupported-but-observable structures may be skipped with `analysis_coverage=limited` and structured caveats.

### DLR-029 — Contract fixtures

The JSON contract is protected by a versioned JSON Schema and golden tests.

### DLR-030 — Format stabilization later

Report format reaches `1.0.0` only after real external-consumer validation.

### DLR-031 — Cross-platform write guarantee

The guarantee is no partial-success report and preservation of the previous valid report on failed replacement, not identical filesystem atomicity everywhere.

### DLR-032 — Four test layers

Unit, package feature, MySQL integration, and contract tests are mandatory.

### DLR-033 — Five-query normal-path budget

Normal metadata collection uses at most five read-only SELECTs independent of table count.

### DLR-034 — No writes by scan code

Package scan behavior executes no DDL, DML, or session-mutating statement.

### DLR-035 — Bounded CI matrix

CI verifies representative minimum/current runtime boundaries rather than every cross-product.

### DLR-036 — Windows smoke

Windows verifies package bootstrap and report/path semantics.

### DLR-037 — Quality gates

Composer validation, Pint, PHPStan level 8, PHPUnit suites, and real MySQL integration tests form the initial gate.

### DLR-038 — Minimal runtime dependencies

Only necessary Illuminate components are runtime dependencies.

### DLR-039 — Library versioning conventions

No committed `composer.lock` and no fixed Composer `version` field.

### DLR-040 — Limited public API

CLI/config/report/finding IDs are supported external contracts; internal PHP types are not frozen plugin APIs.

### DLR-041 — Vertical test-first delivery

Implementation proceeds by working vertical slices without speculative scaffolding.

### DLR-042 — Simple branching

Use `main` plus short-lived feature branches; no `develop` branch.

### DLR-043 — Design first

This approved design document is the first repository commit.

### DLR-044 — Release only after evidence

`v0.0.1` is tagged only when all completion criteria and CI gates pass.

## 34. Deferred Evolution

The architecture intentionally leaves room for later, separately designed milestones such as:

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
