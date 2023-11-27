## Snowflake

Snowflake provides a data platform that allows businesses to store, analyse, and share large amounts of data across multiple cloud providers. It's known for its architecture that separates storage and compute, enabling scalability, flexibility, and ease of use in managing and querying data

A data warehouse (DW) is a relational database that is designed for analytical rather than transactional work. It collects and aggregates data from one or many sources. It serves as a federated repository for all or certain data sets collected by a business’s operational systems.

## Snowflake Architecture Overview

Cloud Services - Managing Infrastructure, Access control, Security, Optimizer, Metadata

Query Processing - Virtual warehouse are the virtual compute resources to perform operations and compute queries. Virtual warehouses perform MMP (Massive Parallel Processing) where complex queries for big data is processed is processed by multiple servers at the same time.

Storage - Data is stored in Amazon AWS S3 bucket, stored in Hybrid Columnar Storage - saved and compressed into blobs. Fetch blobs when querying which is used in big-data making storage efficient with faster querying.

## Virtual Warehouse Sizes

Virtual compute instances or servers used to process queries on the data.

### Sizes
- XS | 1 (For XS 1 credit consumed per hour when server is active, billed by seconds with a minimum of 1 minute
- S | 2
- M | 4
- L | 8
- XL | 16
- 4XL | 128 servers

### Multi-Clustering
With multi-cluster warehouses, Snowflake supports allocating, either statically or dynamically, additional clusters to make a larger pool of compute resources available. Cluster of warehouses allows query processing to be divided between resources.
