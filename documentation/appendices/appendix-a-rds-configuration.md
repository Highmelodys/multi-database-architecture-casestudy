# 📖 Appendix A - RDS Configuration Details

## 🗄️ Primary RDS Configuration

### Database Instance Specifications
```yaml
DB Instance Identifier: xyz-corp-primary-db
Database Engine: MySQL 8.0.35
Instance Class: db.t3.medium
  - vCPUs: 2
  - RAM: 4 GiB
  - Network Performance: Up to 5 Gigabit
  - EBS Bandwidth: Up to 2,085 Mbps

Storage Configuration:
  - Storage Type: General Purpose SSD (gp2)
  - Allocated Storage: 100 GB
  - Max Storage Threshold: 1000 GB (Auto Scaling Enabled)
  - Storage Encrypted: Yes
  - KMS Key: aws/rds
  - Storage Throughput: 3,000 IOPS (baseline)
```

### Multi-AZ Configuration
```yaml
Multi-AZ Deployment: Yes
Primary AZ: us-east-1a
Standby AZ: us-east-1b
Failover Method: Automatic
Estimated Failover Time: 60-120 seconds
Replication Type: Synchronous (Zero Data Loss)
```

### Network and Security Configuration
```yaml
VPC: vpc-xyz-corp-main
DB Subnet Group: xyz-corp-db-subnet-group
  - Subnet 1: subnet-xyz-private-1a (us-east-1a) - 10.0.1.0/24
  - Subnet 2: subnet-xyz-private-1b (us-east-1b) - 10.0.2.0/24

Security Groups:
  - xyz-corp-db-security-group (sg-xyz-db-001)
    Inbound Rules:
      - Type: MySQL/Aurora (3306)
      - Source: Application Security Group (sg-xyz-app-001)
      - Description: Allow database access from applications
    
    Outbound Rules:
      - Type: HTTPS (443)
      - Destination: 0.0.0.0/0
      - Description: Allow outbound HTTPS for updates and patches

Public Accessibility: No
```

### Database Configuration Parameters
```sql
-- Character Set and Collation
character_set_server = utf8mb4
collation_server = utf8mb4_unicode_ci

-- InnoDB Configuration
innodb_buffer_pool_size = 2867085107  -- ~2.7GB (75% of 4GB RAM)
innodb_buffer_pool_instances = 8
innodb_buffer_pool_chunk_size = 134217728  -- 128MB
innodb_log_file_size = 536870912  -- 512MB
innodb_log_files_in_group = 2
innodb_log_buffer_size = 16777216  -- 16MB
innodb_flush_log_at_trx_commit = 1
innodb_flush_method = O_DIRECT
innodb_file_per_table = 1
innodb_open_files = 2000

-- Connection Management
max_connections = 405  -- Based on db.t3.medium limits
max_user_connections = 400
max_connect_errors = 999999
connect_timeout = 10
interactive_timeout = 28800
wait_timeout = 28800

-- Query Cache
query_cache_type = 1
query_cache_size = 33554432  -- 32MB
query_cache_limit = 2097152  -- 2MB

-- Temporary Tables
tmp_table_size = 67108864  -- 64MB
max_heap_table_size = 67108864  -- 64MB

-- MyISAM Configuration
key_buffer_size = 134217728  -- 128MB

-- Logging
general_log = 0
slow_query_log = 1
long_query_time = 2
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1

-- Binary Logging
log_bin = mysql-bin
binlog_format = ROW
expire_logs_days = 7
max_binlog_size = 134217728  -- 128MB

-- Table Cache
table_open_cache = 2000
table_definition_cache = 1400
table_open_cache_instances = 16

-- Thread Configuration
thread_cache_size = 79
thread_stack = 262144

-- Sort and Join Configuration
sort_buffer_size = 2097152  -- 2MB
read_buffer_size = 131072   -- 128KB
read_rnd_buffer_size = 262144  -- 256KB
join_buffer_size = 262144   -- 256KB
```

### Backup and Maintenance Configuration
```yaml
Automated Backups:
  Backup Retention Period: 2 days
  Backup Window: 03:00-04:00 UTC
  Copy Tags to Snapshots: Yes
  Delete Automated Backups: Yes (when instance deleted)

Maintenance:
  Maintenance Window: sun:04:00-sun:05:00 UTC
  Auto Minor Version Upgrade: Yes
  
Manual Snapshots:
  Snapshot Identifier: xyz-corp-db-baseline-snapshot
  Created: Before major configuration changes
  Retention: Manual (indefinite until manually deleted)

Point-in-Time Recovery:
  Enabled: Yes
  Recovery Window: Up to 2 days
  Granularity: 5-minute intervals
```

