# Lakehouse Architecture Playbook

This repository contains a bilingual playbook on designing a low-complexity lakehouse architecture. Snowflake is used as the primary reference platform and analytical core throughout the playbook, but this is not vendor marketing and should not be read as a claim that Snowflake is the right choice for every organization, workload, or latency requirement.

The focus is on architecture design trade-offs: how to structure ingestion, Medallion modeling, near real-time data consumption, governance, operational serving, migration, and FinOps in a way that balances data freshness, system simplicity, governance, operability, and total cost of ownership. Many of the same principles can also be evaluated against other lakehouse or warehouse platforms, such as Databricks, Microsoft Fabric, BigQuery, Redshift, or similar architectures.

The Chinese and English versions are maintained in separate folders for easier reading, comparison, and future refinement.
