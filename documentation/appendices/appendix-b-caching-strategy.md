# 📖 Appendix B - Caching Strategy Details

## 🚀 ElastiCache Redis Configuration

### Primary Redis Cluster (US-East-1)
```yaml
Cluster Configuration:
  Replication Group ID: xyz-corp-redis-primary
  Description: Primary Redis cluster for XYZ Corp multi-database architecture
  Node Type: cache.t3.micro
  Engine: Redis 7.0
  Port: 6379
  
Cluster Mode:
  Enabled: Yes
  Number of Shards: 2
  Replicas per Shard: 1
  Total Nodes: 4 (2 primary + 2 replica)
  
Multi-AZ:
  Enabled: Yes
  Automatic Failover: Yes
  
Network Configuration:
  VPC: vpc-xyz-corp-main
  Subnet Group: xyz-corp-cache-subnet-group
  Security Groups: sg-xyz-cache-001
  
Availability Zones:
  Shard 1: us-east-1a (primary), us-east-1b (replica)
  Shard 2: us-east-1b (primary), us-east-1a (replica)
```

### Regional Redis Clusters

#### US-West-2 Redis Cluster
```yaml
Cluster Configuration:
  Replication Group ID: xyz-corp-redis-west
  Description: West region Redis cluster for XYZ Corp
  Node Type: cache.t3.micro
  Engine: Redis 7.0
  Port: 6379
  
Cluster Mode:
  Enabled: No (single shard for cost optimization)
  Primary Node: 1
  Replica Nodes: 1
  Total Nodes: 2
  
Multi-AZ:
  Enabled: Yes
  Automatic Failover: Yes
  
Network Configuration:
  VPC: vpc-xyz-corp-west
  Subnet Group: xyz-corp-west-cache-subnet-group
  Security Groups: sg-xyz-west-cache-001
  
Availability Zones:
  Primary: us-west-2a
  Replica: us-west-2b
```

#### US-East-2 (Ohio) Redis Cluster
```yaml
Cluster Configuration:
  Replication Group ID: xyz-corp-redis-central
  Description: Central region Redis cluster for XYZ Corp
  Node Type: cache.t3.micro
  Engine: Redis 7.0
  Port: 6379
  
Cluster Mode:
  Enabled: No (single shard for cost optimization)
  Primary Node: 1
  Replica Nodes: 1
  Total Nodes: 2
  
Multi-AZ:
  Enabled: Yes
  Automatic Failover: Yes
  
Network Configuration:
  VPC: vpc-xyz-corp-central
  Subnet Group: xyz-corp-central-cache-subnet-group
  Security Groups: sg-xyz-central-cache-001
  
Availability Zones:
  Primary: us-east-2a
  Replica: us-east-2b
```

## ⚙️ Redis Configuration Parameters

### Memory Management
```redis
# Memory optimization settings
maxmemory 900mb  # 90% of 1GB available on t3.micro
maxmemory-policy allkeys-lru
maxmemory-samples 5

# Memory usage optimization
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Large object handling
proto-max-bulk-len 512mb
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit replica 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
```

### Performance Tuning
```redis
# TCP settings for performance
tcp-keepalive 300
tcp-backlog 511
timeout 300

# Database configuration
databases 16
dbfilename dump.rdb

# Networking
bind 0.0.0.0
port 6379
tcp-nodelay yes

# Threading (Redis 6.0+)
io-threads 2
io-threads-do-reads yes

# Memory defragmentation
activedefrag yes
active-defrag-ignore-bytes 100mb
active-defrag-threshold-lower 10
active-defrag-threshold-upper 100
active-defrag-cycle-min 1
active-defrag-cycle-max 25
```

