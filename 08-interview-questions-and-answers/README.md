# 08. Walmart–Databricks architecture: interview questions and answers

These answers follow the architecture described in this repo:

**Walmart API → Databricks ingestion code → Bronze Delta → Silver transformations and quality checks.**

There is no separate data-landing layer in this design. The private key is stored in a Databricks Volume. Recovery, tracking and duplicate-prevention mechanisms below are **proposed design answers until confirmed in the existing code**.

Statements about Walmart come from the Walmart Data Ventures documentation, and statements about Databricks and Delta come from the Databricks and Delta Lake documentation, as noted in my source notes. HTTP method definitions follow the RFC, and the key-pair explanation follows NIST. I have not re-checked them against the live documentation.

Related folders: [06 final architecture](../06-final-architecture-bronze-to-silver/README.md), [07 multiple feeds](../07-multiple-feeds-in-one-pipeline/README.md), [05 recovering from a gap](../05-recovering-from-a-pipeline-gap/README.md).

## Contents

- [Part 1: Explain the overall architecture](#part-1-explain-the-overall-architecture) (questions 1 to 4)
- [Part 2: Authentication and security](#part-2-authentication-and-security) (5 to 7)
- [Part 3: Historical loading, incrementality and recovery](#part-3-historical-loading-incrementality-and-recovery) (8 to 14)
- [Part 4: Bronze, Silver and data correctness](#part-4-bronze-silver-and-data-correctness) (15 to 19)
- [Part 5: Architecture choices and trade-offs](#part-5-architecture-choices-and-trade-offs) (20 to 26)

## Part 1: Explain the overall architecture

### 1. Can you explain your Walmart integration end to end?

**Interview answer:**

> "We use a scheduled Databricks job to collect approved Walmart feeds, such as Store Sales and Omni Sales. The job makes authenticated requests to Walmart. Walmart returns signed download links, and our code retrieves the associated files."
>
> "In the architecture I am describing, the ingestion code writes the source records directly into the corresponding Bronze Delta tables. Downstream Silver processing cleans and validates those records and applies the source's change instructions. My implementation scope is Bronze ingestion."

The flow you should be able to explain is:

```text
Databricks job starts
        |
Determine which feed deliveries are required
        |
Authenticate and request each feed
        |
Retrieve and read the downloaded files
        |
Write source records into Bronze
        |
Transform and validate completed deliveries
        |
Maintain Silver tables
```

**Example:** Store Sales records go to `bronze.store_sales`; Omni Sales records go to `bronze.omni_sales`. These are illustrative table names, not confirmed names from your workspace.

### 2. How can one pipeline ingest multiple feeds when each endpoint returns one feed?

**Interview answer:**

> "A pipeline run can make multiple API requests. The restriction is that each feed-specific request targets its selected feed, not that a pipeline can process only one feed."

Walmart exposes separate incremental endpoint paths for Store Sales and Omni Sales.

| Feed | Endpoint path | Illustrative target |
| --- | --- | --- |
| Store Sales | `/bulkfeeds/incremental/store-sales2` | `bronze.store_sales` |
| Omni Sales | `/bulkfeeds/incremental/omni-sales2` | `bronze.omni_sales` |

**Example:** One job makes a Store Sales request and an Omni Sales request, retrieves both responses, and routes each dataset to its table.

The requests can run sequentially or through independent parallel tasks. Databricks supports task dependencies and parallel execution within a job.

**The distinction:** one job run does not mean one HTTP request.

### 3. What are GET and POST, and which Walmart operations use them?

**Interview answer:**

> "GET requests a resource. POST submits information for the server to process. POST does not automatically mean inserting business data into a database."

For the sales feeds we discussed:

| Requirement | Operation | Method |
| --- | --- | --- |
| Latest available delivery | Get Latest Data | GET |
| Delivery for a specified feed date | Get Data By FeedDate | POST |
| Currently available historical files | Get Available History | GET |
| Named historical files | Get History By FileName | POST |

These mappings are specific to Walmart's documented operations.

**Example:** Sending `feedDate=2026-10-05` through the incremental POST endpoint asks for that delivery. It does not upload your sales records to Walmart.

**Remember:** "GET means latest" and "POST means a date" are not universal API rules.

### 4. Are all Walmart feeds incremental?

**Interview answer:**

> "No. The loading strategy depends on the feed contract. Walmart lists Store Sales and Omni Sales as incremental feeds, while feeds such as Product Dimensions and Store Dimensions are snapshots."

**Example:** For an incremental sales feed, we process newly delivered changes. For a complete product snapshot, we receive a representation of the dataset at that snapshot's point in time.

In our design, Bronze could preserve successive snapshot versions. Silver could maintain the latest validated snapshot or a historical representation, depending on business requirements.

**Architect-level point:** I would not apply identical append-and-update rules to every feed. In particular, I would not treat a record missing from an incremental delivery as a deletion.

## Part 2: Authentication and security

### 5. How does Walmart authentication work, and why do you share only the public key?

**Interview answer:**

> "Our technical team generates a public/private key pair. We submit only the public key during Walmart onboarding. After processing the request, Walmart supplies a Consumer ID and key version."
>
> "The private key creates signatures. The matching public key allows verification without revealing the private key."

In your described setup, Subrat generated the keys on EC2, and the private key is now stored in a Databricks Volume.

**Example:** Databricks generates a signature using the private key. Walmart verifies that signature using the registered public key.

**Important distinction:** EC2 is where the keys were generated; Walmart is where the public key was registered. Their purpose here is API signing, not logging into EC2 through SSH.

### 6. When is the signature generated, and what does its three-minute validity mean?

**Interview answer:**

> "The running code generates a fresh signature and matching timestamp immediately before an API call. Walmart recommends generating a signature for every call because it expires after three minutes."

The request includes:

```text
WM_CONSUMER.ID
WM_SEC.KEY_VERSION
WM_CONSUMER.INTIMESTAMP
WM_SEC.AUTH_SIGNATURE
```

These are Walmart's mandatory authentication headers.

**Example:** A signature generated at 6:00 PM should not be used for a new request at 6:04 PM. Generate another signature using the existing private key.

The three minutes applies to authentication, not the entire pipeline.

Also distinguish this from the download link: Walmart documents a separate ten-minute window for starting downloads. Downloads already in progress are not stopped simply because that window expires.

### 7. What is the difference between authentication and authorization in this architecture?

**Interview answer:**

> "Authentication establishes who is making the request. Authorization determines whether that identity is permitted to perform the requested action."

There are separate access checks in this architecture.

- Inside Databricks, the job's execution identity needs permission to read the private-key Volume. Unity Catalog requires the relevant catalog and schema access and `READ VOLUME`.
- At Walmart, the supplied identity and signature must be accepted, and the requested API access must be permitted. Walmart documents failures for invalid signatures, missing headers and unauthorized access.

**Example:** A job could successfully read the private key but still fail to access a Walmart endpoint. Conversely, approved Walmart credentials are useless to a job that cannot read them.

For the proposed design, I would restrict key access and prevent private keys, live signatures and usable signed URLs from appearing in logs.

## Part 3: Historical loading, incrementality and recovery

### 8. How do initial and incremental loads fit together?

**Interview answer:**

> "We first establish the historical baseline. After confirming its coverage and the correct continuation point, we process incremental deliveries. Walmart's onboarding guidance requires consuming history before incremental feeds."

I would parameterize one reusable ingestion implementation:

| Proposed parameter | Purpose |
| --- | --- |
| `load_mode=initial` | Establish history and catch up to the required point. |
| `load_mode=incremental` | Process new and previously missed available deliveries. |
| `load_mode=recovery` | Recover an explicitly requested range. |

**Example:** After verifying the baseline and source cutover, the first run loads the required history and outstanding deliveries. Subsequent runs do not reload that entire history.

**Important:** a parameter does not implement incremental loading by itself. Our code must identify what has already succeeded and what remains outstanding.

### 9. The pipeline stopped for two days. How would you recover the missing data?

**Interview answer:**

> "I would identify the missing feed deliveries and retrieve them using Get Data By FeedDate. I would not assume that requesting the latest delivery automatically recovers everything missed."

The latest endpoint retrieves the latest available delivery, while the feed-date endpoint retrieves available files for a specified date.

**Example:**

| Feed date | Bronze status |
| --- | --- |
| 4 October | Complete |
| 5 October | Missing |
| 6 October | Missing |
| 7 October | Complete |

The recovery needs 5 and 6 October, not another copy of 7 October.

I would build this catch-up logic into the daily ingestion process. A separate scheduled recovery pipeline is not inherently necessary.

**Key distinction:** a missed incremental delivery does not become a History API request merely because its date is in the past.

### 10. Is the feed date the same as the business date?

**Interview answer:**

> "No. The feed date identifies the requested delivery. The business date identifies when the underlying business activity occurred."

Walmart's incremental request accepts `feedDate`, while Store Sales records include `bus_dt`.

**Example:**

```text
Feed date:       7 October
Business date:   5 October
Meaning:         A delivery on 7 October contains
                 sales information relating to 5 October.
```

For our design, ingestion tracking uses delivery information. A business request for "sales on 5 October" filters the business date in the appropriate validated table.

**Why it matters:** using the maximum business date as the ingestion checkpoint could miss later corrections to older business dates.

### 11. What would you store in the ingestion tracking table?

**Interview answer:**

> "I would record progress separately for each feed, delivery and file. A single job-success flag is not enough to explain partial completion."

Our proposed tracking record would include the feed, feed date, stable file identity, source version or fingerprint where available, processing status, row count, run identifier and error details.

**Example:**

```text
Feed:             Store Sales
Feed date:        2026-10-05
File identity:    file_02
Bronze status:    Complete
Silver status:    Pending
```

I would track Bronze and Silver separately because successful ingestion does not guarantee successful transformation.

For an operational watermark, I would distinguish the latest date observed from the latest continuously resolved delivery date.

If 7 October is complete but 5 October is missing, "latest date = 7 October" hides the gap. The detailed tracking records must remain the source of truth.

### 12. What happens when one feed succeeds but another fails, or only some files succeed?

**Interview answer:**

> "I would isolate progress at the feed and file levels. Completed work should not need to be repeated just because another independent unit failed."

Walmart can split a delivery into multiple files and return multiple download links.

**Example:** Store Sales contains three files:

```text
File 1 -> Written to Bronze
File 2 -> Failed
File 3 -> Written to Bronze
```

The delivery remains incomplete, even though some records exist in Bronze. Recovery targets file 2, with safe retry handling for uncertain writes.

Meanwhile, Omni Sales might have completed successfully and need no recovery.

In our proposed design, Silver consumes a delivery only after all required files are accounted for. A report combining multiple feeds may additionally require a shared readiness check so it does not silently mix different coverage periods.

### 13. How do you prevent duplicate records when a job retries?

**Interview answer:**

> "I would make the ingestion writes idempotent: retrying the same input should not create an additional copy of the same ingested batch."

Consider this failure:

```text
Write 1,000 rows to Bronze -> Success
Update tracking status     -> Failure
Job retries                -> Must not append another 1,000 rows
```

A tracking table alone does not solve this gap between data commit and status update.

Delta supports idempotent writes using application and transaction identifiers. Retries must reuse the same identifiers for the same data, while new writes require correctly advancing transaction versions.

For our design, I would associate writes with stable source identities and use a tested retry-safe write strategy. I would also prevent competing runs from processing the same work uncontrolled.

**Two cautions:** a refreshed signed URL is not a new delivery. And a genuinely corrected source version must not be discarded as a duplicate.

**Do not claim:** "Delta automatically prevents every duplicate." The ingestion design still matters.

### 14. How do you distinguish "no data today" from "data is not ready"?

**Interview answer:**

> "I would check Walmart's status response rather than treating every empty result as success or failure."

Walmart documents three availability outcomes:

| Status | Meaning |
| --- | --- |
| `available` | Source processing completed and data exists. |
| `nodata` | Source processing completed without data. |
| `unavailable` | Source processing has not completed. |

These are documented for the status-by-date operation.

**Example:** At 6:00 PM, a feed is `unavailable`. Our code should leave it pending and check again later, not mark it as loaded.

For `nodata`, our tracking should record a resolved no-data outcome, distinct from an ingested delivery.

**Architect-level point:** a schedule controls when we ask. It does not guarantee that Walmart has finished preparing the data.

## Part 4: Bronze, Silver and data correctness

### 15. Why keep Bronze and Silver separate?

**Interview answer:**

> "Bronze preserves source records and their provenance. Silver provides a cleaned, validated representation suitable for downstream use. This separation lets us change transformation logic without immediately returning to the source."

Databricks describes Bronze as raw ingestion and Silver as cleaning, validation, deduplication, and handling late or out-of-order data.

For our Bronze design, I would preserve source fields and add metadata such as:

```text
_feed_name
_feed_date
_source_file_id
_ingested_at
_run_id
```

These are proposed names.

**Example:** A transformation incorrectly converts a sales amount. Because Bronze preserves the received value, we can correct the transformation and reprocess it.

Bronze should still verify technical ingestion success. "Raw" does not mean ignoring corrupt files or claiming incomplete loads are complete.

### 16. How do you handle corrections, updates and deletes from Walmart?

**Interview answer:**

> "I would preserve the delivered change records in Bronze and apply their meaning when maintaining Silver."

Walmart defines `delta_flag` values for initial/insert, update and delete. It specifies processing deliveries in feed-generation order and processing D → I → U within the relevant delivery.

**Example:** These records refer to the same complete business key:

| Delivery | Sales amount | Flag |
| --- | --- | --- |
| Original | 1,000 | I |
| Correction | 900 | U |

Bronze preserves both. Current-state Silver should show **900, not 1,900**.

Matching must use the feed's complete business key. Store Sales has a multi-column grain, not simply item number alone.

I would also avoid applying an older recovered change over a newer state blindly. The affected records may need ordered replay.

Delta `MERGE` supports updates, inserts and deletes, but ambiguous multiple source matches must be resolved before merging.

### 17. What quality checks would you apply?

**Interview answer:**

> "I would check both delivery completeness and record correctness, using rules agreed with the business."

Our proposed checks would cover required files, required key fields, valid dates and numeric types, recognized change flags, current-state key uniqueness, and reconciliation of processing outcomes.

**Example:** Suppose a delivery contains 1,000 input records. We should be able to explain that 980 were processed and 20 were quarantined, not silently lose those 20.

For critical errors, I would block publication of the affected delivery. For permitted exceptions, I would retain the record and its failure reason for investigation. Databricks documents quality-control patterns including warnings, failed updates and quarantine handling.

**Important:** Bronze and Silver row counts need not match. Several source change records can produce one current Silver record.

I would not invent business rules such as "every sales amount must be positive" without confirming the feed's semantics.

### 18. If Bronze succeeds but Silver fails, do you call Walmart again?

**Interview answer:**

> "Not when the required source records are already retained in Bronze. I would correct the downstream issue and reprocess those Bronze deliveries."

Preserving source history for reprocessing is a purpose of the Bronze layer.

**Example:**

```text
Walmart retrieval -> Complete
Bronze write      -> Complete
Silver conversion -> Failed
```

After fixing the conversion, rerun the Silver work.

Our separate statuses make this possible:

```text
bronze_status = complete
silver_status = failed
```

Re-downloading would add an unnecessary dependency on Walmart availability and create another opportunity for duplicate ingestion.

The Silver retry must also be safe: we should know which changes were committed before the failure.

### 19. What happens when Walmart adds a column or changes a data type?

**Interview answer:**

> "I would treat schema changes as an explicit contract-management problem, not silently discard new fields or automatically accept every incompatible change."

Databricks supports schema evolution, but new columns, renames and type changes have different behavior and configuration requirements.

**Example:** Walmart adds a nullable descriptive field. Our policy might allow Bronze to retain it while Silver adoption is reviewed.

A numeric field changing to incompatible text is different. That may require stopping the affected load, recording the issue, and updating the parser or transformation.

For our direct-ingestion design, an unreadable file must not be marked successfully ingested. I would retain enough error and source-identification information to recover it.

**Architect-level distinction:** tolerating expected evolution is useful; hiding unexpected incompatibility is dangerous.

## Part 5: Architecture choices and trade-offs

### 20. Why use Delta tables rather than only ordinary files?

**Interview answer:**

> "Delta provides transactional table behavior on top of data files. It supports reliable table writes, schema controls, and operations needed to maintain corrected datasets."

Delta Lake adds a transaction log and ACID transaction capabilities to its storage format.

**Example:** Silver needs to update an existing sales record when Walmart sends a correction. Delta supports table-level update and merge operations rather than requiring us to manage an unrelated collection of corrected files ourselves.

However, I would not describe an entire API-to-Bronze-to-Silver workflow as one automatic transaction. Its stages still need explicit completion tracking and recovery behavior.

The benefit is transactional table management, not automatic correctness of every business rule.

### 21. Why ingest directly into Bronze instead of maintaining a separate landing layer?

**Interview answer:**

> "Direct ingestion avoids operating a separate persistent raw-file landing stage. Its trade-off is that recovery before a successful Bronze write remains more dependent on the source."

For the design you described, I would evaluate that choice as follows:

| Direct-to-Bronze design | Separate raw-file landing alternative |
| --- | --- |
| Fewer persistent processing stages to manage. | Retains original downloaded files for independent replay. |
| Bronze is the first durable business-data copy. | File capture can succeed even when later parsing fails. |
| Recovery before Bronze may require another download. | Recovery can often start from retained files. |

These are design trade-offs, not proof that one option is universally better.

Also: "No separate S3 landing" does not mean "no underlying storage." Databricks managed tables still store data in cloud storage.

Temporary buffers or working files may exist during retrieval; that implementation detail must be verified in the notebook.

### 22. Is this streaming? Does incremental loading require Auto Loader?

**Interview answer:**

> "Incremental describes which data we process. It does not, by itself, mean that we use streaming or Auto Loader."

For the scheduled request-and-load design discussed here, scheduled incremental batch ingestion is an appropriate description.

Auto Loader incrementally processes files arriving in supported cloud storage locations. It does not directly make Walmart REST API requests.

**Example:** A job starts daily, requests unprocessed deliveries, writes Bronze, and finishes. That can be incremental without being a continuously running stream.

I would not claim Auto Loader is part of this architecture unless the code shows its file source and ingestion configuration.

**Interview distinction:** the source's incremental feeds, our incremental tracking, and Spark streaming are different concepts.

### 23. How would you scale from two feeds to many feeds?

**Interview answer:**

> "I would use configuration-driven feed processing with controlled concurrency, rather than copy the entire program for each dataset."

Our proposed configuration would map each feed to its endpoint, target table, feed type, expected schema and processing rules.

Independent feeds can use parallel tasks, while dependencies control downstream execution.

**Example:** Adding a third compatible feed should mainly add configuration and feed-specific rules, not another unrelated authentication implementation.

I would measure API latency, download throughput, parsing memory and Delta write time before selecting the optimization.

Large deliveries should be handled in bounded units rather than assuming the complete dataset fits in one process's memory.

Concurrency must respect Walmart's request quotas; unlimited parallel calls can trigger throttling.

**The objective:** increase throughput without weakening ordering, completeness or duplicate prevention.

### 24. How would you handle failures and monitor the architecture?

**Interview answer:**

> "I would classify the failure before retrying, and monitor data freshness in addition to job execution."

| Condition | Proposed response |
| --- | --- |
| Expired authentication signature | Generate fresh authentication details and retry. |
| Invalid credentials, headers or access | Investigate the configuration; avoid endless retries. |
| Throttling | Reduce request pressure and retry after a controlled delay. |
| Temporary server or network failure | Use bounded retries and increasing delays. |
| Persistent parsing or quality error | Preserve the failure details and stop the affected work. |

Walmart documents distinct authentication, access, throttling and server-error responses.

I would monitor each feed's unresolved deliveries, oldest pending work, file counts, row outcomes, and latest successfully processed delivery.

**Example:** A job can finish successfully while loading nothing because its feed selection is wrong. A green job alone does not prove current data.

Databricks provides job notifications, but our design also needs an overdue-data check.

### 25. Why not fetch old data from Walmart whenever someone needs it?

**Interview answer:**

> "Our purpose is to maintain a reliable internal dataset, not make every business query dependent on a fresh source download."

Walmart's delivery documentation describes a maximum 45-day retention for delivered incremental and historical feed files. That is an availability window, not necessarily the business-date coverage inside a history file.

For example, Store Sales documentation describes two years of historical coverage. Those are two different concepts: how much business history a dataset contains versus how long its delivery files remain retrievable.

**Example:** The team needs sales for an old business date. It should query retained, validated internal data when available.

For a missed delivery outside source availability, I would first check retained Bronze data and backups, then investigate source-supported recovery. I would not promise that any historical date can always be downloaded again.

### 26. What is your responsibility, and what architectural challenge are you addressing?

**Interview answer:**

> "My responsibility is the Bronze ingestion layer: obtaining the required Walmart deliveries, preserving their source records, and providing a dependable handoff to downstream processing. Silver transformation ownership is separate unless explicitly included in my scope."
>
> "The main architectural challenge is not simply calling the API. It is making the ingestion complete and recoverable across multiple feeds, missed days, partial failures, and repeated attempts."

**Example of a strong design explanation:**

> "A latest-only request can leave gaps after an outage. My proposed solution tracks individual feed deliveries and files, recovers missing available data, and makes retries safe. It also distinguishes successful Bronze ingestion from successful Silver processing."

Use "I proposed", "I designed" or "I implemented" according to what you actually did. Do not claim production results, performance improvements or completed recovery controls that have not been verified.

## The central interview message

Your architecture is more than "API → table."

A strong explanation connects source selection, authentication, complete delivery tracking, safe retries, raw-data preservation, correct change processing, and validated downstream data.

The sentence to remember is:

> "The objective is not just to load today's Walmart data; it is to maintain a complete, traceable, and recoverable dataset across all configured feeds."
