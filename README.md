# backend-system-design-architecture-dataengineering

System design and connection flow for data engineering backend architectures, explored one at a time. Each architecture has its own folder with a deep-dive write-up and animated diagrams of how every component connects: protocols, security, rules, failure handling and cadence.

## Architectures

| # | Architecture | Status |
| --- | --- | --- |
| 01 | [Walmart data feeds into Databricks](01-walmart-data-feeds-databricks/README.md) (feed = one dataset, request flow) | 🔍 In progress: feed concept and API flow documented |

A new folder and table row are added as each architecture is explored and confirmed.

## Conventions

- Animations are dark-theme SVGs using SMIL or CSS, with `prefers-reduced-motion` respected, so they play inside GitHub READMEs without JavaScript.
- Numbered badges on a diagram match the rows of the connections table.
- Claims are limited to what was built or verified. Designs that are not built are labelled as designs, and numbers are approximate.
