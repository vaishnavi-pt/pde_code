# Data Quality in PDE

We reviewed the comprehensive list of DQ checks suggested by Databricks at [Data Quality Management With Databricks](https://www.databricks.com/discover/pages/data-quality-management) and chose the techniques that fit PDE.

Data quality is grouped into six dimensions: consistency, accuracy, validity, completeness, timeliness, and uniqueness. A value is trustworthy when it agrees with related data, matches the facts, follows the expected format, is present, is current, and appears once. The sections below are those dimensions. Each one lists only the checks selected for PDE, the Databricks feature that implements them, and where they run in a PDE own catalogs.

Bronze stores data close to the source and is not shared. Silver is the layer that applies constraints and deduplication. Gold is the shape consumers query. `RESTORE` and `VACUUM` repair a table after a bad commit. They are not steps in a normal load and will be scheduled as separate workflows.

## Selected checks

| Dimension | Check | Feature | Where it runs |
| --- | --- | --- | --- |
| Consistency | Concurrent reads and writes see one committed snapshot | Delta optimistic concurrency control | Every Delta table |
| Accuracy | Reject a row that is null or fails a Boolean rule | `NOT NULL`, `CHECK` | Silver and gold |
| Accuracy | Undo a commit that wrote inaccurate data | `RESTORE` (time travel) | The affected Delta table |
| Accuracy | Drop table versions that still hold inaccurate data | `VACUUM` | The affected Delta table, after the correction |
| Validity | Reject a write whose schema does not match the table | Schema enforcement (built in) | Every Delta table |
| Validity | Add new source columns on an approved write | `mergeSchema` | Bronze, for that write |
| Validity | Replace the schema when a breaking change is approved | `overwriteSchema` | The table whose contract changed |
| Completeness | A write lands in full, or not at all | Delta ACID atomicity | Every commit, and any batch of tables that must publish together |
| Completeness | Record a filename a record came from |  `input_file_name()` | Bronze load |
| Completeness | Describe tables, columns, and tags in the catalog | `COMMENT`, `TBLPROPERTIES` | Table definitions |
| Timeliness | Several pipelines commit to one table without blocking each other | Same concurrency control as consistency | Ingest jobs that share a table |
| Uniqueness | Collapse exact duplicate rows | `distinct()`, `dropDuplicates` | Bronze to silver |

## Consistency

Consistency means two readers of the same fact get the same committed data, including while other jobs are writing.

### Optimistic concurrency control

Delta commits each write as a new table version. A write reads the current version, stages its files, then validates that no other commit changed the same data. When validation succeeds, the write becomes the next version. When another commit won that race, this write fails and leaves the table on the last good version. A reader always sees one complete version.

PDE relies on this for every table in bronze, silver, and gold. Ingest jobs that load different sources into one table run together. A conflicting write is retried by the job. It is not merged by hand inside the table.

## Accuracy

Accuracy means a stored value still means what the source sent, and a value that fails a hard rule never becomes the current table.

### Constrain and validate

Delta constraints reject bad rows at write time. A violation fails the whole transaction and rolls back.

| Constraint | Rule | PDE use |
| --- | --- | --- |
| `NOT NULL` | The column cannot be null | Business key, and any other column the silver or gold contract marks as mandatory |
| `CHECK` | A Boolean expression is true for every input row | Ranges and cross-column rules, such as an end on or after a start, or an amount that must be non-negative |

Add the constraint before the first load of that table. Adding it later succeeds only when every existing row already satisfies it.

```sql
ALTER TABLE `ph-rd-pde-dev`.silver.example
  ALTER COLUMN business_key SET NOT NULL;

ALTER TABLE `ph-rd-pde-dev`.silver.example
  ADD CONSTRAINT amount_non_negative CHECK (amount >= 0);
```

`CHECK` is a row expression. It does not look up other tables. A rule that needs another table is a separate query in the silver job, with the same outcome: the write does not commit.

### Roll back an inaccurate update

`RESTORE` returns a table to an earlier version or timestamp. Use it when a commit wrote inaccurate rows, or removed rows that should have stayed. Readers then see that earlier version. Versions after the restored one remain in history until vacuum removes them.

```sql
RESTORE TABLE `ph-rd-pde-dev`.silver.example TO VERSION AS OF 12;
```

PDE treats restore as an operational repair of one table, named by version or by the timestamp of the last known good commit. A scheduled pipeline does not restore as part of a successful load.

### Remove inaccurate history

`VACUUM` deletes data files that the current table no longer references, once they are older than the retention window. After those files are gone, time travel cannot return to the versions that needed them. That is how PDE removes a corrected-but-wrong history, and how deleted sensitive values stop being readable through an old version.

Databricks also vacuums files that have aged past the table retention period. A manual `VACUUM` is for the case where the bad version must disappear sooner. Vacuuming inside the retention window needs an explicit override, and it permanently removes the ability to restore those versions. The cluster cache can still hold the old data until the cluster restarts.

The retention window in PDE is 365 days. VACUUM may remove table versions older than that, so an earlier state of an assay cannot be restored once it is past the window. The current data in the table remains. PDE will run VACUUM once a week. Each run will delete data files older than the 365-day retention window and free up underlying storage.

## Validity

Validity means incoming data matches the table schema: column names and types.

### Schema enforcement

Schema enforcement is on for every Delta table. A write is rejected when a column type conflicts with the table, or when the data contains a column the table does not have. Column names are compared without regard to case. A column that exists on the table and is missing from the data is written as null, unless a `NOT NULL` constraint forbids that.

Bronze, silver, and gold all keep enforcement on. A file or query that drifts from the table fails the job instead of silently widening the table.

### Schema evolution

Schema evolution adds columns that appear in the incoming data. PDE turns it on for a single write with `mergeSchema`, and only on the bronze table that is allowed to gain those columns. The write can also widen a null type to a concrete type, and widen byte toward short or int. It does not drop a column, change an existing type beyond those widenings, or rename a column.

```python
df.write.format("delta") \
  .mode("append") \
  .option("mergeSchema", "true") \
  .saveAsTable("`ph-rd-pde-dev`.bronze.example")
```

The session setting `spark.databricks.delta.schema.autoMerge.enabled` applies that same evolution to every Delta write in the session. PDE leaves it off. Evolution stays on the one bronze write that has an approved additive change.

Silver and gold keep the published schema. A new bronze column reaches them only after the contract for that table adds it.

### Schema overwrite

`overwriteSchema` replaces the table schema as part of an overwrite: drop a column, change a type, or align names that differ only by case. The overwrite is atomic. A reader still sees the previous version while it runs, and that previous version stays available through time travel until vacuum.

```python
df.write.format("delta") \
  .mode("overwrite") \
  .option("overwriteSchema", "true") \
  .saveAsTable("`ph-rd-pde-dev`.silver.example")
```

PDE uses this when a breaking schema change has already been accepted for that table. A routine load does not set the option. Replacing the schema by deleting the table directory is avoided: the overwrite is faster, atomic, and recoverable.

## Completeness

Completeness means the rows for a commit are all present, each row can be traced to the load that wrote it, and the catalog describes what the table holds.

### Atomicity

A Delta commit succeeds entirely or rolls back. PDE writes a logical batch in one commit per table. Splitting one table’s batch across separate statements would commit some statements and leave others unwritten.

Tables that are checked together publish in one step. If any blocking check for that set fails, none of those tables move to the new version.

### Metadata management

Catalog metadata is part of completeness. A table that exists without a description cannot be discovered or audited as a known dataset.

| Mechanism | What PDE stores |
| --- | --- |
| `COMMENT` on the table | TBD by the PDE dev team |
| `COMMENT` on a column | N/A |
| `TBLPROPERTIES` | Tags such as source system, layer (`bronze`, `silver`, `gold`), and the feed name, but the PDE dev team may adjust it according to their needs |

```sql
COMMENT ON TABLE `ph-rd-pde-dev`.silver.example
  IS 'Validated experiment rows, one row per business key.';

ALTER TABLE `ph-rd-pde-dev`.silver.example
  SET TBLPROPERTIES ('layer' = 'silver', 'source' = 'evaluwhale');
```

Unity Catalog is where these comments and tags are read. Names are not distinguished by case.

## Timeliness

Timeliness means a change in the source shows up in the table within the lag agreed for that feed.

Delta concurrency is the mechanism PDE uses. Several ingest jobs commit to the same table, each as its own version, so one source does not wait behind another source’s lock. The gap between a file landing and a committed bronze version is the job schedule plus the write. A conflicting commit fails and is retried, which keeps the table on a consistent version while the retry catches up.

Freshness targets per feed (how old `extract_as_of` or the newest file may be) are agreed on the feed. This page does not add a streaming path.

## Uniqueness

Uniqueness means one business entity is one silver row. Duplicate rows are removed on the way from bronze to silver. The business key is `_bk`.

### distinct() and dropDuplicates()

`distinct()` keeps one row from each group of identical rows, comparing every column. `dropDuplicates()` does the same, and accepts a column list when only part of the row defines a duplicate.

```python
deduped = bronze.dropDuplicates(["_bk"])
```

PDE uses these when a replay or an overlapping file produced exact copies, and any one of the copies is acceptable. They do not choose a newest version. The row that remains is whichever copy Spark keeps.

## How a failure is handled

| Check | Result |
| --- | --- |
| `NOT NULL` or `CHECK` violated | The write rolls back. The table stays on the previous version. |
| Schema enforcement rejected the write | The job fails. The table schema stays as it was. |
| Optimistic concurrency conflict | The write fails. The job retries against the new version. |
| `distinct`, `dropDuplicates`, or the ranking window | Duplicates are removed before the silver commit. The silver commit still has to pass constraints and schema enforcement. |
| Inaccurate data already committed | `RESTORE` returns the table to the last good version. `VACUUM` removes the bad version after it is no longer required. |

A failed bronze or silver job is rerun from the landing files or the last good version. Restore is the tool when the bad data is already the current version and a rerun would write it again.
 