### Persistence Configuration
```redis
# RDB snapshots (optimized for performance)
save 900 1      # Save if at least 1 key changed in 900 seconds
save 300 10     # Save if at least 10 keys changed in 300 seconds  
save 60 10000   # Save if at least 10000 keys changed in 60 seconds

rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb

# AOF disabled for better performance (data durability handled by RDS)
appendonly no
# If AOF were enabled:
# appendfilename "appendonly.aof"
# appendfsync everysec
# no-appendfsync-on-rewrite no
# auto-aof-rewrite-percentage 100
# auto-aof-rewrite-min-size 64mb
```

### Security Configuration
```redis
# Authentication (handled by AWS ElastiCache)
# requirepass is managed by AWS
# auth-token configuration through AWS console

# Security settings
protected-mode yes
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command DEBUG ""
rename-command CONFIG "CONFIG_xyz_secret_token"

# SSL/TLS (managed by AWS)
# tls-port 6380
# tls-cert-file redis.crt
# tls-key-file redis.key
# tls-ca-cert-file ca.crt
```

## 📊 Caching Patterns and Strategies

### Cache-Aside (Lazy Loading) Pattern
```python
import redis
import json
import hashlib
from datetime import datetime, timedelta
from typing import Optional, Dict, Any

class CacheAsideManager:
    def __init__(self, redis_client: redis.Redis, default_ttl: int = 300):
        self.redis = redis_client
        self.default_ttl = default_ttl
    
    def get_or_set(self, key: str, fetch_function, ttl: Optional[int] = None) -> Any:
        """
        Implements cache-aside pattern with automatic fallback to database
        """
        # Try to get from cache first
        cached_value = self.redis.get(key)
        if cached_value is not None:
            self._track_cache_hit(key)
            return json.loads(cached_value)
        
        # Cache miss - fetch from database
        self._track_cache_miss(key)
        fresh_data = fetch_function()
        
        # Store in cache for future requests
        if fresh_data is not None:
            ttl = ttl or self.default_ttl
            self.redis.setex(key, ttl, json.dumps(fresh_data))
        
        return fresh_data
    
    def invalidate(self, key: str) -> bool:
        """
        Invalidate a specific cache key
        """
        return self.redis.delete(key) > 0
    
    def invalidate_pattern(self, pattern: str) -> int:
        """
        Invalidate multiple keys matching a pattern
        """
        keys = self.redis.keys(pattern)
        if keys:
            return self.redis.delete(*keys)
        return 0
    
    def _track_cache_hit(self, key: str):
        """Track cache hits for monitoring"""
        self.redis.incr(f"stats:cache_hits:{datetime.now().strftime('%Y%m%d%H')}")
    
    def _track_cache_miss(self, key: str):
        """Track cache misses for monitoring"""
        self.redis.incr(f"stats:cache_misses:{datetime.now().strftime('%Y%m%d%H')}")

# Usage examples
def get_device_data(device_id: int, start_time: str, end_time: str) -> Dict:
    cache_key = f"device:{device_id}:data:{start_time}:{end_time}"
    
    def fetch_from_db():
        # Database query logic here
        return database.query_device_data(device_id, start_time, end_time)
    
    return cache_manager.get_or_set(cache_key, fetch_from_db, ttl=600)  # 10 minutes
```

### Write-Through Caching Pattern
```python
class WriteThroughCache:
    def __init__(self, redis_client: redis.Redis, database_client):
        self.redis = redis_client
        self.database = database_client
    
    def update_device_data(self, device_id: int, data: Dict) -> bool:
        """
        Write-through pattern: update database first, then cache
        """
        try:
            # Step 1: Update database
            success = self.database.update_device(device_id, data)
            
            if success:
                # Step 2: Update cache
                cache_key = f"device:{device_id}:current"
                self.redis.setex(cache_key, 600, json.dumps(data))
                
                # Step 3: Invalidate related cached queries
                self._invalidate_device_queries(device_id)
                
                return True
            else:
                return False
                
        except Exception as e:
            # If database update fails, don't update cache
            print(f"Write-through cache error: {e}")
            return False
    
    def _invalidate_device_queries(self, device_id: int):
        """Invalidate related cached queries"""
        patterns = [
            f"device:{device_id}:*",
            f"location:*:devices:*{device_id}*",
            f"stats:device:{device_id}:*"
        ]
        
        for pattern in patterns:
            keys = self.redis.keys(pattern)
            if keys:
                self.redis.delete(*keys)
```

