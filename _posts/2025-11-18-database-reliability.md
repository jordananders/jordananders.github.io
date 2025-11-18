---
layout: default
title:  "Database Reliability Engineering: PostgreSQL, Redis, and Vector Databases"
date:   2025-11-18 12:00:00
categories: DevOps Database Reliability PostgreSQL
---

Database reliability is critical for ML systems. Feature stores, model registries, and vector databases all require high availability and performance tuning.

## PostgreSQL High Availability

### Patroni Configuration

```yaml
# patroni.yml
scope: ml-database
name: pg-node-1

restapi:
  listen: 0.0.0.0:8008
  connect_address: pg-node-1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        effective_cache_size: 12GB
        maintenance_work_mem: 1GB
        checkpoint_completion_target: 0.9
        wal_buffers: 64MB
        default_statistics_target: 100
        random_page_cost: 1.1
        effective_io_concurrency: 200
        work_mem: 20MB
        min_wal_size: 1GB
        max_wal_size: 4GB
        max_worker_processes: 8
        max_parallel_workers_per_gather: 4
        max_parallel_workers: 8

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 0.0.0.0/0 md5
    - host all all 0.0.0.0/0 md5

postgresql:
  listen: 0.0.0.0:5432
  connect_address: pg-node-1:5432
  data_dir: /var/lib/postgresql/data
  bin_dir: /usr/lib/postgresql/15/bin
  authentication:
    replication:
      username: replicator
      password: ${REPLICATOR_PASSWORD}
    superuser:
      username: postgres
      password: ${POSTGRES_PASSWORD}
```

### PgBouncer Connection Pooling

```ini
; pgbouncer.ini
[databases]
mldb = host=pg-primary port=5432 dbname=mldb
mldb_readonly = host=pg-replica port=5432 dbname=mldb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

; Pool settings
pool_mode = transaction
default_pool_size = 20
min_pool_size = 5
reserve_pool_size = 5
max_client_conn = 1000

; Timeouts
server_idle_timeout = 600
server_lifetime = 3600
client_idle_timeout = 0
query_timeout = 0
```

### Query Performance Monitoring

```sql
-- Enable pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- Top slow queries
SELECT
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS mean_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER())::numeric, 2) AS percent_total,
    query
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Table bloat analysis
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
    pg_size_pretty(pg_table_size(schemaname || '.' || tablename)) as table_size,
    pg_size_pretty(pg_indexes_size(schemaname || '.' || tablename)) as index_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC;

-- Index usage
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

## Redis for Feature Store

### Redis Cluster

```yaml
# redis-cluster.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta1
kind: RedisCluster
metadata:
  name: feature-store
spec:
  clusterSize: 3
  kubernetesConfig:
    image: redis:7.2
    resources:
      requests:
        cpu: 1000m
        memory: 4Gi
      limits:
        cpu: 2000m
        memory: 8Gi
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
  redisExporter:
    enabled: true
    image: oliver006/redis_exporter:latest
```

### Feature Store Client

```python
# feature_store.py
import redis
from redis.cluster import RedisCluster
import json
import hashlib

class RedisFeatureStore:
    def __init__(self, nodes: list):
        self.client = RedisCluster(
            startup_nodes=nodes,
            decode_responses=True,
            skip_full_coverage_check=True
        )

    def set_features(self, entity_id: str, features: dict,
                     ttl_seconds: int = 3600):
        """Store features for an entity"""
        key = f"features:{entity_id}"
        pipeline = self.client.pipeline()

        # Store features
        pipeline.hset(key, mapping=features)
        pipeline.expire(key, ttl_seconds)

        # Store feature version
        version = self._compute_version(features)
        pipeline.set(f"{key}:version", version, ex=ttl_seconds)

        pipeline.execute()

    def get_features(self, entity_id: str,
                     feature_names: list = None) -> dict:
        """Get features for an entity"""
        key = f"features:{entity_id}"

        if feature_names:
            values = self.client.hmget(key, feature_names)
            return dict(zip(feature_names, values))
        else:
            return self.client.hgetall(key)

    def batch_get_features(self, entity_ids: list,
                          feature_names: list) -> list[dict]:
        """Batch get features for multiple entities"""
        pipeline = self.client.pipeline()

        for entity_id in entity_ids:
            key = f"features:{entity_id}"
            pipeline.hmget(key, feature_names)

        results = pipeline.execute()

        return [
            dict(zip(feature_names, values))
            for values in results
        ]

    def _compute_version(self, features: dict) -> str:
        content = json.dumps(features, sort_keys=True)
        return hashlib.md5(content.encode()).hexdigest()
```

### Redis Sentinel

```yaml
# redis-sentinel.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta1
kind: RedisSentinel
metadata:
  name: feature-store-sentinel
spec:
  clusterSize: 3
  kubernetesConfig:
    image: redis:7.2
  redisSentinelConfig:
    redisReplicationName: feature-store
    quorum: "2"
    parallelSyncs: "1"
    failoverTimeout: "180000"
    downAfterMilliseconds: "30000"