### Monitoring Configuration
```yaml
Enhanced Monitoring:
  Enabled: Yes
  Granularity: 60 seconds
  Monitoring Role: rds-monitoring-role
  
Performance Insights:
  Enabled: Yes
  Retention Period: 7 days (free tier)
  KMS Key: aws/pi
  
CloudWatch Logs:
  Error Log: /aws/rds/instance/xyz-corp-primary-db/error
  General Log: Disabled (for performance)
  Slow Query Log: /aws/rds/instance/xyz-corp-primary-db/slowquery
  
CloudWatch Metrics (Default):
  - CPUUtilization
  - DatabaseConnections
  - FreeableMemory
  - FreeStorageSpace
  - ReadLatency
  - WriteLatency
  - ReadIOPS
  - WriteIOPS
  - NetworkReceiveThroughput
  - NetworkTransmitThroughput
```

## 🔄 Read Replica Configuration

### US-West-2 Read Replica
```yaml
DB Instance Identifier: xyz-corp-replica-west
Source DB: arn:aws:rds:us-east-1:123456789012:db:xyz-corp-primary-db
Region: us-west-2
Instance Class: db.t3.medium
Storage: Inherits from source (100 GB GP2)
Multi-AZ: No (Read replica)

Network Configuration:
  VPC: vpc-xyz-corp-west
  DB Subnet Group: xyz-corp-west-subnet-group
  Security Group: sg-xyz-west-db-001
  Public Accessibility: No

Replication:
  Type: Asynchronous
  Average Lag: 2-5 seconds
  Monitoring: ReplicaLag CloudWatch metric
```

### US-East-2 (Ohio) Read Replica
```yaml
DB Instance Identifier: xyz-corp-replica-central
Source DB: arn:aws:rds:us-east-1:123456789012:db:xyz-corp-primary-db
Region: us-east-2
Instance Class: db.t3.medium
Storage: Inherits from source (100 GB GP2)
Multi-AZ: No (Read replica)

Network Configuration:
  VPC: vpc-xyz-corp-central
  DB Subnet Group: xyz-corp-central-subnet-group
  Security Group: sg-xyz-central-db-001
  Public Accessibility: No

Replication:
  Type: Asynchronous
  Average Lag: 2-5 seconds
  Monitoring: ReplicaLag CloudWatch metric
```

## 🔐 Security Configuration

### Encryption Settings
```yaml
Encryption at Rest:
  Enabled: Yes
  KMS Key: aws/rds (AWS Managed)
  Algorithm: AES-256
  
Encryption in Transit:
  SSL/TLS: Required
  Certificate: rds-ca-2019
  Minimum TLS Version: 1.2

Database Authentication:
  IAM Database Authentication: Enabled
  Traditional Username/Password: Enabled (fallback)
  
User Accounts:
  Master Username: admin
  Master Password: Stored in AWS Secrets Manager
  Additional Users: Created via IAM and SQL
```

### Parameter Groups
```yaml
Primary DB Parameter Group: xyz-corp-mysql8-params
  Family: mysql8.0
  Description: Custom parameter group for XYZ Corp MySQL instances
  
Read Replica Parameter Groups:
  - xyz-corp-mysql8-replica-west
  - xyz-corp-mysql8-replica-central
  
Key Custom Parameters:
  - innodb_buffer_pool_size: {DBInstanceClassMemory*3/4}
  - slow_query_log: 1
  - long_query_time: 2
  - max_connections: 405
  - query_cache_size: 33554432
```

### Option Groups
```yaml
Option Group: xyz-corp-mysql8-options
  Engine: mysql
  Major Engine Version: 8.0
  
Options Enabled:
  - MEMCACHED
    Port: 11211
    VPC Security Groups: sg-xyz-memcached-001
```

## 📊 Performance and Sizing

