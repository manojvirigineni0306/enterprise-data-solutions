# Enterprise Data Security Framework

**Author:** Manoj Kumar Virigineni

## Overview

This document outlines a comprehensive data security framework implemented for enterprise financial data platforms, focusing on PII protection, access control, and compliance.

## Security Architecture

### 1. Data Classification

#### PII Levels
```sql
-- PII Level 1: Public data (no masking required)
-- PII Level 2: Internal data (basic masking)
-- PII Level 3: Sensitive data (advanced masking)
-- PII Level 4: Highly sensitive data (full encryption/tokenization)

-- Example tagging strategy
ALTER TABLE customer_data SET TAG pii_level = 'level_3';
ALTER TABLE customer_data MODIFY COLUMN ssn SET TAG pii_level = 'level_4';
ALTER TABLE customer_data MODIFY COLUMN email SET TAG pii_level = 'level_2';
```

### 2. Masking Policies

#### Dynamic Data Masking Implementation
```sql
-- Level 2 String Masking Policy
CREATE OR REPLACE MASKING POLICY pii_level2_string_policy AS (val STRING) 
RETURNS STRING ->
CASE 
  WHEN CURRENT_ROLE() IN ('ADMIN_ROLE', 'UNMASKING_ROLE') THEN val
  ELSE REGEXP_REPLACE(val, '.', '*')
END;

-- Level 3 String Masking Policy  
CREATE OR REPLACE MASKING POLICY pii_level3_string_policy AS (val STRING) 
RETURNS STRING ->
CASE 
  WHEN CURRENT_ROLE() IN ('ADMIN_ROLE', 'LEVEL3_UNMASKING_ROLE') THEN val
  WHEN CURRENT_ROLE() IN ('ANALYST_ROLE') THEN 
    CONCAT(LEFT(val, 2), REPEAT('*', LENGTH(val) - 4), RIGHT(val, 2))
  ELSE '***MASKED***'
END;

-- Level 4 String Masking Policy (Highest Security)
CREATE OR REPLACE MASKING POLICY pii_level4_string_policy AS (val STRING) 
RETURNS STRING ->
CASE 
  WHEN CURRENT_ROLE() IN ('ADMIN_ROLE', 'LEVEL4_UNMASKING_ROLE') THEN val
  ELSE '***CLASSIFIED***'
END;
```

#### Numeric Data Masking
```sql
-- Numeric masking for financial data
CREATE OR REPLACE MASKING POLICY financial_amount_policy AS (val NUMBER) 
RETURNS NUMBER ->
CASE 
  WHEN CURRENT_ROLE() IN ('FINANCE_ADMIN', 'AUDIT_ROLE') THEN val
  WHEN CURRENT_ROLE() IN ('ANALYST_ROLE') THEN 
    ROUND(val / 1000) * 1000  -- Round to nearest thousand
  ELSE NULL
END;
```

### 3. Role-Based Access Control (RBAC)

#### Hierarchical Role Structure
```sql
-- Create role hierarchy
CREATE ROLE enterprise_admin_role;
CREATE ROLE data_engineer_role;
CREATE ROLE data_analyst_role;
CREATE ROLE business_user_role;

-- Grant role hierarchy
GRANT ROLE business_user_role TO ROLE data_analyst_role;
GRANT ROLE data_analyst_role TO ROLE data_engineer_role;
GRANT ROLE data_engineer_role TO ROLE enterprise_admin_role;

-- Database-specific roles
CREATE ROLE mortgage_db_admin;
CREATE ROLE mortgage_db_read_write;
CREATE ROLE mortgage_db_read_only;

-- Grant permissions
GRANT USAGE ON DATABASE mortgage_db TO ROLE mortgage_db_read_only;
GRANT USAGE ON ALL SCHEMAS IN DATABASE mortgage_db TO ROLE mortgage_db_read_only;
GRANT SELECT ON ALL TABLES IN DATABASE mortgage_db TO ROLE mortgage_db_read_only;
```