```

## Vector Database (pgvector)

### Setup pgvector

```sql
-- Enable extension
CREATE EXTENSION vector;

-- Create embeddings table
CREATE TABLE model_embeddings (
    id SERIAL PRIMARY KEY,
    model_name VARCHAR(255) NOT NULL,
    version VARCHAR(50) NOT NULL,
    embedding vector(1536),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create index for similarity search
CREATE INDEX ON model_embeddings
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Or use HNSW for better accuracy
CREATE INDEX ON model_embeddings
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

### Vector Search Operations

```python
# vector_store.py
import psycopg2
from pgvector.psycopg2 import register_vector

class VectorStore:
    def __init__(self, connection_string: str):
        self.conn = psycopg2.connect(connection_string)
        register_vector(self.conn)

    def insert_embedding(self, text: str, embedding: list,
                        metadata: dict = None):
        """Insert embedding into store"""
        with self.conn.cursor() as cur:
            cur.execute(
                """
                INSERT INTO model_embeddings
                (model_name, version, embedding, metadata)
                VALUES (%s, %s, %s, %s)
                """,
                ('text-embedding-ada-002', 'v1', embedding, metadata)
            )
        self.conn.commit()

    def similarity_search(self, query_embedding: list,
                         top_k: int = 10) -> list:
        """Find similar embeddings"""
        with self.conn.cursor() as cur:
            cur.execute(
                """
                SELECT id, metadata, 1 - (embedding <=> %s) as similarity
                FROM model_embeddings
                ORDER BY embedding <=> %s
                LIMIT %s
                """,
                (query_embedding, query_embedding, top_k)
            )
            return cur.fetchall()

    def filtered_search(self, query_embedding: list,
                       filter_metadata: dict, top_k: int = 10) -> list:
        """Similarity search with metadata filter"""
        with self.conn.cursor() as cur:
            cur.execute(
                """
                SELECT id, metadata, 1 - (embedding <=> %s) as similarity
                FROM model_embeddings
                WHERE metadata @> %s
                ORDER BY embedding <=> %s
                LIMIT %s
                """,
                (query_embedding, filter_metadata, query_embedding, top_k)
            )
            return cur.fetchall()
```

## Backup and Recovery

### pg_basebackup

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p $BACKUP_DIR

# Full backup
pg_basebackup -h pg-primary -U replicator \
  -D $BACKUP_DIR \
  -Ft -z -P \
  --checkpoint=fast \
  --wal-method=stream

# Upload to S3
aws s3 sync $BACKUP_DIR s3://db-backups/pg/$BACKUP_DIR

# Cleanup old backups
find /backups -type d -mtime +7 -exec rm -rf {} +
```

### Point-in-Time Recovery

```bash
# Restore to specific time
pg_restore -h localhost -U postgres \
  -d mldb_restored \
  --target-time="2024-01-15 14:30:00" \
  /backups/mldb_backup.dump
```

## Monitoring

### PostgreSQL Exporter

```yaml
# postgres-exporter.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-exporter-queries
data:
  queries.yaml: |
    pg_replication:
      query: "SELECT CASE WHEN pg_is_in_recovery() THEN 0 ELSE 1 END AS is_primary,
              COALESCE(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn), 0) AS lag_bytes
              FROM pg_stat_replication"
      metrics:
        - is_primary:
            usage: "GAUGE"
            description: "1 if primary, 0 if replica"
        - lag_bytes:
            usage: "GAUGE"
            description: "Replication lag in bytes"

    pg_database_size:
      query: "SELECT datname, pg_database_size(datname) as size_bytes FROM pg_database"
      metrics:
        - datname:
            usage: "LABEL"
        - size_bytes:
            usage: "GAUGE"
            description: "Database size in bytes"
```

### Prometheus Alerts

```yaml
# database-alerts.yaml
groups:
  - name: database-alerts
    rules:
      - alert: PostgreSQLDown
        expr: pg_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL instance down"

      - alert: ReplicationLag
        expr: pg_replication_lag_bytes > 100000000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag exceeds 100MB"

      - alert: ConnectionPoolExhausted
        expr: pgbouncer_pools_sv_active / pgbouncer_pools_size > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connection pool nearly exhausted"

      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis memory usage above 90%"
```

## Best Practices

1. **High availability**: Use Patroni or similar for auto-failover
2. **Connection pooling**: PgBouncer for PostgreSQL
3. **Query optimization**: Index properly, analyze slow queries
4. **Backup strategy**: Regular backups with tested recovery
5. **Monitoring**: Track replication lag, connections, performance
6. **Vector indexing**: Choose IVFFlat vs HNSW based on needs
7. **TTL management**: Clean expired data in Redis
8. **Capacity planning**: Monitor growth, plan scaling

## Resources

- [Patroni Documentation](https://patroni.readthedocs.io/)
- [pgvector Guide](https://github.com/pgvector/pgvector)
- [Redis Cluster Tutorial](https://redis.io/docs/management/scaling/)
- [PostgreSQL Performance](https://wiki.postgresql.org/wiki/Performance_Optimization)

---

*Questions about database reliability? [Let me know](mailto:jordan@jordananderson.us).*