### Instance Sizing Rationale
```yaml
Current Sizing (db.t3.medium):
  Justification:
    - Expected concurrent connections: 100-200
    - Dataset size: 100GB (growing to 500GB in Year 1)
    - Query complexity: Medium
    - Budget constraints: $85/month for Multi-AZ
  
  Performance Expectations:
    - CPU Utilization: 40-60% average
    - Memory Utilization: 75-85%
    - IOPS: 1000-2000 average
    - Connection Pool: 80-90% utilization

Future Scaling Path:
  Year 2: db.t3.large (4 vCPU, 8GB RAM)
  Year 3: db.m5.xlarge (4 vCPU, 16GB RAM)
  High Load Events: db.r5.large (2 vCPU, 16GB RAM)
```

### Storage Performance
```yaml
GP2 Storage Performance:
  Baseline: 3 IOPS per GB (300 IOPS for 100GB)
  Burst: Up to 3,000 IOPS
  Burst Credit Balance: 5.4 million I/O credits
  Throughput: Up to 250 MB/s

Storage Scaling:
  Auto Scaling Enabled: Yes
  Maximum Storage: 1000 GB
  Scaling Threshold: 90% utilization
  Scaling Increment: 20% or 10GB (whichever is greater)
```

## 🔧 Connection Management

### Connection Pooling Strategy
```yaml
Application-Level Connection Pool:
  Pool Size: 20-30 connections per application instance
  Max Pool Size: 50 connections
  Min Pool Size: 5 connections
  Connection Timeout: 30 seconds
  Idle Timeout: 10 minutes
  
Database-Level Limits:
  max_connections: 405
  max_user_connections: 400
  Reserved connections: 5 (for admin access)
  
Connection Distribution:
  Primary Database (writes): 30% of connections
  Read Replicas (reads): 70% of connections
  Monitoring/Admin: 5% reserved
```

### SSL Configuration
```yaml
SSL Configuration:
  ssl_ca: /rdsdbdata/rds-ca-2019-root.pem
  ssl_cert: /rdsdbdata/server-cert.pem
  ssl_key: /rdsdbdata/server-key.pem
  
Client Connection:
  require_secure_transport: ON
  ssl_cipher: ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256
  tls_version: TLSv1.2,TLSv1.3
```

## 📈 Monitoring and Alerting

### Custom CloudWatch Metrics
```yaml
Custom Metrics:
  - Query Response Time
  - Connection Pool Utilization
  - Cache Hit Ratio
  - Replica Lag Distribution
  - Error Rate by Query Type

Alarms:
  High CPU:
    Threshold: 80%
    Period: 5 minutes
    Evaluation Periods: 2
    Action: SNS notification + auto-scaling trigger
    
  High Connection Count:
    Threshold: 350 connections (85% of max)
    Period: 5 minutes
    Evaluation Periods: 1
    Action: SNS notification
    
  High Replica Lag:
    Threshold: 10 seconds
    Period: 1 minute
    Evaluation Periods: 2
    Action: SNS notification + fallback to primary
    
  Low Free Storage:
    Threshold: 20% remaining
    Period: 5 minutes
    Evaluation Periods: 1
    Action: Auto-scaling + notification
```

## 🚨 Disaster Recovery Configuration

### Backup Strategy
```yaml
Automated Backup Schedule:
  Daily Backup: 03:00 UTC
  Retention: 2 days
  Encryption: AES-256 with aws/rds key
  Cross-Region Copy: No (cost optimization)

Manual Snapshot Strategy:
  Pre-deployment Snapshots: Before major changes
  Monthly Snapshots: Long-term retention
  Testing Snapshots: For DR testing

Point-in-Time Recovery:
  Granularity: 5-minute intervals
  Recovery Window: 2 days
  Storage: S3 in same region
```

### Failover Procedures
```yaml
Multi-AZ Failover:
  Trigger Conditions:
    - Primary AZ failure
    - Primary instance failure
    - Planned maintenance
    - User-initiated failover
  
  Failover Process:
    1. DNS endpoint switch (automatic)
    2. Connection drainage (60-120 seconds)
    3. Standby promotion
    4. Application reconnection
  
  Recovery Time Objective (RTO): 2 minutes
  Recovery Point Objective (RPO): 0 (synchronous replication)

Read Replica Promotion:
  Use Case: Primary region failure
  Manual Process:
    1. Stop application writes
    2. Promote read replica
    3. Update application configuration
    4. Resume operations
  
  RTO: 15-30 minutes
  RPO: 2-5 minutes (last committed transaction)
```

---
*This appendix is part of the XYZ Corporation Multi-Database Architecture Case Study project.*