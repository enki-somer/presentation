1. Are the scripts full refresh or incremental?
2. Where do we store last successful load time/checkpoint?
3. Does Airflow run the scripts, or only monitor them?
4. How do we handle deletes from source systems?
5. How do we handle schema changes?
6. Are raw tables stored in ClickHouse before joins, or do scripts directly create joined tables?
7. Do we have a metadata table for row counts, duration, failures, and last loaded timestamp?
8. For 1B-row tables, do we chunk by date, id range, or partitions?
9. Who owns pipeline alerting when a job fails?
10. Are transformations expected to stay in Python/SQL scripts or move to dbt?
