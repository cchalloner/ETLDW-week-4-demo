# SCD_Demo — Slowly Changing Dimensions Demo

This repo contains the demo scripts referenced in Week 4, Videos 5 and 6, showing
how Type 1 and Type 2 slowly changing dimensions are implemented in SQL Server.

There are two scripts:

1. **Main demo script** — builds the database, source table, and both
   dimension tables, and walks through Type 1 and Type 2 changes step by step
2. **MERGE-based Type 1 upsert** — a more compact, alternative way to write
   the same Type 1 logic shown in the main script, using SQL Server's `MERGE`
   statement instead of separate `INSERT` and `UPDATE` statements

The main script is self-contained and safe to re-run — it drops and recreates
its own `SCD_Demo` database each time, so it won't affect your Chinook or Wide
World Importers work.

## What this demonstrates

- The difference between a source (OLTP-style) table and two dimensional
  versions of the same data: one Type 1, one Type 2
- How to "upsert" new and changed records into a Type 1 dimension, where
  updates simply overwrite the existing row and no history is kept — shown
  two different ways (separate insert/update statements, and a single `MERGE`)
- How to detect and process changes into a Type 2 dimension using a stored
  procedure, preserving history with an effective date, an end date, and a
  current-record flag
- How to query a Type 2 table to see full change history for a customer, or
  reconstruct what their record looked like as of a specific past date

## What's in the main script

1. **Fresh database** — drops and recreates `SCD_Demo`
2. **Source table** — `Customers_OLTP`, seeded with three customers as of
   2010-01-15
3. **Two dimension tables**:
   - `DimCustomer_T1` — Type 1 (surrogate key, natural key, current values only)
   - `DimCustomer_T2` — same columns, plus `RowEffectiveDate`, `RowEndDate`,
     and `IsCurrent`, with a filtered unique index ensuring only one current
     row per customer
4. **Initial load** — both dimension tables loaded from the source table
5. **Type 1 walkthrough** — four simulated changes (a name change, an address
   change, an unrelated customer's ZIP correction, a hyphenated name change),
   each followed by the same insert/update ("upsert") logic, overwriting the
   dimension row in place
6. **Type 2 setup** — resets the source and `DimCustomer_T2` back to the
   2010-01-15 baseline, then creates a stored procedure,
   `SCD2_Process_Once`, that detects changed rows via a CTE, closes out the
   old row (sets `RowEndDate` and `IsCurrent = 0`), and inserts a new current
   row
7. **Type 2 walkthrough** — the same four events as the Type 1 section, each
   applied to the source table and then processed with
   `EXEC dbo.SCD2_Process_Once`
8. **History and as-of queries** — shows full change history per customer,
   and a parameterized "as of" query reconstructing a customer's record as of
   a given date (`2016-12-31` in the example)

## What's in the MERGE script

This is a standalone alternative to the insert/update pattern in step 5 of the
main script above. It performs the exact same Type 1 logic — update changed
attributes, insert brand-new customers — but as a single `MERGE` statement
instead of two separate statements. It's wrapped in an explicit transaction
with `SET XACT_ABORT ON` and uses `WITH (HOLDLOCK)` on the target table, which
is a defensive pattern that guards against race conditions if this were ever
run concurrently (not something you'll hit in this demo, but good practice to
be aware of if you use `MERGE` in real ETL work).

This script is worth reviewing side-by-side with the main script's Type 1
section: same result, two different SQL patterns for getting there. `MERGE`
is more compact, but some data engineers avoid it in production SQL Server
code due to a handful of well-documented edge-case bugs Microsoft has had with
concurrent modifications over the years — worth knowing that history if you
end up using `MERGE` outside of this class.

## How to use this

**Note:** This demo is for understanding the concepts only — you are not
required to run or modify these scripts for the Week 4 lab. The lab asks for a
star schema diagram and written SCD justification, not SQL implementation.

If you want to explore it:

1. Download the main script and open it in SQL Server Management Studio
2. Run it section by section (the script is numbered `[0]` through `[8]`)
   rather than all at once, so you can see each step's effect before moving on
3. After each change event, query `DimCustomer_T1` or `DimCustomer_T2` to see
   how each type handled the update differently
4. Pay particular attention to section `[6]`, the `SCD2_Process_Once`
   procedure — this is the core logic behind how a Type 2 dimension gets
   maintained in an automated load process
5. Once you've run the main script, try the MERGE script against the same
   `DimCustomer_T1` table (after making another change to `Customers_OLTP`) to
   see the same result produced a different way

## Why this matters for your lab

These scripts show *how* Type 1 and Type 2 changes work under the hood — a
Type 1 overwrite versus a Type 2 row-versioned history. When you're deciding
which SCD type fits Artist, Customer, Genre, and Date in your Chinook star
schema, it may help to think through what an update to each of those
dimensions would actually look like using the patterns shown here.
