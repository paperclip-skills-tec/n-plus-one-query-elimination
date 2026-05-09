---
name: n-plus-one-query-elimination
description: Systematic workflow for identifying and eliminating N+1 query patterns in Node.js/Knex/PostgreSQL codebases. Use this skill whenever you encounter per-record DB lookups inside loops, are working on sync/import/bulk-processing code that iterates with individual queries, or a task mentions N+1 queries, query count reduction, or performance issues related to database calls. Also use when refactoring upsert loops, batch insert/update operations, or any code where queries scale O(n) with record count. Invoke even when the fix seems obvious — the decision framework prevents choosing the wrong batching strategy (e.g., upsert vs. lookup map vs. CTE).
---

# N+1 Query Elimination

N+1 queries happen when code issues one DB call per record in a collection — O(n) queries instead of O(1) or O(batch). This skill guides you from identification through implementation to verification.

## Step 1: Identify the Pattern

Locate loops (or functional equivalents like `map`, `forEach`, `reduce`) that contain DB calls. Classify the pattern:

- **Read-N+1**: a `SELECT` per record (e.g., fetching a related entity for each item in a list)
- **Write-N+1 (uniform)**: an `INSERT` or `UPDATE` per record with no per-record conditional branching (pure upsert)
- **Write-N+1 (conditional)**: INSERT or UPDATE where the logic differs per record depending on fetched state
- **Mixed**: read then write per record (fetch-then-upsert per item)

Count the queries: if you have a loop over N records and each iteration issues ≥ 1 query, that's an N+1.

## Step 2: Choose the Right Strategy

| Pattern | Strategy | Why |
|---|---|---|
| Read-N+1 | Batch SELECT + lookup map | One `WHERE id IN (...)` fetches all rows; a `Map<key, row>` replaces per-record lookups in O(1) |
| Write-N+1 (uniform) | `INSERT ... ON CONFLICT DO UPDATE` (upsert) | Single statement handles both insert and update paths; PostgreSQL optimizes internally |
| Write-N+1 (conditional) | CTE or staged temp table | When per-record decisions depend on current DB state, batch-read first, compute decisions in-process, then batch-write |
| Mixed read+write | Batch read → lookup map → batch write | Never interleave reads and writes per record; separate the two passes |

Choose based on what the loop is actually doing, not on what seems familiar. A read-N+1 does not benefit from upsert; a conditional write-N+1 does not simplify to a plain upsert.

## Step 3: Implement

### Batch SELECT with lookup map (Read-N+1)

```ts
// Before: per-record SELECT inside loop
for (const record of records) {
  const related = await knex('related').where('id', record.relatedId).first();
  // use related...
}

// After: one query, O(1) lookups
const ids = records.map(r => r.relatedId);
const relatedRows = await knex('related').whereIn('id', ids);
const relatedMap = new Map(relatedRows.map(r => [r.id, r]));

for (const record of records) {
  const related = relatedMap.get(record.relatedId);
  // use related...
}
```

### Upsert (Write-N+1 uniform)

Use `knex.raw()` with parameterized bindings — never string-interpolate values:

```ts
// Before: INSERT or UPDATE per record
for (const record of records) {
  await knex('table').insert(record).onConflict('key').merge();
}

// After: single upsert
await knex('table')
  .insert(records)
  .onConflict('key')
  .merge();

// Or via knex.raw for complex upserts:
await knex.raw(
  `INSERT INTO table (col_a, col_b) VALUES ?
   ON CONFLICT (key) DO UPDATE SET col_b = EXCLUDED.col_b`,
  [records.map(r => [r.colA, r.colB])]
);
```

### CTE approach (Write-N+1 conditional)

```ts
// Batch-read current state, decide in-process, batch-write
const existing = await knex('table').whereIn('key', keys);
const existingMap = new Map(existing.map(r => [r.key, r]));

const toInsert = [];
const toUpdate = [];
for (const record of records) {
  if (existingMap.has(record.key)) {
    toUpdate.push({ ...record, id: existingMap.get(record.key).id });
  } else {
    toInsert.push(record);
  }
}

if (toInsert.length) await knex('table').insert(toInsert);
if (toUpdate.length) {
  // batch update — use a single UPDATE ... FROM (VALUES ...) WHERE pattern
  // or chunked knex updates if update logic is simple
}
```

## Step 4: Preserve Semantics

Before finalizing, verify:

- **Return values**: if the original loop accumulated results (e.g., `insertedIds`), ensure the batch implementation returns the same shape
- **Error handling**: if the original loop had per-record try/catch, decide whether errors should be per-record (difficult with batch) or abort-on-first (usually acceptable). Document the choice in a comment if it changes behavior
- **Side effects**: if the loop triggered events, notifications, or cache invalidations per record, ensure those still fire after the batch
- **Transaction scope**: if the original loop ran in a transaction, the batch operation should too

## Step 5: Verify Query Count

Add or update tests that assert the new code issues O(1) or O(batch) queries instead of O(n):

```ts
// Using knex query listener
let queryCount = 0;
knex.on('query', () => queryCount++);

await batchOperation(records);

knex.removeAllListeners('query');
expect(queryCount).toBeLessThanOrEqual(2); // e.g., 1 read + 1 write
```

If query counting is not feasible in the test environment, at minimum add a comment that explains the expected query count and why it is bounded.

## Step 6: Handle Batch Size Limits

PostgreSQL has a hard parameter limit (~65535 bound parameters per query). Large datasets must be chunked:

```ts
const CHUNK_SIZE = 1000; // conservative; adjust based on column count
for (let i = 0; i < records.length; i += CHUNK_SIZE) {
  const chunk = records.slice(i, i + CHUNK_SIZE);
  await knex('table').insert(chunk).onConflict('key').merge();
}
```

Document the chunk size and the reason in a comment when it is not obvious. For reads, `WHERE id IN (...)` with thousands of IDs is safe but may be slow — consider a join against a temp table or CTE for very large sets.

## Common Pitfalls

- **Forgetting the lookup map step**: replacing a SELECT loop with a single SELECT is only half the fix; you still need the map so the subsequent logic stays O(1) per record.
- **Using upsert for conditional writes**: if the per-record decision requires reading DB state, a plain `ON CONFLICT DO UPDATE` won't capture the conditional logic — use the staged approach instead.
- **Not chunking**: a single upsert of 100k records will hit PostgreSQL parameter limits or memory pressure.
- **Ignoring transaction boundaries**: batch inserts without a transaction can leave partial state on error.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
