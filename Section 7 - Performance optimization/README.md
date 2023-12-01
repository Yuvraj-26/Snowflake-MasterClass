## Performance Optimization

Optimization of the performance of data warehouse

Goal: Make queries run faster, which save costs (compute power usage)

Traditional way:
- Add indexes, primary keys
- Create table partitions
- Analyse the query execution table plan
- Remove unnecessary full table scans

In Snowflake:
- Automatically managed micro-partitions - ensures performance when running queries

### Our job:
- Assign appropriate data types
- sizing virtual warehouses
- cluster keys for large tables

#### Dedicated virtual warehouses
- separated according to different workloads
- create a dedicated warehouse for certain user groups that are larger with different needs and queries

#### Scaling Up
- For known patterns of high work load, increase size of the data warehouse

#### Scaling Out
- Introducing multi cluster warehouses
- Dynamically, depending on the workload, scale our multi cluster warehouse out, and automatically create new warehouse clusters
- Dynamically for unknown patterns of work loads (different amount of users and workload)

#### Maximise Cache usage
- Automatic caching can be maximised

#### Cluster keys
- For large tables



```sql

```


```sql

```
