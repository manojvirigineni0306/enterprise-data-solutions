# Snowflake Performance Optimization Guide

**Author:** Manoj Kumar Virigineni

## Overview

This guide outlines best practices and techniques for optimizing Snowflake data warehouse performance, based on real-world enterprise implementations.

## Table of Contents

1. [Warehouse Sizing and Management](#warehouse-sizing)
2. [Query Optimization](#query-optimization)
3. [Data Loading Strategies](#data-loading)
4. [Cost Optimization](#cost-optimization)
5. [Monitoring and Alerting](#monitoring)

## Warehouse Sizing and Management {#warehouse-sizing}

### Auto-Scaling Configuration

```sql
-- Create auto-scaling warehouse
CREATE OR REPLACE WAREHOUSE ANALYTICS_WH WITH
WAREHOUSE_TYPE = 'STANDARD'
WAREHOUSE_SIZE = 'MEDIUM'
MAX_CLUSTER_COUNT = 5
MIN_CLUSTER_COUNT = 1
SCALING_POLICY = 'STANDARD'
AUTO_SUSPEND = 900
AUTO_RESUME = TRUE
INITIALLY_SUSPENDED = FALSE
MAX_CONCURRENCY_LEVEL = 8
STATEMENT_QUEUED_TIMEOUT_IN_SECONDS = 0;
```

### Warehouse Performance Monitoring

```sql
-- Monitor warehouse performance
SELECT 
    warehouse_name,
    TO_DATE(start_time) as date,
    SUM(avg_running) as sum_running,
    SUM(avg_queued_load) as sum_queued
FROM snowflake.account_usage.warehouse_load_history
WHERE TO_DATE(start_time) >= CURRENT_DATE - 30
GROUP BY 1,2
HAVING SUM(avg_queued_load) > 0
ORDER BY date DESC;
```

## Query Optimization {#query-optimization}

### Identifying Performance Issues

```sql
-- Find queries with memory spillage
SELECT 
    query_id, 
    SUBSTR(query_text, 1, 50) as partial_query,
    user_name, 
    warehouse_name,
    bytes_spilled_to_local_storage,
    bytes_spilled_to_remote_storage
FROM snowflake.account_usage.query_history
WHERE (bytes_spilled_to_local_storage > 0 
       OR bytes_spilled_to_remote_storage > 0)
    AND start_time::date > CURRENT_DATE - 7
ORDER BY bytes_spilled_to_remote_storage DESC
LIMIT 10;
```

### Cache Performance Analysis

```sql
-- Analyze cache performance
SELECT 
    warehouse_name,
    COUNT(*) as query_count,
    SUM(bytes_scanned) as bytes_scanned,
    SUM(bytes_scanned * percentage_scanned_from_cache) as bytes_from_cache,
    (bytes_from_cache / bytes_scanned) * 100 as cache_hit_ratio
FROM snowflake.account_usage.query_history
WHERE start_time >= CURRENT_DATE - 7
    AND bytes_scanned > 0
GROUP BY 1
ORDER BY cache_hit_ratio DESC;
```

## Data Loading Strategies {#data-loading}

### Bulk Loading Best Practices

```sql
-- Create optimized file format
CREATE OR REPLACE FILE FORMAT enterprise_csv_format
TYPE = CSV 
FIELD_DELIMITER = ',' 
SKIP_HEADER = 1 
DATE_FORMAT = 'YYYY-MM-DD' 
TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS'
EMPTY_FIELD_AS_NULL = TRUE
NULL_IF = ('', 'NULL', 'null')
FIELD_OPTIONALLY_ENCLOSED_BY = '"';

-- Create stage for data loading
CREATE OR REPLACE STAGE enterprise_data_stage
FILE_FORMAT = enterprise_csv_format
DIRECTORY = (ENABLE = TRUE);
```

### Change Data Capture (CDC) Implementation

```sql
-- CDC processing with merge operation
MERGE INTO target_table t
USING (
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY unique_key 
            ORDER BY last_modified_date DESC
        ) as rn
    FROM stage_table
    WHERE last_modified_date > $last_processed_timestamp
) s ON t.unique_key = s.unique_key AND s.rn = 1
WHEN MATCHED AND s.operation = 'DELETE' THEN DELETE
WHEN MATCHED AND s.operation != 'DELETE' THEN UPDATE SET
    t.column1 = s.column1,
    t.column2 = s.column2,
    t.last_modified_date = s.last_modified_date
WHEN NOT MATCHED AND s.operation != 'DELETE' THEN INSERT
    (unique_key, column1, column2, last_modified_date)
VALUES 
    (s.unique_key, s.column1, s.column2, s.last_modified_date);
```

## Cost Optimization {#cost-optimization}

### Credit Usage Monitoring

```sql
-- Monitor credit consumption by warehouse
SELECT 
    warehouse_name,
    DATE_TRUNC('day', start_time) as usage_date,
    SUM(credits_used) as daily_credits,
    AVG(credits_used) as avg_credits_per_query
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time >= CURRENT_DATE - 30
GROUP BY 1, 2
ORDER BY daily_credits DESC;
```

### Storage Cost Analysis

```sql
-- Analyze storage costs and usage
SELECT 
    database_name,
    schema_name,
    SUM(active_bytes) / POWER(1024, 3) as active_storage_gb,
    SUM(time_travel_bytes) / POWER(1024, 3) as time_travel_gb,
    SUM(failsafe_bytes) / POWER(1024, 3) as failsafe_gb
FROM snowflake.account_usage.table_storage_metrics
WHERE deleted IS NULL
GROUP BY 1, 2
ORDER BY active_storage_gb DESC;
```

## Monitoring and Alerting {#monitoring}

### Automated Monitoring Procedure

```sql
CREATE OR REPLACE PROCEDURE monitor_system_health()
RETURNS VARCHAR(16777216)
LANGUAGE SQL
EXECUTE AS CALLER
AS 
$$
DECLARE
    failed_queries INTEGER := 0;
    long_running_queries INTEGER := 0;
    alert_message VARCHAR;
BEGIN
    
    -- Check for failed queries in last hour
    SELECT COUNT(*) INTO failed_queries
    FROM snowflake.account_usage.query_history
    WHERE execution_status = 'FAIL'
        AND start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP());
    
    -- Check for long-running queries
    SELECT COUNT(*) INTO long_running_queries
    FROM snowflake.account_usage.query_history
    WHERE execution_time > 300000  -- 5 minutes
        AND start_time >= DATEADD(hour, -1, CURRENT_TIMESTAMP());
    
    -- Generate alerts if thresholds exceeded
    IF (failed_queries > 10) THEN
        SET alert_message = 'ALERT: High number of failed queries: ' || failed_queries;
        -- Send notification (implementation specific)
    END IF;
    
    IF (long_running_queries > 5) THEN
        SET alert_message = 'ALERT: Multiple long-running queries detected: ' || long_running_queries;
        -- Send notification (implementation specific)
    END IF;
    
    RETURN 'Monitoring completed. Failed queries: ' || failed_queries || 
           ', Long-running queries: ' || long_running_queries;
           
END;
$$;
```

## Performance Tuning Checklist

### Daily Operations
- [ ] Monitor warehouse credit consumption
- [ ] Review query performance metrics
- [ ] Check for failed or long-running queries
- [ ] Validate data loading processes
- [ ] Review storage growth patterns

### Weekly Operations
- [ ] Analyze warehouse sizing efficiency
- [ ] Review cache hit ratios
- [ ] Optimize high-cost queries
- [ ] Clean up unnecessary time travel data
- [ ] Update clustering keys if needed

### Monthly Operations
- [ ] Comprehensive cost analysis
- [ ] Performance trend analysis
- [ ] Capacity planning review
- [ ] Security and compliance audit
- [ ] Documentation updates

## Best Practices Summary

1. **Warehouse Management**
   - Use auto-scaling for variable workloads
   - Set appropriate auto-suspend timeouts
   - Monitor and adjust warehouse sizes based on usage

2. **Query Optimization**
   - Minimize data scanning with proper filtering
   - Use clustering keys for frequently accessed data
   - Leverage result caching when possible

3. **Data Loading**
   - Use bulk loading for large datasets
   - Implement proper error handling
   - Validate data quality during loading

4. **Cost Control**
   - Regular monitoring of credit usage
   - Optimize storage through lifecycle policies
   - Right-size warehouses based on actual usage

---

**Author:** Manoj Kumar Virigineni  
**Last Updated:** October 2025  
**Version:** 1.0