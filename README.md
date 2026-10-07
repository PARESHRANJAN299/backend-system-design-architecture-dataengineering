# backend-system-design-architecture-dataengineering

System design and connection flow for data engineering backend architectures, explored one at a time. Each architecture has its own folder with a deep-dive write-up and animated diagrams of how every component connects: protocols, security, rules, failure handling and cadence.

## Phase 1: What data to request and when to load it

**Status: ✅ understood and documented.** Three folders cover the first part of the system design: which data to ask Walmart for, and when to load it.

| # | Folder | What it settles |
| --- | --- | --- |
| 01 | [Walmart data feeds into Databricks](01-walmart-data-feeds-databricks/README.md) | A feed is one dataset, and each request names one feed. |
| 02 | [Walmart operations and methods](02-walmart-operations-methods/README.md) | Four operations using two methods (GET and POST), and which one to use for which data. |
| 03 | [Production pipeline design](03-production-pipeline-design/README.md) | One parameterized job: initial load, then daily incremental loads, and the team reads from Delta instead of calling Walmart. |

**Phase 1 in one line:** pick one feed, choose the operation for the data you need, load the history once, then load only new files each day, and let the team query the stored Delta table.

The animations are in each folder's `animations/` directory. Parts that are proposed designs, not built pipelines, are labelled as such in the folders.

## Phase 2: Next, not started

Phase 1 settled what to request and when. Phase 2 will cover the part that comes after:

- **How the data reaches Bronze Delta:** where the downloaded files land and how they are loaded into the Bronze table.
- **How the pipeline runs reliably:** retries, tracking what loaded, handling failures and missed deliveries.

| # | Folder | What it covers | Status |
| --- | --- | --- | --- |
| 04 | [Authentication to Bronze](04-authentication-to-bronze/README.md) | One-time key setup, signing every request, download links, landing files and writing Bronze Delta, with failure cases. | 🔍 In progress |
| 05 | [Recovering from a pipeline gap](05-recovering-from-a-pipeline-gap/README.md) | A two-day outage found by Paresh and fixed inside the same pipeline: tracking table, status check by date, request by feed date. | 🔍 In progress |

More Phase 2 folders will be added once the understanding is complete and confirmed.

## Conventions

- Animations are dark-theme SVGs using SMIL or CSS, with `prefers-reduced-motion` respected, so they play inside GitHub READMEs without JavaScript.
- Numbered badges on a diagram match the rows of the connections table.
- Claims are limited to what was built or verified. Designs that are not built are labelled as designs, and numbers are approximate.
