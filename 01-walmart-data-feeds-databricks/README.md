# 01. Walmart data feeds into Databricks

> **Clarification:** "one request, one feed" means one feed per **request**, not one feed per pipeline run. One run can make several feed requests. See [07](../07-multiple-feeds-in-one-pipeline/README.md).

System design and connection flow for pulling Walmart Data Ventures data into Databricks, one feed at a time.

## My feed understanding

<div align="center">
    <img src="animations/feeds.svg" alt="Animated diagram: a Databricks job requests the Store Sales feed and gets only Store Sales data, then makes a separate request to the Store Inventory feed endpoint and gets only Store Inventory data" width="100%"/>
</div>

- **A feed is one particular dataset.** For example, Store Sales holds sales information and Store Inventory holds inventory information.
- **The Databricks code chooses which feed to request.** It does not ask for all Walmart data together.
- **One request, one feed.** If there are 10 datasets and Databricks requests Store Sales, it gets Store Sales data only, not all 10.
- **A second dataset needs a second request,** made to that feed's own endpoint.
- The "10" is only an example. The real number of feeds is not fixed here, and the feed names shown are examples, not a full list.

## Two ways to receive Walmart data

| Route | How it works |
| --- | --- |
| **API data feeds** | Request a feed from an endpoint, receive a signed URL, download files. |
| **Cloud Feeds (Delta Sharing)** | Walmart shares datasets cloud to cloud through Databricks Delta Sharing, with no download step. |

Neither route is drawn yet. The animation above covers only the feed concept.

## Still to confirm

- Which route Databricks uses in your setup, API data feeds or Cloud Feeds.
- The exact feed names and endpoints available to your account. Store Sales and Store Inventory come from my notes, and I have not confirmed them against the API reference.
- Where the files land (Unity Catalog Volume or S3) and how the job is scheduled.

## Sources

- [Walmart Data Ventures developer docs: Step 5, Downloading Data](https://developer.walmartdataventures.com/scintilla-media-data-feed/docs/step-5-downloading-data)
- [Walmart Data Ventures API v2 docs](https://developer.walmartdataventures.com/apis/v2/docs)
- [Databricks Data + AI Summit: Cloud-to-Cloud Data Sharing by Walmart](https://www.databricks.com/dataaisummit/session/cloud-cloud-data-sharing-walmart-direct-access-omni-channel-sales-data)
