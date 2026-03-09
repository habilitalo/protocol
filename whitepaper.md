# Habilitalo Protocol v0.1

**An Open Connector Protocol for Financial Data Integration**

Franco Danioni -- Digitalo SpA -- habilitalo.com

March 2026

---

## Table of Contents

1. [Abstract](#abstract)
2. [The Problem](#the-problem)
3. [What Habilitalo Is](#what-habilitalo-is)
4. [Core Concepts](#core-concepts)
   - [Connector](#connector)
   - [Canonical Event](#canonical-event)
   - [Registry](#registry)
5. [The 3-Stage Pipeline](#the-3-stage-pipeline)
   - [Stage 1: Ingest](#stage-1-ingest)
   - [Stage 2: Process](#stage-2-process)
   - [Stage 3: Analyze](#stage-3-analyze)
6. [Deduplication](#deduplication)
7. [Double-Entry Ledger](#double-entry-ledger)
8. [Foundational Connectors](#foundational-connectors)
   - [Fintoc (Open Banking Chile)](#fintoc-open-banking-chile)
   - [CSV (Bank Statements)](#csv-bank-statements)
   - [Manual (Structured Forms)](#manual-structured-forms)
9. [Event System](#event-system)
10. [Analysis Engines](#analysis-engines)
    - [Anomaly Detection](#anomaly-detection)
    - [Recurrence Detection](#recurrence-detection)
11. [Stack Position](#stack-position)
12. [Technical Implementation](#technical-implementation)
13. [Governance](#governance)
14. [Roadmap](#roadmap)
15. [Appendix A: Connector Manifest Schema](#appendix-a-connector-manifest-schema)
16. [Appendix B: Event Type Reference](#appendix-b-event-type-reference)
17. [Appendix C: Pipeline Data Flow](#appendix-c-pipeline-data-flow)

---

## Abstract

Every fintech that connects to a bank, processes a CSV, or receives a webhook
faces the same problem: transforming messy external data into clean, internal
records. The industry's answer has been intermediary platforms -- Plaid, MX,
Yodlee, Salt Edge -- that abstract the bank away but replace one dependency with
another. You trade vendor lock-in for aggregator lock-in, pay per-connection
rent in perpetuity, and lose control over the data pipeline between your source
and your ledger.

We built Habilitalo to solve this for ourselves. It started as the integration
layer inside Balancealo, the financial observability platform for Grupo
Digitalo. After building three connectors, a double-entry ledger, an event
system, and two analysis engines, we realized the pattern was general enough to
be a protocol.

This whitepaper describes that protocol as it exists today: approximately 5,500
lines of TypeScript across 24 files, running in production. It is not a
roadmap document. Everything described here has been implemented, tested, and
deployed. Where we note future work, we say so explicitly.

The thesis is simple: integration should be a protocol, not a platform. A
connector should be a declaration -- what it receives, what it emits, how it
authenticates -- accompanied by a transform function. The rest (deduplication,
ledger posting, event delivery, anomaly detection) is infrastructure that any
connector can use without building it themselves.

---

## The Problem

Financial data integration is broken in three specific ways.

### 1. Fragmentation

Every financial institution exposes data differently. Chilean banks through
Fintoc use `post_date` and nested `sender_account.holder_name`. A bank
statement CSV might have `Fecha`, `Monto`, and `Glosa` with Chilean number
formatting (`1.234.567,89`). Another CSV uses `Date`, `Amount`, and
`Description` with US formatting (`1,234,567.89`). A manual entry from a form
has yet another shape.

Each source requires its own parsing logic, its own date handling, its own
amount normalization. Teams build this from scratch every time. The parsing
code is straightforward; the problem is that it gets entangled with business
logic, making it impossible to reuse.

### 2. Lock-In

The aggregator platforms that promise to solve fragmentation create their own
dependency. Your data pipeline becomes: Bank -> Aggregator -> Your App.
Switching aggregators means rewriting your ingestion layer. The aggregator
controls the data format, the delivery mechanism, the sync schedule, and the
error handling. You cannot inspect or modify the pipeline between the bank and
your database.

### 3. Rent Extraction

Aggregators charge per connection, per account, per month. This makes economic
sense for them but creates perverse incentives. A fintech serving small
businesses in Latin America cannot afford $0.50/connection/month when their
average customer has 3 accounts and pays $5/month total. The aggregator takes
30% of revenue for doing what amounts to an HTTP call and a data transform.

---

## What Habilitalo Is

Habilitalo is an open Connector protocol that separates the _what_ of
integration (what data comes in, what events go out) from the _how_ (the
pipeline that processes it).

A Connector is a declaration with a transform function. The declaration tells
the system what transport the connector uses (webhook, file upload, API poll,
manual entry), what events it can receive from the external source, what
canonical events it emits into the system, and what authentication it requires.

The transform function takes raw source data and a context object, and returns
a `CanonicalEvent` -- a uniform envelope that the rest of the system can
process without knowing anything about the source.

Everything downstream of the Connector -- staging, deduplication, ledger
posting, event delivery, anomaly detection -- is shared infrastructure.
Building a new connector means writing one transform function and one JSON
manifest. The pipeline handles the rest.

### Design Principles

1. **Declaration over configuration.** A connector manifest (`habilitalo.connector.json`) declares capabilities. No imperative setup code.

2. **Transform isolation.** The transform function is the only code a connector author writes. It receives raw data and returns a canonical event. Side effects (database writes, webhooks, analysis) happen in the pipeline.

3. **At-least-once delivery.** Events are persisted before delivery is attempted. The system tolerates duplicate delivery; consumers are expected to be idempotent.

4. **Hash-based deduplication.** Every entry gets a deterministic SHA-256 hash. The same data ingested twice produces the same hash and is silently dropped.

5. **Double-entry correctness.** Every financial movement produces a balanced journal entry. Debits equal credits, always. The system rejects unbalanced entries at the validation layer.

6. **Fire-and-forget events.** Event emission never blocks the pipeline. If webhook delivery fails after retries, the pipeline continues. Events are a notification mechanism, not a control flow mechanism.

---

## Core Concepts

### Connector

The `HabilitaloConnector` interface is the central abstraction. Every data
source -- a bank API, a file upload, a manual form -- is represented as a
Connector.

```typescript
interface HabilitaloConnector {
  /** Unique connector identifier (e.g. "fintoc-v1") */
  id: string

  /** Semver version of this connector */
  version: string

  /** What this connector can receive */
  inbound: InboundDeclaration

  /** What canonical events this connector emits */
  outbound: OutboundDeclaration

  /** Authentication requirements */
  auth: AuthDeclaration

  /** Transform function: raw source data -> CanonicalEvent */
  transform: HabilitaloTransform
}
```

The `InboundDeclaration` specifies the transport mechanism and the event types
the connector handles from the source:

```typescript
interface InboundDeclaration {
  /** Transport mechanism: webhook, file, manual, api_poll */
  transport: 'webhook' | 'file' | 'manual' | 'api_poll'

  /** Event types this connector can receive from the source */
  events: string[]
}
```

The `OutboundDeclaration` specifies what canonical event types this connector
produces:

```typescript
interface OutboundDeclaration {
  /** Canonical event types this connector emits */
  events: string[]
}
```

The `AuthDeclaration` tells the system what credentials are required to
activate the connector for a given tenant:

```typescript
interface AuthDeclaration {
  /** Authentication type required by the source */
  type: 'api_key' | 'oauth2' | 'link_token' | 'none'

  /** Required credential fields (e.g. ["apiKey", "secretKey"]) */
  requiredFields?: string[]
}
```

The transform function receives raw data from the source and a context object
containing tenant and tracing information:

```typescript
type HabilitaloTransform = (
  raw: unknown,
  context: ConnectorContext
) => CanonicalEvent

interface ConnectorContext {
  /** Organization/tenant ID */
  tenantId: string
  /** Data feed ID (if applicable) */
  feedId?: string
  /** Connector configuration (source-specific) */
  config: Record<string, unknown>
  /** Trace ID for distributed tracing */
  traceId: string
}
```

### Canonical Event

Every connector emits events in a single canonical shape. This is the
universal envelope that flows through the entire system -- from connector to
pipeline to webhook to analysis engine.

```typescript
interface CanonicalEvent {
  /** Unique event identifier (UUIDv4) */
  id: string

  /** ISO8601 timestamp of when the event was emitted */
  timestamp: string

  /** System that originated the event (e.g. "habilitalo") */
  source: string

  /** Event type in dot-notation (e.g. "bank.movement.created") */
  type: string

  /** Schema version of this event type (semver) */
  version: string

  /** Event-specific payload -- shape depends on `type` */
  payload: Record<string, unknown>

  /** Routing and tracing metadata */
  meta: {
    /** Connector ID that produced this event */
    connector: string
    /** Organization/tenant ID */
    tenant: string
    /** Trace ID for distributed tracing */
    trace_id: string
  }
}
```

The `createCanonicalEvent` helper provides sensible defaults:

```typescript
function createCanonicalEvent(
  params: Pick<CanonicalEvent, 'type' | 'payload' | 'meta'> & {
    id?: string
    version?: string
    source?: string
  }
): CanonicalEvent {
  return {
    id: params.id ?? crypto.randomUUID(),
    timestamp: new Date().toISOString(),
    source: params.source ?? 'habilitalo',
    type: params.type,
    version: params.version ?? '1.0.0',
    payload: params.payload,
    meta: params.meta,
  }
}
```

### Registry

The `ConnectorRegistry` is the runtime catalog of all available connectors. It
serves two purposes: it is the lookup mechanism for the pipeline (given a feed
type, find the right connector) and it is the compatibility bridge between the
new protocol layer and the existing `SourceAdapter` system.

```typescript
const ConnectorRegistry = {
  register(connector: HabilitaloConnector, adapter?: SourceAdapter): void
  get(id: string): HabilitaloConnector | undefined
  list(): HabilitaloConnector[]
  has(id: string): boolean
  getSourceAdapter: getSourceAdapter  // Legacy bridge
  listSourceAdapters: listSourceAdapters  // Legacy bridge
}
```

The `bridgeAdapterToConnector` function wraps an existing `SourceAdapter` in
the `HabilitaloConnector` interface without modifying the adapter code. This
is how the three foundational connectors were brought into the protocol -- their
transform logic remained untouched:

```typescript
function bridgeAdapterToConnector(
  adapter: SourceAdapter,
  manifest: {
    id: string
    version: string
    inbound: HabilitaloConnector['inbound']
    outbound: HabilitaloConnector['outbound']
    auth: HabilitaloConnector['auth']
  }
): HabilitaloConnector
```

The bridge creates a `HabilitaloConnector` whose transform function wraps the
adapter's output in a `CanonicalEvent` envelope. The actual data transformation
still happens in the adapter's `transform` method during the ingest stage; the
protocol-level transform is for event wrapping only.

---

## The 3-Stage Pipeline

The pipeline is the core of Habilitalo. It takes raw data from any source and
produces balanced ledger entries, emitted events, and analysis results. The
three stages are sequential and decoupled -- each stage reads from and writes
to the database, so a failure in one stage does not corrupt the others.

```
                         Stage 1              Stage 2              Stage 3
                        (Ingest)             (Process)           (Analyze)
                     +------------+      +--------------+     +------------+
  External Source -> | Adapter    | ---> | RawEntry     | --> | JournalEntry|
  (bank, CSV,       | Transform  |      | (staging)    |     | (ledger)    |
   form)            | Validate   |      | Deduplicate  |     | Balanced    |
                    | Hash       |      | Double-entry |     | debit/credit|
                    +------------+      +--------------+     +------------+
                                                                    |
                                                              +-----+------+
                                                              |            |
                                                        [Events]    [Analysis]
                                                              |            |
                                                        Webhooks    Anomalies
                                                                    Patterns
```

### Stage 1: Ingest

The ingest stage transforms raw source data into `RawEntry` records in the
staging table. This is where source-specific parsing happens: date formats,
amount formats, field mapping, currency detection.

**Input:** Raw source data (Fintoc movements, CSV rows, manual form data).

**Process:**

1. The appropriate `SourceAdapter` transforms source data into `RawEntryInput[]`.
2. Each entry is validated: required fields, amount bounds, date sanity, currency validity.
3. A SHA-256 hash is calculated from normalized fields: `date|amount|accountId|description|reference`.
4. The hash is checked against existing entries in the same organization.
5. If duplicate: silently skip (or use sequence suffix for legitimate collisions).
6. If new: create a `RawEntry` with status `pending`.

**Output:** `IngestResult` with counts of staged, duplicate, and error entries.

```typescript
interface IngestResult {
  staged: number
  duplicates: number
  errors: number
  entries: Array<{
    id?: string
    status: 'staged' | 'duplicate' | 'error'
    hash?: string
    error?: string
  }>
}
```

The `RawEntryInput` is the universal intermediate format that every adapter
produces:

```typescript
interface RawEntryInput {
  dataFeedId: string
  organizationId: string
  transactionDate: Date
  postDate?: Date | null
  amount: number
  currency?: Currency
  description: string
  counterparty?: string | null
  reference?: string | null
  sourceId?: string | null
  rawData: Record<string, unknown>
}
```

Key design decisions in the ingest stage:

- **Batch vs. individual.** For batches under 100 entries, we use `createMany` with pre-calculated hashes and a single existing-hash query. For larger batches, we fall back to individual processing to avoid long-running transactions.
- **Raw data preservation.** The original source data is stored in `rawData` as a JSON column. This allows re-processing or auditing without re-fetching from the source.
- **Feed statistics.** After successful ingestion, the `DataFeed` record is updated with `totalEntries` increment and `lastSyncAt` timestamp.

### Stage 2: Process

The processor takes pending `RawEntry` records and converts them into
double-entry `JournalEntry` records in the ledger.

**Input:** `RawEntry` records with status `pending`.

**Process:**

1. Fetch pending entries ordered by transaction date.
2. For each entry, mark as `processing` (prevents concurrent processing).
3. Race condition check: verify no other entry with the same hash is already `processed`.
4. Determine transaction direction from amount sign (positive = income, negative = expense).
5. Create a `JournalEntry` with balanced debit/credit `JournalLine[]`.
6. Mark `RawEntry` as `processed` with a reference to the `JournalEntry`.
7. Fire-and-forget: emit `bank.movement.created` event.

**Output:** `ProcessorResult` with counts and per-entry details.

```typescript
interface ProcessorResult {
  processed: number
  duplicates: number
  errors: number
  details: Array<{
    rawEntryId: string
    status: 'processed' | 'duplicate' | 'error'
    journalEntryId?: string
    error?: string
  }>
}
```

**Error handling:** If processing fails for an entry, it is marked as `error`
with the error message stored. The pipeline continues with the next entry.
Failed entries can be reprocessed later via `reprocessFailedEntries`, which
resets their status to `pending` and runs the processor again.

### Stage 3: Analyze

The analysis stage runs asynchronously after ledger posting. It examines
journal entries for two classes of patterns:

1. **Anomalies:** Transactions that deviate significantly from historical norms.
2. **Recurrences:** Repeating patterns (subscriptions, salary, rent, etc.).

Both engines read from the `JournalEntry` and `JournalLine` tables and write
their results to dedicated tables (`Anomaly`, `RecurringPattern`,
`RecurringOccurrence`).

Analysis is described in detail in the [Analysis Engines](#analysis-engines)
section.

---

## Deduplication

Deduplication is critical for a system that ingests data from multiple sources
on overlapping schedules. Fintoc may deliver the same movement on two
consecutive syncs. A user may upload the same CSV twice. The system must
silently absorb duplicates without creating double entries in the ledger.

### Hash Calculation

Every entry gets a SHA-256 hash computed from five normalized fields:

```
SHA-256( date | amount | accountId | description | reference )
```

Normalization rules:

- **Date:** Converted to `YYYY-MM-DD` (time component stripped).
- **Amount:** Absolute value with exactly 2 decimal places.
- **AccountId:** Used as-is (UUID).
- **Description:** Lowercased, accents removed (NFD normalization), special characters stripped, multiple spaces collapsed.
- **Reference:** Same normalization as description, included only if present.

The hash is deterministic: the same transaction from any source at any time
produces the same hash.

```typescript
function calculateEntryHash(entry: HashableEntry): HashResult {
  const normalizedFields = {
    date: normalizeDate(entry.transactionDate),
    amount: normalizeAmount(entry.amount),
    accountId: entry.accountId,
    description: normalizeString(entry.description),
    reference: entry.reference ? normalizeString(entry.reference) : '',
  }

  const parts = [
    normalizedFields.date,
    normalizedFields.amount,
    normalizedFields.accountId,
    normalizedFields.description,
  ]

  if (normalizedFields.reference) {
    parts.push(normalizedFields.reference)
  }

  const hashInput = parts.join('|')
  const hash = createHash('sha256').update(hashInput).digest('hex')

  return { hash, normalizedFields }
}
```

### Collision Handling

Legitimate collisions occur when a person makes two identical purchases on the
same day (same amount, same merchant, same description). The system
distinguishes these by checking `sourceId` -- if two entries have the same
hash but different source identifiers, they are treated as distinct
transactions.

For legitimate collisions, a sequence suffix is appended:

```typescript
function calculateEntryHashWithSequence(
  entry: HashableEntry,
  sequence: number
): string {
  const { hash: baseHash } = calculateEntryHash(entry)
  if (sequence === 0) return baseHash
  return createHash('sha256')
    .update(`${baseHash}|seq:${sequence}`)
    .digest('hex')
}
```

The system tries up to 10 sequence numbers before giving up and marking the
entry as a duplicate.

### Database Constraint

The deduplication hash is enforced at the database level with a unique
constraint on `(organizationId, entryHash)`. This provides a safety net
against race conditions that the application-level check might miss. The bulk
ingest path uses `createMany` with `skipDuplicates: true` as an additional
guard.

---

## Double-Entry Ledger

Every financial movement in Habilitalo is recorded as a `JournalEntry` with
balanced `JournalLine[]` records. This is proper double-entry bookkeeping:
for every debit, there is an equal credit, per currency.

### Data Model

```
JournalEntry
  id: UUID
  organizationId: UUID
  entryNumber: auto-incrementing per org
  description: string
  transactionDate: Date
  postDate: Date (defaults to creation time)
  status: 'draft' | 'posted' | 'voided'
  sourceType: DataFeedType (fintoc, csv_import, manual, etc.)
  sourceRef: string (reference to RawEntry or other source)
  lines: JournalLine[]

JournalLine
  id: UUID
  journalEntryId: UUID
  accountId: UUID
  entryType: 'debit' | 'credit'
  amount: Decimal
  currency: Currency
  description: string
  counterparty: string
  categoryId: UUID (optional)
  ownerId: UUID (optional)
```

### Balance Validation

Before any `JournalEntry` is created, the system validates that debits equal
credits for each currency:

```typescript
function validateJournalLinesBalance(
  lines: Array<{ entryType: 'debit' | 'credit'; amount: number; currency: string }>
): ValidationResult {
  // Group by currency
  const byCurrency = new Map<string, { debits: number; credits: number }>()

  for (const line of lines) {
    const curr = byCurrency.get(line.currency) || { debits: 0, credits: 0 }
    if (line.entryType === 'debit') {
      curr.debits += line.amount
    } else {
      curr.credits += line.amount
    }
    byCurrency.set(line.currency, curr)
  }

  // Check balance for each currency (epsilon = 0.01 for floating point)
  for (const [currency, { debits, credits }] of byCurrency) {
    const diff = Math.abs(debits - credits)
    if (diff > 0.01) {
      errors.push(
        `Lines in ${currency} are not balanced: debits=${debits}, credits=${credits}`
      )
    }
  }

  return { valid: errors.length === 0, errors, warnings }
}
```

An entry that fails balance validation is rejected with an error. The system
never creates an unbalanced journal entry.

### Account Nature

Accounts have a `nature` that determines how their balance is calculated:

- **`debit_normal`** (assets, expenses): Balance = SUM(debits) - SUM(credits). A positive balance means the account holds value.
- **`credit_normal`** (liabilities, equity, revenue): Balance = SUM(credits) - SUM(debits). A positive balance means the account owes value.

```typescript
async function calculateAccountBalance(
  accountId: string,
  asOfDate?: Date,
  includeVoided = false
): Promise<AccountBalance> {
  // ... fetch account and aggregate lines ...

  const balance =
    account.accountNature === 'debit_normal'
      ? debits - credits
      : credits - debits

  return {
    accountId: account.id,
    accountName: account.name,
    accountNature: account.accountNature,
    totalDebits: debits,
    totalCredits: credits,
    balance,
    currency: account.currency,
  }
}
```

### Transaction Types

The ledger supports three simple transaction types that map to journal entries:

- **Income:** DEBIT the bank account (increase asset), CREDIT the revenue contra (tracked via category).
- **Expense:** DEBIT the expense contra (tracked via category), CREDIT the bank account (decrease asset).
- **Transfer:** DEBIT the destination account, CREDIT the source account.

Each type produces a balanced two-line journal entry. The `createSimpleEntry`
function dispatches to the appropriate creator:

```typescript
async function createSimpleEntry(
  input: SimpleTransactionInput & { organizationId: string }
): Promise<JournalEntryWithLines> {
  switch (input.type) {
    case 'income':
      return createIncomeEntry(input)
    case 'expense':
      return createExpenseEntry(input)
    case 'transfer':
      return createTransferEntry(input)
  }
}
```

### Void/Reversal

Journal entries can be voided but never deleted. A voided entry retains its
lines and metadata but is excluded from balance calculations (unless
`includeVoided` is explicitly set). The void operation records a reason and
timestamp for audit purposes.

---

## Foundational Connectors

Habilitalo ships with three connectors that cover the most common ingestion
scenarios in Latin American fintech. Each connector is a `SourceAdapter` that
has been wrapped in the `HabilitaloConnector` protocol via the bridge function.

### Fintoc (Open Banking Chile)

**Connector ID:** `fintoc-v1`
**Transport:** webhook
**Auth:** link_token
**Emits:** `bank.movement.created`

Fintoc provides Open Banking access to Chilean financial institutions. The
connector transforms `FintocMovement` objects into `RawEntryInput` records.

**Field mapping:**

| Fintoc Field | RawEntryInput Field | Notes |
|---|---|---|
| `post_date` | `postDate` | ISO date string |
| `transaction_date` | `transactionDate` | Falls back to `post_date` |
| `amount` | `amount` | Positive = income, negative = expense |
| `currency` | `currency` | Mapped: CLP, USD, EUR, UF |
| `description` + `comment` | `description` | Concatenated with separator |
| `sender_account.holder_name` | `counterparty` | For income (amount > 0) |
| `recipient_account.holder_name` | `counterparty` | For expense (amount < 0) |
| `reference_id` | `reference` | Used in hash calculation |
| `id` | `sourceId` | Fintoc movement ID for dedup |
| `sender_account.holder_id` | `rawData.counterpartyRut` | Chilean RUT for transfer detection |

**Incremental sync strategy:**

The Fintoc integration uses a chunked synchronization approach designed to
handle accounts with large transaction histories without hitting timeout
limits:

1. **24-month window:** Syncs up to 2 years of historical data.
2. **3 months per chunk:** Each invocation processes 3 months of movements.
3. **Parallel accounts:** All accounts within a link are synced in parallel for each month.
4. **Pipeline integration:** Movements flow through the full pipeline: `FintocMovement -> FintocAdapter.transform() -> ingestEntriesBulk() -> processDataFeed()`.
5. **Resumable:** Sync progress is persisted as JSON on the `FintocLink` record. If a chunk fails, the next invocation resumes from where it left off.

The sync is called repeatedly by a cron job until `isComplete` returns `true`.

**Connector manifest** (`habilitalo.connector.json`):

```json
{
  "id": "fintoc-v1",
  "name": "Fintoc Open Banking",
  "version": "1.0.0",
  "category": "financial",
  "tags": ["banking", "chile", "latam"],
  "inbound": {
    "transport": "webhook",
    "events": ["payment.received"]
  },
  "outbound": {
    "events": ["bank.movement.created"]
  },
  "auth": {
    "type": "link_token",
    "requiredFields": ["linkToken"]
  },
  "habilitalo": "0.1"
}
```

### CSV (Bank Statements)

**Connector ID:** `csv-v1`
**Transport:** file
**Auth:** none
**Emits:** `bank.movement.created`

The CSV connector parses bank statement files with intelligent format
detection. It handles the diversity of CSV formats encountered across Latin
American and international banks.

**Date format detection:**

The parser tries formats in order of specificity:

1. ISO format first: `YYYY-MM-DD` (regex: `/^\d{4}-\d{2}-\d{2}/`)
2. Day-first formats: `DD/MM/YYYY`, `DD-MM-YYYY`, `DD.MM.YYYY` (default for Latin America)
3. US format: `MM/DD/YYYY` (only if explicitly configured)
4. Native `Date` parsing as last resort

**Amount format detection:**

The parser auto-detects between Chilean and US number formatting:

| Format | Thousands | Decimal | Example |
|---|---|---|---|
| Chilean | `.` (dot) | `,` (comma) | `1.234.567,89` |
| US/International | `,` (comma) | `.` (dot) | `1,234,567.89` |

Detection heuristic: if a string contains exactly one comma followed by 1-2
digits at the end, the comma is the decimal separator. Otherwise, the dot is
assumed to be decimal.

**Column auto-detection:**

If column names are not explicitly configured, the parser tries common
variations:

| Field | Fallback Column Names |
|---|---|
| Date | `fecha`, `date`, `transaction_date`, `transactiondate` |
| Amount | `monto`, `amount`, `valor`, `value`, `importe` |
| Description | `descripcion`, `description`, `detalle`, `glosa`, `concepto` |
| Counterparty | `contraparte`, `counterparty`, `comercio`, `merchant` |
| Reference | `referencia`, `reference`, `ref`, `numero` |

Column matching is case-insensitive.

**Connector manifest:**

```json
{
  "id": "csv-v1",
  "name": "CSV File Import",
  "version": "1.0.0",
  "category": "financial",
  "tags": ["csv", "file", "import", "bank-statement"],
  "inbound": {
    "transport": "file",
    "events": ["file.uploaded"]
  },
  "outbound": {
    "events": ["bank.movement.created"]
  },
  "auth": {
    "type": "none"
  },
  "habilitalo": "0.1"
}
```

### Manual (Structured Forms)

**Connector ID:** `manual-v1`
**Transport:** manual
**Auth:** none
**Emits:** `bank.movement.created`

The Manual connector is a passthrough adapter for structured data submitted
through forms or API calls. It validates and normalizes the data but does not
perform any parsing -- the data is expected to be well-formed.

The adapter accepts either a single entry or an array of entries. Each entry
must include at minimum: `transactionDate`, `amount`, and `description`.

This connector exists to ensure that manually entered transactions flow
through the same pipeline (hash dedup, ledger posting, event emission,
analysis) as automated sources. There is no special path for manual data.

**Connector manifest:**

```json
{
  "id": "manual-v1",
  "name": "Manual Entry",
  "version": "1.0.0",
  "category": "financial",
  "tags": ["manual", "entry", "form"],
  "inbound": {
    "transport": "manual",
    "events": ["entry.submitted"]
  },
  "outbound": {
    "events": ["bank.movement.created"]
  },
  "auth": {
    "type": "none"
  },
  "habilitalo": "0.1"
}
```

---

## Event System

Habilitalo emits events via webhooks when significant things happen in the
pipeline. Events are the mechanism by which downstream services (Compensalo for
payments, Coordinalo for scheduling, third-party integrations) learn about
financial activity.

### Architecture

The event system is built on three database tables:

- **`WebhookEndpoint`**: A registered URL that receives events. Has a list of subscribed event types, a secret for HMAC signing, and failure tracking.
- **`WebhookDelivery`**: A record of each delivery attempt. Persisted _before_ the first attempt (at-least-once guarantee).
- **The event emitter**: The `emit()` function that orchestrates delivery.

### Delivery Semantics

**At-least-once:** A `WebhookDelivery` record is created in the database
before the first HTTP request is made. If the process crashes between creating
the record and delivering the webhook, the delivery can be retried from the
persisted state.

**Fire-and-forget:** The `emit()` function never throws an exception. All
errors are caught and logged. The pipeline that triggers the event (e.g., the
processor creating a journal entry) is never blocked or rolled back due to
event delivery failures.

**Retry with exponential backoff:** Failed deliveries are retried up to 3
times with delays of 1 second, 4 seconds, and 16 seconds (base 1s, factor 4x):

```
Attempt 1: immediate
Attempt 2: after 1s    (1000ms * 4^0)
Attempt 3: after 4s    (1000ms * 4^1)
          (give up after 3 failures)
```

**Non-retryable errors:** HTTP 4xx responses (except 429 Too Many Requests)
are not retried. A 400 or 403 indicates a permanent problem that retrying
will not fix.

### HMAC-SHA256 Signing

Every webhook delivery is signed with the endpoint's secret using HMAC-SHA256.
The signature is sent in the `X-Habilitalo-Signature` header:

```typescript
const signature = createHmac('sha256', endpoint.secretHash)
  .update(payloadString)
  .digest('hex')

// Headers sent:
// Content-Type: application/json
// X-Habilitalo-Event: bank.movement.created
// X-Habilitalo-Signature: <hex-encoded HMAC>
```

Consumers verify the signature by computing the same HMAC with their copy of
the secret and comparing it to the header value.

### Event Filtering

Each `WebhookEndpoint` has an `events` array that specifies which event types
it wants to receive. The wildcard `*` subscribes to all events. The emitter
filters endpoints before delivery:

```typescript
const subscribedEndpoints = endpoints.filter((ep) => {
  const events = ep.events as string[]
  return Array.isArray(events) && (
    events.includes(eventType) || events.includes('*')
  )
})
```

### Event Types

The system currently defines four event types:

| Event Type | Trigger | Payload |
|---|---|---|
| `bank.movement.created` | Processor creates a JournalEntry from a RawEntry | Movement details, account, amount, counterparty, journal entry reference |
| `analysis.anomaly.detected` | Anomaly detector flags a transaction | Anomaly type, severity, Z-score, expected vs. detected values |
| `analysis.recurrence.detected` | Recurrence detector identifies a pattern | Pattern name, frequency, amount, confidence, next expected date |
| `ledger.entry.created` | A JournalEntry is posted to the ledger | Full journal entry with all lines |

Each event type has a typed schema (TypeScript interface) that defines the
payload structure. For example, `BankMovementCreatedEvent`:

```typescript
interface BankMovementCreatedEvent {
  eventType: 'bank.movement.created'
  eventId: string          // UUIDv4
  source: 'habilitalo'
  timestamp: string        // ISO8601
  data: {
    movementId: string
    accountId: string
    institutionId: string
    amount: number
    currency: 'CLP'
    date: string           // ISO8601
    description: string
    counterparty: string | null
    type: 'credit' | 'debit'
    category: string | null
    categoryId: string | null
    categorySource: 'fintoc' | 'rule' | 'ai' | 'manual' | null
    rawFintocId: string
    journalEntryId: string
  }
}
```

### Webhook Management API

Webhook endpoints are managed through REST API routes at
`/api/habilitalo/webhooks`:

- `POST /api/habilitalo/webhooks` -- Create a new endpoint (requires HTTPS URL, secret, event list, org admin access).
- `GET /api/habilitalo/webhooks?organizationId=...` -- List active endpoints.
- `GET /api/habilitalo/webhooks/:id/deliveries` -- View delivery history for an endpoint.

---

## Analysis Engines

Habilitalo includes two analysis engines that run on ledger data. They are
designed to surface insights that would be difficult for a human to detect
manually: statistical anomalies and recurring patterns.

### Anomaly Detection

The anomaly detector uses Z-score analysis on per-category transaction
statistics to flag outliers. It detects three types of anomalies:

#### 1. Unusual Amount (`unusual_amount`)

For each transaction category, the system maintains running statistics (mean,
standard deviation, min, max, median) computed from journal lines in the
lookback period (default: 90 days, minimum 10 samples for statistical
significance).

A transaction's Z-score is calculated as:

```
Z = |amount - category_mean| / category_stddev
```

The Z-score is compared against configurable thresholds:

| Severity | Default Z-score Threshold |
|---|---|
| Low | 2.0 |
| Medium | 2.5 |
| High | 3.0 |
| Critical | 4.0 |

**Sensitivity levels** adjust all thresholds by a multiplier:

| Sensitivity | Multiplier | Effect |
|---|---|---|
| Low | 1.3x | Fewer alerts, only extreme outliers |
| Medium | 1.0x | Default behavior |
| High | 0.7x | More alerts, catches smaller deviations |

Example: At high sensitivity, the "low" threshold becomes `2.0 * 0.7 = 1.4`,
meaning transactions 1.4 standard deviations from the mean are flagged.

#### 2. New Counterparty (`new_counterparty`)

Flags the first transaction with a previously unseen counterparty when the
amount exceeds a configurable threshold (default: CLP 100,000). Severity
scales with amount:

- `low`: Amount >= threshold
- `medium`: Amount >= 2x threshold
- `high`: Amount >= 5x threshold

Counterparty comparison uses normalized strings (lowercased, accents removed,
special characters stripped) to avoid false positives from formatting
differences.

#### 3. Duplicate Suspect (`duplicate_suspect`)

Finds transactions with identical amount and normalized description within a
3-day window. These are flagged as potential duplicates that may have evaded
hash-based deduplication (e.g., because they came from different sources with
different raw data).

**Configuration:**

```typescript
interface AnomalyConfig {
  lowThreshold: number            // Default: 2.0
  mediumThreshold: number         // Default: 2.5
  highThreshold: number           // Default: 3.0
  criticalThreshold: number       // Default: 4.0
  minSampleSize: number           // Default: 10
  lookbackDays: number            // Default: 90
  newCounterpartyAmountThreshold: number  // Default: 100000 (CLP)
}
```

**Anomaly lifecycle:**

Detected anomalies are persisted with status `pending`. They can be reviewed
and marked as `acknowledged`, `false_positive`, or `resolved`. This feedback
loop is essential -- false positives are expected, and the review process
prevents alert fatigue.

### Recurrence Detection

The recurrence detector identifies repeating transaction patterns:
subscriptions, salary deposits, rent payments, utility bills, and similar
periodic movements.

**Algorithm:**

1. **Group transactions** by normalized description and amount bucket. The
   amount bucket uses logarithmic bucketing based on a configurable variance
   percentage (default: 15%). Two amounts within 15% of each other land in the
   same bucket.

2. **Filter groups** with fewer than the minimum occurrences (default: 3).

3. **Calculate intervals** between consecutive transactions in each group,
   measured in days.

4. **Check regularity.** If the standard deviation of intervals exceeds the
   maximum variance (default: 5 days), the group is not regular enough to be a
   pattern.

5. **Detect frequency** from average interval:

   | Average Interval | Frequency |
   |---|---|
   | 1-2 days | daily |
   | 5-9 days | weekly |
   | 12-18 days | biweekly |
   | 25-35 days | monthly |
   | 85-100 days | quarterly |
   | 350-380 days | yearly |
   | 55-70 days | custom (bimonthly) |
   | 170-200 days | custom (semiannual) |

6. **Calculate confidence** as a weighted score:

   ```
   confidence = regularity_score * 0.5
              + occurrence_score * 0.3
              + amount_consistency_score * 0.2
   ```

   Where:
   - `regularity_score = 1 - (interval_stddev / avg_interval)`
   - `occurrence_score = min(1, occurrence_count / 12)`
   - `amount_consistency_score = 1 - (amount_stddev / amount_avg)`

   Patterns with confidence below the minimum threshold (default: 0.7) are
   discarded.

7. **Predict next occurrence** by adding the average interval to the last
   occurrence date.

**Configuration:**

```typescript
interface RecurrenceConfig {
  minOccurrences: number        // Default: 3
  maxIntervalVariance: number   // Default: 5 days
  amountVariancePercent: number // Default: 15%
  lookbackDays: number          // Default: 180
  minConfidence: number         // Default: 0.7
}
```

**Pattern lifecycle:**

Detected patterns are stored as `RecurringPattern` records with status
`active`. Each matched transaction creates a `RecurringOccurrence` record
linked to the pattern. The system also tracks missed occurrences -- when a
pattern's `nextExpected` date passes without a matching transaction, a
`missed` occurrence is created and the pattern's miss count is incremented.

This enables proactive alerts: "Your Netflix subscription of $12,990 CLP was
expected on March 5 but has not arrived."

---

## Stack Position

Habilitalo is the integration and observation layer of the Digitalo ecosystem.
It sits at the boundary between external systems and the internal service
mesh.

```
+------------------------------------------------------------------+
|                        EXTERNAL WORLD                             |
|  Banks (Fintoc)  |  CSVs  |  Manual  |  Future: SAT, SII, etc.  |
+--------+---------+--------+----------+---------------------------+
         |              |         |
         v              v         v
+------------------------------------------------------------------+
|                        HABILITALO                                 |
|  Connectors  ->  Pipeline  ->  Ledger  ->  Events  ->  Analysis  |
|  (protocol)      (ingest)     (double     (webhooks)  (anomaly,  |
|                  (process)     entry)                   recurrence)|
+--------+---------------------------------------------------------+
         |
         v
+------------------------------------------------------------------+
|                        SERVICIALO                                 |
|  Service mesh, orchestration, inter-service communication         |
+--------+---------------------------------------------------------+
         |
         v
+------------------------------------------------------------------+
|                        COMPENSALO                                 |
|  Payments, disbursements, collections                             |
+--------+---------------------------------------------------------+
         |
         v
+------------------------------------------------------------------+
|                        COORDINALO                                 |
|  Scheduling, agenda, appointments                                 |
+--------+---------------------------------------------------------+
         |
         v
+------------------------------------------------------------------+
|                        ACUMULALO                                  |
|  Wallet, loyalty, points                                          |
+------------------------------------------------------------------+
```

Data flows inward through Habilitalo: external sources push or are polled,
connectors transform the data, the pipeline processes it, and events notify
downstream services. Compensalo listens for `bank.movement.created` to
reconcile payments. Coordinalo listens for `analysis.recurrence.detected` to
schedule reminders.

The key property is that Habilitalo is _extractable_. It runs as a module
within Balancealo today, but the dependency graph is designed for future
extraction into an independent service at habilitalo.com. All Habilitalo code
lives under `src/lib/habilitalo/` with explicit exports through
`src/lib/habilitalo/index.ts`.

---

## Technical Implementation

### Technology Stack

- **Runtime:** Next.js 15+ on Node.js
- **Language:** TypeScript 5 (strict mode)
- **Database:** PostgreSQL via Supabase
- **ORM:** Prisma with migrations
- **Cryptography:** Node.js built-in `crypto` module (SHA-256, HMAC-SHA256)
- **Multi-tenancy:** Organization-based with row-level data isolation

### File Structure

```
src/lib/habilitalo/                           (~5,500 LOC)
|-- index.ts                                  Central barrel exports
|-- README.md                                 Module documentation
|
|-- protocol/                                 Connector protocol
|   |-- connector.ts                          HabilitaloConnector interface
|   |-- canonical-event.ts                    CanonicalEvent schema + helper
|   |-- registry.ts                           ConnectorRegistry + bridge
|   +-- index.ts                              Protocol exports
|
|-- events/                                   Event system
|   |-- HabilitaloEventEmitter.ts             Core emitter (HMAC, retry)
|   |-- emitBankMovementCreated.ts            Event builder
|   |-- index.ts                              Event exports
|   +-- schemas/                              Event type definitions
|       |-- BankMovementCreated.ts            bank.movement.created
|       |-- AnomalyDetected.ts               analysis.anomaly.detected
|       |-- RecurrenceDetected.ts            analysis.recurrence.detected
|       +-- LedgerEntryCreated.ts            ledger.entry.created
|
|-- feeds/                                    Pipeline
|   |-- index.ts                              Feed exports
|   |-- types.ts                              Core interfaces
|   |-- hash.ts                               SHA-256 dedup engine
|   |
|   |-- pipeline/                             3-stage ETL
|   |   |-- ingest.ts                         Stage 1: source -> RawEntry
|   |   |-- validate.ts                       Entry + balance validation
|   |   +-- processor.ts                      Stage 2: RawEntry -> JournalEntry
|   |
|   |-- sources/                              Connector implementations
|   |   |-- base.ts                           SourceAdapter interface + registry
|   |   |-- fintoc.ts                         Fintoc adapter
|   |   |-- fintoc/habilitalo.connector.json  Fintoc manifest
|   |   |-- csv.ts                            CSV adapter
|   |   |-- csv/habilitalo.connector.json     CSV manifest
|   |   |-- manual.ts                         Manual adapter
|   |   +-- manual/habilitalo.connector.json  Manual manifest
|   |
|   |-- ledger/                               Double-entry accounting
|   |   |-- journal.ts                        CRUD, void, query
|   |   +-- balance.ts                        Balance by account nature
|   |
|   +-- analysis/                             Analysis engines
|       |-- types.ts                           Config + result types
|       |-- anomaly-detector.ts               Z-score anomaly detection
|       |-- recurrence-detector.ts            Pattern detection
|       +-- index.ts                          Analysis exports
|
+-- integrations/                             Third-party orchestrators
    +-- fintoc/
        +-- sync-incremental.ts               24-month chunked sync
```

### API Surface

The module exposes its functionality through `src/app/api/` routes:

- **Webhook management:** `POST/GET /api/habilitalo/webhooks`
- **Delivery inspection:** `GET /api/habilitalo/webhooks/:id/deliveries`
- **Feed operations:** Various routes under `/api/v1/feeds/`
- **Ledger queries:** Routes under `/api/v1/ledger/`
- **Sync triggers:** Cron job routes for Fintoc synchronization

### Multi-Tenancy

Every record in the system belongs to an organization. The `organizationId`
is present on: `DataFeed`, `RawEntry`, `JournalEntry`, `JournalLine`,
`Anomaly`, `RecurringPattern`, `WebhookEndpoint`, `WebhookDelivery`.

The deduplication constraint is scoped to the organization:
`UNIQUE(organizationId, entryHash)`. Two organizations can have identical
transactions without collision.

---

## Governance

### License

The Habilitalo protocol specification and reference implementation will be
released under the MIT License. This means:

- Anyone can build a connector without asking permission.
- Anyone can run the pipeline on their own infrastructure.
- Anyone can fork, modify, and redistribute.
- There is no "premium" tier of the protocol. The full protocol is open.

### Open Registry

We plan to maintain an open registry of community connectors at
habilitalo.com. Submitting a connector requires:

1. A `habilitalo.connector.json` manifest.
2. A transform function that passes validation (input/output type checking).
3. At least one example input/output pair for testing.

The registry is a catalog, not a gate. If your connector meets the interface
requirements, it is listed. We do not editorially curate connectors.

### Self-Hostable

The pipeline and event system are designed to run on any PostgreSQL + Node.js
deployment. There is no dependency on proprietary cloud services beyond what
the host application (Balancealo) uses. The Prisma schema defines all required
tables, and migrations are versioned and portable.

### Connector Quality

We distinguish between three tiers of connector quality, as guidance for
consumers:

- **Foundational:** Built and maintained by the Habilitalo team. Currently: Fintoc, CSV, Manual.
- **Verified:** Community-built, reviewed by the Habilitalo team for correctness and security.
- **Community:** Self-published. Not reviewed. Use at your own discretion.

### Versioning

- The protocol version follows semver. The current version is `0.1`.
- Connector manifests declare the protocol version they target (`"habilitalo": "0.1"`).
- Event schemas are versioned independently (the `version` field on `CanonicalEvent`).
- Breaking changes to the `HabilitaloConnector` interface or `CanonicalEvent` schema will increment the major protocol version.

---

## Roadmap

### v0.1 -- Current (Implemented)

Everything described in this document is implemented and deployed:

- HabilitaloConnector interface and ConnectorContext
- CanonicalEvent schema with createCanonicalEvent helper
- ConnectorRegistry with bridgeAdapterToConnector
- 3-stage pipeline: ingest (hash dedup, validation, staging), process (double-entry ledger), analyze (anomaly + recurrence)
- 3 foundational connectors: Fintoc, CSV, Manual
- Connector manifests (habilitalo.connector.json) for all three
- Event system with HMAC-SHA256, at-least-once delivery, 3 retries with exponential backoff
- 4 event types: bank.movement.created, analysis.anomaly.detected, analysis.recurrence.detected, ledger.entry.created
- Double-entry ledger with balance validation, account nature, void/reversal
- Anomaly detector: Z-score on category stats, new counterparty detection, duplicate suspect detection
- Recurrence detector: interval analysis, frequency classification, confidence scoring, missed occurrence tracking
- Fintoc 24-month incremental sync with resumable chunks
- Webhook management API (create, list, delivery history)
- Multi-tenant isolation with organization-scoped deduplication

### v0.2 -- Next (Planned)

- **Connector UI:** Visual connector management in the Balancealo dashboard. Enable/disable connectors per organization. View connector health and sync status.
- **Additional connectors:** Chilean SII (tax authority) for VAT/invoice data. Bank-specific CSV templates (BancoEstado, BCI, Santander). Manual CSV column mapping wizard.
- **Analysis API endpoints:** REST endpoints to trigger anomaly detection and recurrence detection on demand. Dashboard widgets for anomaly review and pattern management.
- **Test coverage:** Unit tests for hash calculation, adapter transforms, balance validation. Integration tests for the full pipeline (ingest -> process -> event). End-to-end tests for Fintoc sync flow.
- **Connector SDK:** CLI tool to scaffold a new connector (`npx habilitalo init my-connector`). Validation tool to test manifests and transforms.
- **Protocol documentation:** OpenAPI/Swagger specification for the webhook API. JSON Schema for event payloads. Connector manifest JSON Schema.

### v0.3 -- Future (Considered)

- **Connector marketplace:** Public registry at habilitalo.com with search, documentation, and installation.
- **Streaming pipeline:** Replace batch processing with event-driven streaming for real-time analysis.
- **Cross-organization connectors:** Inter-company transfers and reconciliation through shared connectors.
- **AI-powered categorization:** LLM-based transaction categorization as a pipeline stage between process and analyze.
- **Protocol extraction:** Extract `src/lib/habilitalo/` into a standalone npm package and independent service.

---

## Appendix A: Connector Manifest Schema

Every connector ships with a `habilitalo.connector.json` file that declares
its capabilities. This is a static declaration -- no code, no dependencies.

```json
{
  "id": "string",
  "name": "string",
  "version": "string (semver)",
  "category": "string",
  "tags": ["string"],
  "inbound": {
    "transport": "webhook | file | manual | api_poll",
    "events": ["string"]
  },
  "outbound": {
    "events": ["string"]
  },
  "auth": {
    "type": "api_key | oauth2 | link_token | none",
    "requiredFields": ["string"]
  },
  "habilitalo": "string (protocol version)"
}
```

**Field descriptions:**

| Field | Required | Description |
|---|---|---|
| `id` | Yes | Unique connector identifier. Convention: `{source}-v{major}` (e.g., `fintoc-v1`). |
| `name` | Yes | Human-readable connector name. |
| `version` | Yes | Semver version of this connector implementation. |
| `category` | Yes | Broad category: `financial`, `commerce`, `tax`, `custom`. |
| `tags` | No | Searchable tags for registry discovery. |
| `inbound.transport` | Yes | How data enters the connector. |
| `inbound.events` | Yes | Source event types this connector handles. |
| `outbound.events` | Yes | Canonical event types this connector emits. |
| `auth.type` | Yes | Authentication mechanism required by the source. |
| `auth.requiredFields` | No | Credential field names (for UI generation). |
| `habilitalo` | Yes | Protocol version this manifest targets. |

---

## Appendix B: Event Type Reference

### bank.movement.created

Emitted when the pipeline processor creates a JournalEntry from a bank
movement.

```typescript
{
  eventType: 'bank.movement.created',
  eventId: string,       // UUIDv4
  source: 'habilitalo',
  timestamp: string,     // ISO8601
  data: {
    movementId: string,          // RawEntry.id
    accountId: string,           // Balancealo Account.id
    institutionId: string,       // FintocLink institution or 'unknown'
    amount: number,              // Signed amount
    currency: string,            // ISO currency code
    date: string,                // ISO8601 transaction date
    description: string,         // Normalized description
    counterparty: string | null, // Sender or recipient
    type: 'credit' | 'debit',   // Derived from amount sign
    category: string | null,     // Source-provided category
    categoryId: string | null,   // Internal category ID
    categorySource: string | null, // 'fintoc' | 'rule' | 'ai' | 'manual'
    rawFintocId: string,         // Original source ID
    journalEntryId: string       // Posted JournalEntry.id
  }
}
```

### analysis.anomaly.detected

Emitted when the anomaly detector flags a transaction.

```typescript
{
  eventType: 'analysis.anomaly.detected',
  eventId: string,
  source: 'habilitalo',
  timestamp: string,
  data: {
    anomalyId: string,
    journalLineId: string,
    type: 'unusual_amount' | 'unusual_frequency' | 'new_counterparty' | 'duplicate_suspect',
    severity: 'low' | 'medium' | 'high' | 'critical',
    title: string,
    description: string,
    detectedValue: number,
    expectedValue: number | null,
    deviation: number | null,     // Percentage
    context: {
      mean?: number,
      stdDev?: number,
      zScore?: number,
      threshold?: number,
      categoryAvg?: number,
      categoryMax?: number,
      similarTransactionsCount?: number
    }
  }
}
```

### analysis.recurrence.detected

Emitted when the recurrence detector identifies or updates a pattern.

```typescript
{
  eventType: 'analysis.recurrence.detected',
  eventId: string,
  source: 'habilitalo',
  timestamp: string,
  data: {
    patternId: string,
    name: string,
    frequency: 'daily' | 'weekly' | 'biweekly' | 'monthly' | 'quarterly' | 'yearly' | 'custom',
    intervalDays: number,
    expectedAmount: number,
    amountVariance: number,     // Percentage
    currency: string,
    matchPattern: string,       // Normalized description used for matching
    counterparty: string | null,
    categoryId: string | null,
    confidence: number,         // 0.0 to 1.0
    occurrenceCount: number,
    firstOccurrence: string,    // ISO8601
    lastOccurrence: string,     // ISO8601
    nextExpected: string,       // ISO8601
    status: 'active' | 'paused' | 'ended'
  }
}
```

### ledger.entry.created

Emitted when a JournalEntry is posted to the ledger.

```typescript
{
  eventType: 'ledger.entry.created',
  eventId: string,
  source: 'habilitalo',
  timestamp: string,
  data: {
    journalEntryId: string,
    entryNumber: number,
    description: string,
    transactionDate: string,    // ISO8601
    postDate: string,           // ISO8601
    status: 'draft' | 'posted' | 'voided',
    sourceType: string | null,  // 'fintoc' | 'csv_import' | 'manual' | etc.
    sourceRef: string | null,   // Reference to source record
    lines: Array<{
      lineId: string,
      accountId: string,
      entryType: 'debit' | 'credit',
      amount: number,
      currency: string,
      description: string | null,
      counterparty: string | null,
      categoryId: string | null
    }>
  }
}
```

---

## Appendix C: Pipeline Data Flow

A complete trace of a single Fintoc movement through the system:

```
1. FINTOC API
   GET /accounts/{id}/movements?since=2026-01-01&until=2026-01-31
   Returns: FintocMovement {
     id: "mov_abc123",
     amount: -45000,
     currency: "CLP",
     description: "COMPRA SUPERMERCADO LIDER",
     post_date: "2026-01-15",
     recipient_account: { holder_name: "WALMART CHILE SA" }
   }

2. FINTOC ADAPTER (transform)
   FintocMovement -> RawEntryInput {
     transactionDate: 2026-01-15,
     amount: -45000,
     currency: "CLP",
     description: "COMPRA SUPERMERCADO LIDER",
     counterparty: "WALMART CHILE SA",
     sourceId: "mov_abc123",
     rawData: { ...original FintocMovement }
   }

3. INGEST STAGE
   a. Validate: amount != 0, date valid, description present       [PASS]
   b. Normalize: date="2026-01-15", amount="45000.00",
      description="compra supermercado lider"
   c. Hash: SHA-256("2026-01-15|45000.00|{accountId}|
      compra supermercado lider") = "a1b2c3..."
   d. Check existing: SELECT * FROM RawEntry
      WHERE organizationId=? AND entryHash="a1b2c3..."          [NOT FOUND]
   e. Create: INSERT INTO RawEntry (status='pending', ...)

4. PROCESS STAGE
   a. Fetch: SELECT * FROM RawEntry WHERE status='pending'
   b. Mark: UPDATE RawEntry SET status='processing'
   c. Determine type: amount < 0 -> expense
   d. Create JournalEntry:
      Line 1: DEBIT  {accountId} $45,000 CLP "Gasto"
      Line 2: CREDIT {accountId} $45,000 CLP
              "COMPRA SUPERMERCADO LIDER" counterparty="WALMART CHILE SA"
   e. Validate balance: debit(45000) == credit(45000)             [PASS]
   f. INSERT JournalEntry + JournalLines
   g. UPDATE RawEntry SET status='processed', journalEntryId=...

5. EVENT EMISSION (fire-and-forget)
   a. Build BankMovementCreatedEvent payload
   b. Find subscribed WebhookEndpoints
   c. For each endpoint:
      - CREATE WebhookDelivery (at-least-once)
      - HMAC-SHA256 sign payload
      - POST to endpoint.url with signature header
      - If 2xx: mark delivered
      - If 5xx/timeout: retry (1s, 4s, 16s)
      - If 4xx (!429): stop, mark failed

6. ANALYSIS (asynchronous)
   a. Anomaly detector:
      - Category "supermercado" stats: mean=38000, stddev=8000
      - Z-score: |45000 - 38000| / 8000 = 0.875 < 2.0 threshold
      - Result: NOT anomalous
   b. Recurrence detector:
      - Group "compra supermercado lider|bucket_X" now has 4 occurrences
      - Intervals: [7, 7, 8] days -> avg=7.33, stddev=0.47
      - Frequency: weekly
      - Confidence: 0.94 * 0.5 + 0.33 * 0.3 + 0.98 * 0.2 = 0.77
      - Result: RecurringPattern created/updated
```

---

## Closing

Habilitalo exists because we needed it. We were building Balancealo and found
ourselves writing the same integration code for every source -- the same
date parsing, the same amount normalization, the same dedup logic, the same
ledger posting. We extracted the pattern into a protocol because protocols
compose and libraries do not.

The current implementation is 5,500 lines of TypeScript, 24 files, 3
connectors, 4 event types, and 2 analysis engines. It is not a toy. It
processes real bank movements from real Chilean banks for real users. But it is
also not complete. There is no UI. There are not enough connectors. The test
coverage needs work. The analysis engines need API endpoints.

We are publishing this whitepaper and the protocol specification to invite
others to build on it. If you have a data source that does not have a
Habilitalo connector, write one. If the pipeline is missing a feature you
need, propose it. If the protocol design has a flaw, tell us.

The goal is not to build another platform. It is to make integration a solved
problem -- a protocol that anyone can implement, a pipeline that anyone can
run, and a registry that anyone can contribute to.

---

*Habilitalo Protocol v0.1 -- March 2026*
*Franco Danioni -- Digitalo SpA -- habilitalo.com*
*MIT License*