### Write-Behind (Write-Back) Caching Pattern
```python
import asyncio
import time
from collections import defaultdict

class WriteBehindCache:
    def __init__(self, redis_client: redis.Redis, database_client, 
                 batch_size: int = 100, flush_interval: int = 30):
        self.redis = redis_client
        self.database = database_client
        self.batch_size = batch_size
        self.flush_interval = flush_interval
        self.write_buffer = defaultdict(dict)
        self.last_flush = time.time()
        self._start_background_flush()
    
    async def update_device_data(self, device_id: int, data: Dict) -> bool:
        """
        Write-behind pattern: update cache immediately, batch database writes
        """
        try:
            # Step 1: Update cache immediately
            cache_key = f"device:{device_id}:current"
            self.redis.setex(cache_key, 1800, json.dumps(data))  # 30 minutes
            
            # Step 2: Add to write buffer for batched database update
            self.write_buffer[device_id].update(data)
            
            # Step 3: Check if we need to flush immediately
            if (len(self.write_buffer) >= self.batch_size or 
                time.time() - self.last_flush > self.flush_interval):
                await self._flush_to_database()
            
            return True
            
        except Exception as e:
            print(f"Write-behind cache error: {e}")
            return False
    
    async def _flush_to_database(self):
        """Flush batched writes to database"""
        if not self.write_buffer:
            return
        
        try:
            # Batch update database
            updates = dict(self.write_buffer)
            await self.database.batch_update_devices(updates)
            
            # Clear buffer after successful write
            self.write_buffer.clear()
            self.last_flush = time.time()
            
        except Exception as e:
            print(f"Database flush error: {e}")
            # Keep data in buffer for retry
    
    def _start_background_flush(self):
        """Start background task for periodic flushing"""
        async def background_flush():
            while True:
                await asyncio.sleep(self.flush_interval)
                if time.time() - self.last_flush > self.flush_interval:
                    await self._flush_to_database()
        
        asyncio.create_task(background_flush())
```

## 🔑 Cache Key Design Patterns

### Hierarchical Key Structure
```python
class CacheKeyBuilder:
    """
    Standardized cache key builder for consistent naming and easy invalidation
    """
    
    @staticmethod
    def device_data_key(device_id: int, timestamp: str) -> str:
        return f"device:{device_id}:data:{timestamp}"
    
    @staticmethod
    def device_summary_key(device_id: int, period: str) -> str:
        return f"device:{device_id}:summary:{period}"
    
    @staticmethod
    def location_devices_key(location_id: int) -> str:
        return f"location:{location_id}:devices"
    
    @staticmethod
    def user_session_key(user_id: int, session_id: str) -> str:
        return f"user:{user_id}:session:{session_id}"
    
    @staticmethod
    def analytics_key(metric: str, time_window: str, filters: str) -> str:
        filter_hash = hashlib.md5(filters.encode()).hexdigest()[:8]
        return f"analytics:{metric}:{time_window}:{filter_hash}"
    
    @staticmethod
    def get_invalidation_patterns(entity_type: str, entity_id: int) -> list:
        """Get patterns for cache invalidation"""
        patterns = {
            'device': [
                f"device:{entity_id}:*",
                f"location:*:devices:*{entity_id}*",
                f"analytics:*:*:*device*{entity_id}*"
            ],
            'location': [
                f"location:{entity_id}:*",
                f"analytics:*:*:*location*{entity_id}*"
            ],
            'user': [
                f"user:{entity_id}:*",
                f"analytics:*:*:*user*{entity_id}*"
            ]
        }
        return patterns.get(entity_type, [])

# Usage examples
keys = CacheKeyBuilder()

# Cache device data with structured keys
device_data_key = keys.device_data_key(device_id=123, timestamp="2024-01-15T10:00:00")
# Result: "device:123:data:2024-01-15T10:00:00"

# Cache analytics with hash to handle complex filters
analytics_key = keys.analytics_key(
    metric="temperature", 
    time_window="1h", 
    filters='{"location_id": 45, "device_type": "sensor"}'
)
# Result: "analytics:temperature:1h:a1b2c3d4"
```

