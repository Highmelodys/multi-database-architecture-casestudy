# Disaster Recovery Procedures
## XYZ Corporation Multi-Database Architecture

### 📋 Overview
This document outlines disaster recovery procedures for the XYZ Corporation multi-database architecture, including RDS, DynamoDB, and ElastiCache services.

### 🎯 Recovery Objectives
- **RTO (Recovery Time Objective)**: 15 minutes
- **RPO (Recovery Point Objective)**: 2 days for RDS, Point-in-time for DynamoDB
- **Availability Target**: 99.9% uptime

---

## 🗄️ RDS MySQL Disaster Recovery

### Automated Failover (Multi-AZ)
```bash
# Check RDS instance status
aws rds describe-db-instances --db-instance-identifier xyz-corp-primary-db

# Monitor failover status
aws rds describe-events --source-identifier xyz-corp-primary-db --source-type db-instance
```

### Manual Failover Procedures
```bash
# Force failover for testing or emergency
aws rds failover-db-cluster --db-cluster-identifier xyz-corp-primary-db

# Promote read replica (if primary fails)
aws rds promote-read-replica --db-instance-identifier xyz-corp-replica-west
```

### Point-in-Time Recovery
```bash
# Restore RDS from specific time
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier xyz-corp-primary-db \
    --target-db-instance-identifier xyz-corp-restored-db \
    --restore-time 2024-01-15T10:30:00.000Z \
    --db-instance-class db.t3.medium \
    --no-multi-az
```

### Cross-Region Recovery
```bash
# Create cross-region snapshot
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier xyz-corp-primary-db-snapshot-2024-01-15 \
    --target-db-snapshot-identifier xyz-corp-recovery-snapshot \
    --source-region us-east-1 \
    --target-region us-west-2

# Restore from cross-region snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier xyz-corp-disaster-recovery \
    --db-snapshot-identifier xyz-corp-recovery-snapshot \
    --db-instance-class db.t3.medium
```

---

## 🔥 DynamoDB Disaster Recovery

### Point-in-Time Recovery
```bash
# Restore DynamoDB table to specific point in time
aws dynamodb restore-table-to-point-in-time \
    --source-table-name xyz-corp-realtime-analytics \
    --target-table-name xyz-corp-realtime-analytics-restored \
    --restore-date-time 2024-01-15T10:30:00

# Check restore status
aws dynamodb describe-table --table-name xyz-corp-realtime-analytics-restored
```

### Backup Restoration
```bash
# List available backups
aws dynamodb list-backups --table-name xyz-corp-realtime-analytics

# Restore from backup
aws dynamodb restore-table-from-backup \
    --target-table-name xyz-corp-realtime-analytics-dr \
    --backup-arn arn:aws:dynamodb:us-east-1:123456789012:table/xyz-corp-realtime-analytics/backup/01234567890123-12345678
```

### Cross-Region Recovery Setup
```bash
# Enable Global Tables for cross-region replication
aws dynamodb create-global-table \
    --global-table-name xyz-corp-realtime-analytics \
    --replication-group RegionName=us-east-1 RegionName=us-west-2
```

---

## ⚡ ElastiCache Redis Disaster Recovery

### Snapshot Restoration
```bash
# List available snapshots
aws elasticache describe-snapshots --replication-group-id xyz-corp-redis-cluster

# Create new cluster from snapshot
aws elasticache create-replication-group \
    --replication-group-id xyz-corp-redis-dr \
    --replication-group-description "Disaster Recovery Redis Cluster" \
    --snapshot-name xyz-corp-redis-cluster-snapshot-2024-01-15 \
    --cache-node-type cache.t3.micro \
    --num-cache-clusters 2
```

### Cross-Region Backup Restoration
```bash
# Copy snapshot to another region
aws elasticache copy-snapshot \
    --source-snapshot-name xyz-corp-redis-cluster-snapshot-2024-01-15 \
    --target-snapshot-name xyz-corp-redis-dr-snapshot \
    --target-region us-west-2

# Create cluster in DR region
aws elasticache create-replication-group \
    --region us-west-2 \
    --replication-group-id xyz-corp-redis-dr-west \
    --replication-group-description "DR Redis Cluster West" \
    --snapshot-name xyz-corp-redis-dr-snapshot \
    --cache-node-type cache.t3.micro \
    --num-cache-clusters 2
```

---

## 🚨 Emergency Response Procedures

### 1. Database Outage Detection
```bash
# Monitor database health
aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus]'
aws dynamodb describe-table --table-name xyz-corp-realtime-analytics --query 'Table.TableStatus'
aws elasticache describe-replication-groups --query 'ReplicationGroups[*].[ReplicationGroupId,Status]'
```

