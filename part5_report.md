# Part 5- Assignment 1: Final Summary Report
**Author:** Ferran Dalmau Codina

---

## 1. MongoDB vs Parquet

**Trade-offs for analytical queries on tabular data:**

MongoDB stores each trip as a BSON document (row-oriented). For analytical queries that aggregate millions of rows, this creates significant overhead:

- **I/O amplification:** To compute, for example, average `tip_amount` by `payment_type`, MongoDB must deserialize all 19 fields of every document, even the 17 it does not need. Parquet reads only the two requested column chunks, skipping everything else entirely.
- **Compression inefficiency:** MongoDB's row store cannot exploit the value locality that makes Parquet's dictionary and run-length encodings so effective. The 100 K-row JSON export used for `mongoimport` was **42 MB** — for the full 9.5 M rows that extrapolates to ~4,017 MB, versus **160 MB** for the original Parquet files (a **25× size difference**).
- **No predicate pushdown:** Parquet embeds min/max statistics per row group, so a query engine can skip entire row groups without reading them. MongoDB has no equivalent for sequential scans.
- **Query engine:** DuckDB compiles SQL to vectorised, SIMD-optimised operations processing thousands of values per CPU instruction. MongoDB's aggregation pipeline operates document-by-document.

| Dimension | MongoDB | Parquet + DuckDB |
|---|---|---|
| Storage (9.5 M rows) | ~4,017 MB (JSON est.) | 160 MB |
| Read paradigm | Row-oriented | Columnar |
| Compression | Limited | Highly effective |
| Predicate pushdown | No | Yes (row-group stats) |
| Schema flexibility | Flexible / nested | Fixed, typed |

**When to prefer MongoDB:** When the schema is irregular or nested (e.g., variable fields per document), when the workload is transactional (many point lookups, inserts, or updates by `_id`), or when data is genuinely semi-structured and does not fit a fixed tabular schema.

**When to prefer Parquet + DuckDB:** For large-scale analytical workloads with fixed schemas — aggregations, scans, and filters over millions of rows — where storage efficiency and query speed are critical.

---

## 2. Format Choice: CSV vs Parquet

| | CSV | Parquet |
|---|---|---|
| File size (9.5 M rows) | **1,032.7 MB** | **160.4 MB** (6.4× smaller) |
| Read time (full dataset) | **1.22 s** | **0.29 s** (4.2× faster) |
| Schema encoded | No | Yes |
| Column pruning | No | Yes |
| Human-readable | Yes | No |

**Use CSV when:** The downstream consumer cannot read Parquet — for example, exporting a result to share with a business analyst who will open it in Excel, or feeding data to a legacy system that only accepts flat text files.

**Use Parquet when:** The data will be processed programmatically at scale. For example, storing the NYC taxi dataset for monthly analytical reports: Parquet is 6× smaller on disk and 4× faster to read, with no loss of type fidelity (timestamps, integers, and floats are stored natively rather than parsed from strings).

---

## 3. Compression

Measured results writing the full 9,554,778-row combined table as a single Parquet file:

| Compression | Size (MB) | Write time (s) | Read time (s) |
|---|---|---|---|
| snappy | 195.3 | 2.21 | 0.37 |
| gzip | 148.7 | 70.29 | 0.31 |
| zstd | 160.2 | 2.19 | 0.29 |
| none | 253.1 | 1.69 | 0.31 |

**(a) Archival dataset, rarely read → recommend `gzip`**
Gzip produces the smallest files (148.7 MB, 41% smaller than uncompressed) at the cost of a much longer write time (70 s vs ~2 s). For a dataset written once and read infrequently, the 70 s write penalty is paid only once, and the smaller file reduces long-term storage costs.

**(b) Frequently queried dataset → recommend `zstd`**
Zstd achieves nearly the same size as gzip (160.2 MB) but writes in 2.19 s and reads in 0.29 s — matching uncompressed read performance. It gives the best size-to-speed ratio for interactive workloads. Snappy is a reasonable alternative but produces files 24% larger than zstd for no significant speed advantage in this dataset.

---

## 4. Partitioning

**What makes a good partition column:**

A good partition column should:
1. **Have low-to-medium cardinality** that maps naturally to typical query filters. Month (3–12 distinct values per year) is ideal — queries like "show me January data" skip 2/3 of files without reading a single row.
2. **Produce balanced partitions.** The month-partitioned output was well-balanced: month=1 had 2,964,621 rows (62.8 MB), month=2 had 3,007,533 rows (63.0 MB), and month=3 had 3,582,607 rows (75.5 MB). Each partition file is in the optimal range (50–128 MB).
3. **Be frequently used in filter predicates.** If analysts always query by month, partitioning by month means partition pruning kicks in on every real query.

**Partition pruning benefit (measured):**

| Method | Rows returned | Time |
|---|---|---|
| Partitioned dataset (month=1 only) | 2,964,621 | **0.06 s** |
| Full file scan + in-memory filter | 2,964,621 | **0.34 s** |
| **Speedup** | | **5.9×** |

**Risks of over-partitioning (day-level experiment):**

Partitioning by year/month/day created **96 directories and 96 Parquet files**, with individual files averaging only **2.12 MB** (min 0.01 MB, max 2.88 MB). This is far below the recommended 128 MB–1 GB target. The consequences are:

- **Metadata overhead dominates:** Opening 96 files, reading their footers, and planning the scan takes more time than the actual data read for small queries.
- **Poor compression:** Tiny files have fewer repeated values to exploit with dictionary encoding.
- **Scheduler thrashing:** Distributed engines (Spark, Trino) spin up one task per file — 96 tasks doing microseconds of work each wastes cluster resources on overhead.

The conclusion is clear: partition at a granularity that keeps individual files above ~128 MB. For 3 months of NYC taxi data, month is ideal; day creates the small-file problem.

---

## 5. One Thing I Learned

The most surprising finding was **how extreme column pruning's speedup is in practice**. Reading only the two columns needed for "average tip by payment type" (`payment_type` and `tip_amount`) took **0.03 s**, while reading all 19 columns took **0.36 s** — a **12.6× speedup** just by specifying columns.

This is more dramatic than I expected because each Parquet column chunk is stored contiguously and independently compressed. Selecting 2 out of 19 columns means the engine physically reads roughly 2/19 (~10%) of the file bytes, with no wasted I/O. By contrast, in a row-oriented format like CSV or MongoDB, every field of every row must be read and parsed even if only one value per row is needed. This makes the case for Parquet more compelling than just the compression ratio numbers suggest — the combination of columnar layout and column pruning is what truly separates it from row-based formats for analytical workloads.