### TTL Management Strategy
```python
class TTLManager:
    """
    Manages different TTL strategies based on data characteristics
    """
    
    TTL_STRATEGIES = {
        'static_data': 3600,      # 1 hour - rarely changing data
        'user_session': 1800,     # 30 minutes - user sessions
        'device_data': 300,       # 5 minutes - sensor readings
        'analytics': 900,         # 15 minutes - computed analytics
        'real_time': 60,          # 1 minute - real-time data
        'long_term': 86400,       # 24 hours - historical data
    }
    
    @classmethod
    def get_ttl(cls, data_type: str, custom_ttl: Optional[int] = None) -> int:
        """Get appropriate TTL based on data type"""
        if custom_ttl:
            return custom_ttl
        return cls.TTL_STRATEGIES.get(data_type, 300)  # Default 5 minutes
    
    @classmethod
    def set_with_smart_ttl(cls, redis_client: redis.Redis, key: str, 
                          value: Any, data_type: str, custom_ttl: Optional[int] = None):
        """Set cache value with intelligent TTL"""
        ttl = cls.get_ttl(data_type, custom_ttl)
        redis_client.setex(key, ttl, json.dumps(value))
    
    @classmethod
    def extend_ttl_on_access(cls, redis_client: redis.Redis, key: str, 
                            data_type: str, extension_factor: float = 0.5):
        """Extend TTL when data is accessed (sliding expiration)"""
        current_ttl = redis_client.ttl(key)
        if current_ttl > 0:
            base_ttl = cls.get_ttl(data_type)
            new_ttl = min(base_ttl, int(current_ttl * (1 + extension_factor)))
            redis_client.expire(key, new_ttl)
```

## 📈 Cache Performance Optimization

### Connection Pool Configuration
```python
import redis.connection

class OptimizedRedisClient:
    def __init__(self, host: str, port: int = 6379, max_connections: int = 50):
        self.connection_pool = redis.ConnectionPool(
            host=host,
            port=port,
            max_connections=max_connections,
            retry_on_timeout=True,
            socket_timeout=5,
            socket_connect_timeout=5,
            socket_keepalive=True,
            socket_keepalive_options={},
            health_check_interval=30,
            decode_responses=True
        )
        self.client = redis.Redis(connection_pool=self.connection_pool)
    
    def get_pool_stats(self) -> Dict:
        """Get connection pool statistics"""
        pool = self.connection_pool
        return {
            'max_connections': pool.max_connections,
            'created_connections': pool.created_connections,
            'available_connections': len(pool._available_connections),
            'in_use_connections': len(pool._in_use_connections)
        }
```

### Batch Operations Optimization
```python
class BatchCacheOperations:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def batch_get(self, keys: list) -> Dict[str, Any]:
        """Efficiently retrieve multiple cache keys"""
        if not keys:
            return {}
        
        # Use pipeline for batch operations
        pipe = self.redis.pipeline()
        for key in keys:
            pipe.get(key)
        
        results = pipe.execute()
        
        # Parse results and return as dictionary
        parsed_results = {}
        for key, result in zip(keys, results):
            if result is not None:
                try:
                    parsed_results[key] = json.loads(result)
                except json.JSONDecodeError:
                    parsed_results[key] = result
        
        return parsed_results
    
    def batch_set(self, key_value_pairs: Dict[str, Any], ttl: int = 300) -> bool:
        """Efficiently set multiple cache keys"""
        if not key_value_pairs:
            return True
        
        try:
            pipe = self.redis.pipeline()
            for key, value in key_value_pairs.items():
                pipe.setex(key, ttl, json.dumps(value))
            
            pipe.execute()
            return True
            
        except Exception as e:
            print(f"Batch set error: {e}")
            return False
    
    def batch_delete(self, keys: list) -> int:
        """Efficiently delete multiple cache keys"""
        if not keys:
            return 0
        
        return self.redis.delete(*keys)
```