### 2. Failover Decision Matrix
| Scenario | RDS Action | DynamoDB Action | ElastiCache Action |
|----------|------------|-----------------|-------------------|
| AZ Outage | Automatic Multi-AZ failover | No action required | Redis cluster failover |
| Region Outage | Promote read replica | Switch to Global Tables | Restore from cross-region snapshot |
| Corruption | Point-in-time recovery | Point-in-time recovery | Restore from snapshot |
| Human Error | Restore from snapshot | Restore from backup | Restore from snapshot |

### 3. Communication Plan
```bash
# Send notifications via SNS
aws sns publish \
    --topic-arn arn:aws:sns:us-east-1:123456789012:xyz-corp-dr-notifications \
    --subject "DATABASE DISASTER RECOVERY INITIATED" \
    --message "DR procedure started at $(date). RTO: 15 minutes"
```

---

## 🔧 Recovery Validation

### RDS Recovery Tests
```sql
-- Validate data integrity after recovery
SELECT COUNT(*) as total_records FROM main_table;
SELECT MAX(created_at) as latest_record FROM main_table;

-- Check replication lag (if applicable)
SHOW SLAVE STATUS\G
```

### DynamoDB Recovery Tests
```bash
# Validate table status and item count
aws dynamodb describe-table --table-name xyz-corp-realtime-analytics-restored
aws dynamodb scan --table-name xyz-corp-realtime-analytics-restored --select "COUNT"
```

### ElastiCache Recovery Tests
```bash
# Test Redis connectivity and data
redis-cli -h xyz-corp-redis-dr.abc123.cache.amazonaws.com -p 6379 ping
redis-cli -h xyz-corp-redis-dr.abc123.cache.amazonaws.com -p 6379 info replication
```

---

## 📊 Recovery Monitoring

### CloudWatch Metrics to Monitor
- `DatabaseConnections` (RDS)
- `ConsumedReadCapacityUnits` (DynamoDB)
- `CurrConnections` (ElastiCache)
- `NetworkBytesIn/Out` (All services)

### Custom Dashboards
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/RDS", "DatabaseConnections", "DBInstanceIdentifier", "xyz-corp-primary-db" ],
          [ "AWS/DynamoDB", "ConsumedReadCapacityUnits", "TableName", "xyz-corp-realtime-analytics" ],
          [ "AWS/ElastiCache", "CurrConnections", "CacheClusterId", "xyz-corp-redis-cluster" ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Post-Recovery Service Health"
      }
    }
  ]
}
```

---

## 📝 Recovery Checklist

### Pre-Recovery
- [ ] Assess impact and determine appropriate recovery method
- [ ] Notify stakeholders via SNS/email
- [ ] Document incident details and timeline
- [ ] Verify backup availability and integrity

### During Recovery
- [ ] Execute appropriate recovery procedure
- [ ] Monitor recovery progress via CloudWatch
- [ ] Update DNS/connection strings if necessary
- [ ] Validate service connectivity

### Post-Recovery
- [ ] Perform data integrity checks
- [ ] Update monitoring dashboards
- [ ] Conduct performance validation
- [ ] Update incident documentation
- [ ] Review and improve procedures

---

## 🔄 Automated Recovery Scripts

### Lambda-based Auto-Recovery
```python
import boto3
import json

def lambda_handler(event, context):
    """
    Automated disaster recovery for critical database failures
    """
    rds = boto3.client('rds')
    dynamodb = boto3.client('dynamodb')
    
    # Check for RDS failures and initiate failover
    if event['source'] == 'aws.rds':
        if 'failure' in event['detail-type'].lower():
            # Initiate automatic failover
            response = rds.failover_db_cluster(
                DBClusterIdentifier='xyz-corp-primary-db'
            )
            
    return {
        'statusCode': 200,
        'body': json.dumps('Auto-recovery initiated')
    }
```

### Testing Schedule
- **Weekly**: Automated backup verification
- **Monthly**: Multi-AZ failover test
- **Quarterly**: Cross-region recovery drill
- **Annually**: Full disaster recovery simulation

---

## 📞 Emergency Contacts

### Internal Team
- **Database Administrator**: dba@xyzcorp.com
- **DevOps Team**: devops@xyzcorp.com
- **On-Call Engineer**: +1-555-0123

### AWS Support
- **Business Support**: Case via AWS Console
- **Emergency**: 1-206-266-4064
- **Account ID**: 123456789012

### Recovery SLA
- **Detection**: < 5 minutes
- **Response**: < 10 minutes  
- **Recovery**: < 15 minutes (RTO)
- **Validation**: < 30 minutes

---

*Last Updated: 2024-01-15*  
*Document Version: 2.1*  
*Review Schedule: Quarterly*