#### Functional Roles
```sql
-- Create functional roles for specific access patterns
CREATE ROLE pii_level1_unmasking_role;
CREATE ROLE pii_level2_unmasking_role;
CREATE ROLE pii_level3_unmasking_role;
CREATE ROLE pii_level4_unmasking_role;

-- Service account roles
CREATE ROLE etl_service_role;
CREATE ROLE reporting_service_role;
CREATE ROLE api_service_role;
```

### 4. Authentication Policies

#### Multi-Factor Authentication
```sql
-- Create authentication policy for service accounts
CREATE OR REPLACE AUTHENTICATION POLICY service_account_auth_policy
  CLIENT_TYPES = ('DRIVERS', 'SNOWFLAKE_UI')
  AUTHENTICATION_METHODS = ('PASSWORD', 'KEYPAIR')
  MFA_ENROLLMENT = REQUIRED
  MFA_AUTHENTICATION_METHODS = ('PASSWORD', 'SAML')
  COMMENT = 'Authentication policy for service accounts requiring MFA';

-- Apply to service accounts
ALTER USER etl_service_user SET AUTHENTICATION POLICY service_account_auth_policy;
```

### 5. Network Security

#### Network Policies
```sql
-- Create network policy for IP restrictions
CREATE OR REPLACE NETWORK POLICY corporate_network_policy
  ALLOWED_IP_LIST = ('192.168.1.0/24', '10.0.0.0/8')
  BLOCKED_IP_LIST = ()
  COMMENT = 'Allow access only from corporate network ranges';

-- Apply network policy to users
ALTER USER production_user SET NETWORK_POLICY = corporate_network_policy;
```

### 6. Audit and Monitoring

#### Security Monitoring Queries
```sql
-- Monitor failed login attempts
SELECT 
    user_name,
    client_ip,
    reported_client_type,
    error_code,
    error_message,
    event_timestamp
FROM snowflake.account_usage.login_history
WHERE is_success = 'NO'
    AND event_timestamp >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
ORDER BY event_timestamp DESC;

-- Monitor privilege escalations
SELECT 
    grantee_name,
    role,
    granted_on,
    granted_by,
    created_on
FROM snowflake.account_usage.grants_to_users
WHERE created_on >= DATEADD(day, -7, CURRENT_TIMESTAMP())
    AND role IN ('SYSADMIN', 'SECURITYADMIN', 'ACCOUNTADMIN')
ORDER BY created_on DESC;

-- Monitor data access patterns
SELECT 
    user_name,
    role_name,
    database_name,
    schema_name,
    COUNT(*) as query_count,
    SUM(CASE WHEN execution_status = 'SUCCESS' THEN 1 ELSE 0 END) as successful_queries
FROM snowflake.account_usage.query_history
WHERE start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
    AND query_type = 'SELECT'
GROUP BY 1,2,3,4
HAVING query_count > 100
ORDER BY query_count DESC;
```

### 7. Data Loss Prevention

#### Row Access Policies
```sql
-- Create row access policy for department-based access
CREATE OR REPLACE ROW ACCESS POLICY department_access_policy AS (department_id NUMBER) 
RETURNS BOOLEAN ->
  CASE 
    WHEN CURRENT_ROLE() IN ('ADMIN_ROLE') THEN TRUE
    WHEN CURRENT_ROLE() = 'FINANCE_ROLE' AND department_id = 100 THEN TRUE
    WHEN CURRENT_ROLE() = 'HR_ROLE' AND department_id = 200 THEN TRUE
    ELSE FALSE
  END;

-- Apply row access policy
ALTER TABLE employee_data ADD ROW ACCESS POLICY department_access_policy ON (department_id);
```

### 8. Encryption Strategy

#### Data Encryption at Rest
```sql
-- All data automatically encrypted at rest in Snowflake
-- Additional client-side encryption for highly sensitive data
CREATE OR REPLACE FUNCTION encrypt_sensitive_data(plaintext STRING)
RETURNS STRING
LANGUAGE PYTHON
RUNTIME_VERSION = 3.8
HANDLER = 'encrypt_data'
AS $$
import base64
from cryptography.fernet import Fernet

def encrypt_data(plaintext):
    # Use organization-specific encryption key
    key = b'your-organization-encryption-key-here'
    f = Fernet(key)
    encrypted = f.encrypt(plaintext.encode())
    return base64.urlsafe_b64encode(encrypted).decode()
$$;
```