## 📊 Cache Monitoring and Metrics

### Performance Metrics Collection
```python
import time
from dataclasses import dataclass
from typing import Dict, List

@dataclass
class CacheMetrics:
    hits: int = 0
    misses: int = 0
    total_operations: int = 0
    avg_response_time: float = 0.0
    memory_usage: int = 0
    evicted_keys: int = 0

class CacheMonitor:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.start_time = time.time()
    
    def get_cache_metrics(self) -> CacheMetrics:
        """Collect comprehensive cache performance metrics"""
        info = self.redis.info()
        
        hits = info.get('keyspace_hits', 0)
        misses = info.get('keyspace_misses', 0)
        total_ops = hits + misses
        
        hit_rate = (hits / total_ops) if total_ops > 0 else 0
        
        return CacheMetrics(
            hits=hits,
            misses=misses,
            total_operations=total_ops,
            avg_response_time=self._calculate_avg_response_time(),
            memory_usage=info.get('used_memory', 0),
            evicted_keys=info.get('evicted_keys', 0)
        )
    
    def _calculate_avg_response_time(self) -> float:
        """Calculate average response time using sample operations"""
        sample_keys = ['test_key_1', 'test_key_2', 'test_key_3']
        response_times = []
        
        for key in sample_keys:
            start_time = time.time()
            self.redis.get(key)  # This will be a cache miss for test keys
            response_time = time.time() - start_time
            response_times.append(response_time * 1000)  # Convert to milliseconds
        
        return sum(response_times) / len(response_times) if response_times else 0.0
    
    def get_key_distribution(self) -> Dict[str, int]:
        """Analyze cache key patterns and distribution"""
        all_keys = self.redis.keys('*')
        key_patterns = {}
        
        for key in all_keys:
            # Extract pattern (first two levels)
            parts = key.split(':')
            pattern = ':'.join(parts[:2]) if len(parts) >= 2 else key
            key_patterns[pattern] = key_patterns.get(pattern, 0) + 1
        
        return key_patterns
    
    def get_memory_analysis(self) -> Dict[str, Any]:
        """Analyze memory usage patterns"""
        info = self.redis.info('memory')
        
        return {
            'used_memory_human': info.get('used_memory_human'),
            'used_memory_peak_human': info.get('used_memory_peak_human'),
            'memory_fragmentation_ratio': info.get('mem_fragmentation_ratio'),
            'memory_efficiency': (info.get('used_memory_rss', 1) / info.get('used_memory', 1)) * 100
        }

# CloudWatch integration for monitoring
import boto3

class CloudWatchCacheMetrics:
    def __init__(self, cache_monitor: CacheMonitor):
        self.cache_monitor = cache_monitor
        self.cloudwatch = boto3.client('cloudwatch')
    
    def publish_metrics(self, cluster_name: str):
        """Publish cache metrics to CloudWatch"""
        metrics = self.cache_monitor.get_cache_metrics()
        
        metric_data = [
            {
                'MetricName': 'CacheHitRate',
                'Value': (metrics.hits / metrics.total_operations * 100) if metrics.total_operations > 0 else 0,
                'Unit': 'Percent',
                'Dimensions': [{'Name': 'CacheCluster', 'Value': cluster_name}]
            },
            {
                'MetricName': 'CacheResponseTime',
                'Value': metrics.avg_response_time,
                'Unit': 'Milliseconds',
                'Dimensions': [{'Name': 'CacheCluster', 'Value': cluster_name}]
            },
            {
                'MetricName': 'CacheMemoryUsage',
                'Value': metrics.memory_usage,
                'Unit': 'Bytes',
                'Dimensions': [{'Name': 'CacheCluster', 'Value': cluster_name}]
            },
            {
                'MetricName': 'CacheEvictedKeys',
                'Value': metrics.evicted_keys,
                'Unit': 'Count',
                'Dimensions': [{'Name': 'CacheCluster', 'Value': cluster_name}]
            }
        ]
        
        try:
            self.cloudwatch.put_metric_data(
                Namespace='XYZCorp/Cache',
                MetricData=metric_data
            )
        except Exception as e:
            print(f"Failed to publish metrics to CloudWatch: {e}")
```

