# backend-system-design-architecture-dataengineering

System design and connection flow for data engineering backend architectures, explored one at a time. Each architecture has its own folder with a deep-dive write-up and animated diagrams of how every component connects: protocols, security, rules, failure handling and cadence.

## Architectures

| # | Architecture | Status |
| --- | --- | --- |
| 01 | [Streaming ingestion to a lakehouse](01-streaming-ingestion-lakehouse/README.md) (Coinbase → EC2 → S3 → Databricks) | ✅ Explored, built and running |

A new folder and table row are added as each architecture is explored and confirmed.

## Conventions

- Animations are dark-theme SVGs using SMIL or CSS, with `prefers-reduced-motion` respected, so they play inside GitHub READMEs without JavaScript.
- Numbered badges on a diagram match the rows of the connections table.
- Claims are limited to what was built or verified. Designs that are not built are labelled as designs, and numbers are approximate.