### 9. Compliance Framework

#### SOX Compliance Monitoring
```sql
-- Create audit table for SOX compliance
CREATE OR REPLACE TABLE sox_audit_log (
    audit_id NUMBER AUTOINCREMENT,
    user_name VARCHAR(100),
    action_type VARCHAR(50),
    table_name VARCHAR(200),
    affected_rows NUMBER,
    timestamp TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    session_id VARCHAR(100)
);

-- Create trigger for audit logging (conceptual - Snowflake uses streams)
CREATE OR REPLACE STREAM financial_data_changes ON TABLE financial_transactions;

-- Process audit trail
CREATE OR REPLACE TASK process_sox_audit
WAREHOUSE = audit_warehouse
SCHEDULE = '5 MINUTE'
AS
INSERT INTO sox_audit_log (user_name, action_type, table_name, affected_rows, session_id)
SELECT 
    CURRENT_USER(),
    METADATA$ACTION,
    'FINANCIAL_TRANSACTIONS',
    COUNT(*),
    CURRENT_SESSION()
FROM financial_data_changes
WHERE METADATA$ACTION IN ('INSERT', 'UPDATE', 'DELETE')
GROUP BY METADATA$ACTION;
```

### 10. Security Best Practices

#### Daily Security Checklist
```sql
-- Daily security health check procedure
CREATE OR REPLACE PROCEDURE daily_security_check()
RETURNS TABLE(check_type VARCHAR, status VARCHAR, details VARCHAR)
LANGUAGE SQL
AS $$
BEGIN
  LET result RESULTSET := (
    -- Check 1: Failed logins
    SELECT 'FAILED_LOGINS' as check_type,
           CASE WHEN COUNT(*) > 10 THEN 'ALERT' ELSE 'OK' END as status,
           'Failed logins in last 24h: ' || COUNT(*) as details
    FROM snowflake.account_usage.login_history
    WHERE is_success = 'NO' 
      AND event_timestamp >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
    
    UNION ALL
    
    -- Check 2: Unusual data access
    SELECT 'UNUSUAL_ACCESS' as check_type,
           CASE WHEN COUNT(*) > 5 THEN 'ALERT' ELSE 'OK' END as status,
           'Users with >1000 queries: ' || COUNT(*) as details
    FROM (
      SELECT user_name, COUNT(*) as query_count
      FROM snowflake.account_usage.query_history
      WHERE start_time >= DATEADD(day, -1, CURRENT_TIMESTAMP())
      GROUP BY user_name
      HAVING query_count > 1000
    )
    
    UNION ALL
    
    -- Check 3: Privilege changes
    SELECT 'PRIVILEGE_CHANGES' as check_type,
           CASE WHEN COUNT(*) > 0 THEN 'ALERT' ELSE 'OK' END as status,
           'Admin role grants in last 24h: ' || COUNT(*) as details
    FROM snowflake.account_usage.grants_to_users
    WHERE created_on >= DATEADD(hour, -24, CURRENT_TIMESTAMP())
      AND role IN ('SYSADMIN', 'SECURITYADMIN', 'ACCOUNTADMIN')
  );
  RETURN TABLE(result);
END;
$$;
```

## Implementation Timeline

### Phase 1: Foundation (Weeks 1-2)
- Implement basic RBAC structure
- Deploy initial masking policies
- Set up authentication policies

### Phase 2: Advanced Security (Weeks 3-4)
- Implement row-level security
- Deploy network policies
- Create audit monitoring

### Phase 3: Compliance (Weeks 5-6)
- SOX compliance framework
- Advanced monitoring and alerting
- Documentation and training

## Maintenance and Updates

### Weekly Tasks
- Review security audit logs
- Update access permissions as needed
- Monitor policy effectiveness

### Monthly Tasks  
- Comprehensive security assessment
- Update masking policies based on new data
- Review and update role assignments

### Quarterly Tasks
- Full compliance audit
- Security policy review
- Penetration testing coordination

---

**Author:** Manoj Kumar Virigineni  
**Classification:** Internal Use Only  
**Last Updated:** October 2025  
**Version:** 2.0