## 🚨 Cache Failure Handling and Resilience

### Circuit Breaker Pattern
```python
import time
from enum import Enum
from typing import Callable, Any

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open" 
    HALF_OPEN = "half_open"

class CacheCircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 60):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = CircuitState.CLOSED
    
    def call(self, cache_operation: Callable, fallback_operation: Callable, *args, **kwargs) -> Any:
        """Execute cache operation with circuit breaker protection"""
        
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.failure_count = 0
            else:
                # Circuit is open, use fallback
                return fallback_operation(*args, **kwargs)
        
        try:
            result = cache_operation(*args, **kwargs)
            
            if self.state == CircuitState.HALF_OPEN:
                # Successful call in half-open state, close circuit
                self.state = CircuitState.CLOSED
                self.failure_count = 0
            
            return result
            
        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()
            
            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
            
            print(f"Cache operation failed: {e}, using fallback")
            return fallback_operation(*args, **kwargs)

# Usage example
cache_breaker = CacheCircuitBreaker()

def get_device_data_with_fallback(device_id: int):
    def cache_operation():
        return cache_client.get(f"device:{device_id}:data")
    
    def fallback_operation():
        return database.get_device_data(device_id)
    
    return cache_breaker.call(cache_operation, fallback_operation)
```

### Cache Warming Strategies
```python
class CacheWarmer:
    def __init__(self, redis_client: redis.Redis, database_client):
        self.redis = redis_client
        self.database = database_client
    
    async def warm_device_cache(self, device_ids: List[int]):
        """Pre-populate cache with frequently accessed device data"""
        for device_id in device_ids:
            try:
                # Fetch from database
                device_data = await self.database.get_device_data(device_id)
                
                if device_data:
                    # Store in cache with appropriate TTL
                    cache_key = f"device:{device_id}:current"
                    self.redis.setex(cache_key, 1800, json.dumps(device_data))
                    
                    # Also cache common query patterns
                    summary_data = await self.database.get_device_summary(device_id)
                    summary_key = f"device:{device_id}:summary:24h"
                    self.redis.setex(summary_key, 3600, json.dumps(summary_data))
                    
            except Exception as e:
                print(f"Cache warming failed for device {device_id}: {e}")
    
    async def warm_analytics_cache(self):
        """Pre-populate cache with common analytics queries"""
        common_queries = [
            {'metric': 'temperature', 'time_window': '1h'},
            {'metric': 'humidity', 'time_window': '1h'},
            {'metric': 'device_count', 'time_window': '24h'}
        ]
        
        for query in common_queries:
            try:
                result = await self.database.get_analytics(**query)
                cache_key = f"analytics:{query['metric']}:{query['time_window']}"
                self.redis.setex(cache_key, 900, json.dumps(result))  # 15 minutes
                
            except Exception as e:
                print(f"Analytics cache warming failed: {e}")
```

---
*This appendix is part of the XYZ Corporation Multi-Database Architecture Case Study project.*