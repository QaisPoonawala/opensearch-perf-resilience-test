# OpenSearch Resilience Testing Guide

This guide provides step-by-step instructions for conducting resilience tests on your OpenSearch deployment.

## Test Categories

### 1. OpenSearch Node Failure Tests

#### Test 1.1: OpenSearch Data Node Failure
```bash
# 1. Identify target node
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes?v
NODE_ID=$(curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes | grep data | head -1 | awk '{print $1}')

# 2. Check shard allocation
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?v

# 3. Start baseline performance test
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=geonames \
  --workload-params "bulk_size:5000,number_of_replicas:1" \
  --test-iterations=1

# 4. Monitor pre-failure metrics and state
## Node stats
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/$NODE_ID/stats | jq

## Node's shard allocation
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?h=index,shard,prirep,state,node | grep $NODE_ID

## Node's segment info
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/$NODE_ID/stats/indices/segments | jq

## Node's current tasks
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_tasks?nodes=$NODE_ID | jq

# 5. Pre-failure preparation
## Disable shard allocation temporarily
curl -X PUT -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/settings -H 'Content-Type: application/json' -d '{
  "persistent": {
    "cluster.routing.allocation.enable": "none"
  }
}'

# 6. Simulate node failure (multiple methods)

## Method 1: EC2 Instance Stop
aws ec2 stop-instances --instance-ids INSTANCE_ID

## Method 2: Network Partition
aws ec2 modify-instance-attribute \
  --instance-id INSTANCE_ID \
  --groups ISOLATED_SECURITY_GROUP_ID

## Method 3: Process Kill (if you have SSH access)
ssh ec2-user@NODE_IP "sudo systemctl stop opensearch"

# 7. Monitor recovery process
## Re-enable shard allocation
curl -X PUT -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/settings -H 'Content-Type: application/json' -d '{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}'

## Monitor cluster health
watch -n 5 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq'

## Monitor recovery progress
watch -n 5 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/recovery'

## Monitor shard movement
watch -n 5 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards'

## Check cluster state
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/state | jq '.routing_nodes'

## Monitor recovery speed
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/indices/recovery | jq

# 7. Run continuous performance test during recovery
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=geonames \
  --test-iterations=5

# 8. Recovery Optimization (if needed):
## Adjust recovery settings
curl -X PUT -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/settings -H 'Content-Type: application/json' -d '{
  "persistent": {
    "indices.recovery.max_bytes_per_sec": "100mb",
    "indices.recovery.concurrent_streams": "5",
    "cluster.routing.allocation.node_concurrent_recoveries": "4",
    "cluster.routing.allocation.node_initial_primaries_recoveries": "4"
  }
}'

# 9. Detailed validation:
## Check cluster health
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq

## Verify shard allocation
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?v

## Check for unassigned shards
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?h=index,shard,prirep,state,unassigned.reason | grep UNASSIGNED

## Verify indices
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/indices?v

# 9. Compare performance metrics
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/indices?human | jq
```

#### Test 1.2: OpenSearch Hot/Warm Node Failures
```bash
# 1. Identify hot and warm nodes
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodeattrs | grep box_type

# 2. Check data distribution
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/indices?v&h=index,docs.count,store.size,pri,rep,box_type

# 3. Simulate hot node failure
HOT_NODE=$(curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodeattrs | grep box_type=hot | awk '{print $1}' | head -1)
aws ec2 stop-instances --instance-ids $HOT_NODE

# 4. Monitor hot data migration
watch -n 5 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?v&h=index,shard,prirep,state,node,box_type'

# 5. Test warm tier access
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=geonames \
  --workload-params "bulk_size:0,number_of_replicas:1" \
  --test-iterations=2
```

#### Test 1.3: Multiple Data Node Failures
```bash
# 1. Identify target nodes
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes?v
NODE_IDS=$(curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes | grep data | head -2 | awk '{print $1}')

# 2. Start load generation
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=http_logs \
  --workload-params "bulk_size:1000,number_of_replicas:2" \
  --test-iterations=1 &

# 3. Simulate cascading failures
for node in $NODE_IDS; do
  echo "Failing node: $node"
  # Stop node
  aws ec2 stop-instances --instance-ids $node
  
  # Wait and monitor
  sleep 300
  
  # Check cluster health
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq
  
  # Check shard allocation
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?v
done

# 4. Monitor recovery
watch -n 10 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq'
```

#### Test 1.4: Master Node Election and Failure
```bash
# 1. Identify current master
MASTER_NODE=$(curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/master | awk '{print $1}')
echo "Current master: $MASTER_NODE"

# 2. Monitor master elections
watch -n 1 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/master'

# 3. Start continuous operations
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=geonames \
  --workload-params "bulk_size:100,number_of_replicas:1" \
  --test-iterations=1 &

# 4. Simulate master failure
aws ec2 stop-instances --instance-ids MASTER_INSTANCE_ID

# 5. Monitor election and recovery
watch -n 1 'echo "Master Node:"; \
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/master; \
  echo "\nCluster Health:"; \
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq'

# 6. Verify cluster operations and master node behavior
## Check discovery settings
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/settings | jq '.persistent.discovery'

## Verify minimum master nodes setting
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/settings | jq '.persistent."discovery.zen.minimum_master_nodes"'

## Check cluster voting configuration
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/state?filter_path=metadata.cluster_coordination | jq

## Check master node stats
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/_master/stats | jq

## Monitor cluster state changes
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/state?filter_path=metadata.cluster_uuid,metadata.cluster_coordination.last_committed_config | jq

## Verify no pending tasks
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/pending_tasks | jq

## Check node roles and configuration
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/_all/settings | jq
```

#### Test 1.5: Coordinating Node Failure and Load Balancing
```bash
# 1. Identify coordinating nodes
COORD_NODES=$(curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes | grep coordinating | awk '{print $1}')

# 2. Start search-heavy workload
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=pmc \
  --test-iterations=3 &

# 3. Monitor search performance
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/indices?human | jq '.nodes[] | {name, search: .indices.search}'

# 4. Fail coordinating node
aws ec2 stop-instances --instance-ids COORD_INSTANCE_ID

# 5. Monitor query routing and load balancing
## Monitor query routing
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/transport | jq '.nodes[] | {name, rx_count: .transport.rx_count, tx_count: .transport.tx_count}'

## Check query load distribution
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/indices/search | jq '.nodes[] | {name, query_total: .indices.search.query_total, query_time: .indices.search.query_time_in_millis}'

## Monitor thread pools
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/thread_pool | jq '.nodes[] | {name, search: .thread_pool.search}'

## Check connection distribution
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/http | jq '.nodes[] | {name, current_open: .http.current_open, total_opened: .http.total_opened}'
```

### 2. Multi-Zone Resilience Tests

#### Test 2.1: Single AZ Failure
```bash
# 1. Identify nodes in target AZ
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes?v

# 2. Start baseline test
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=http_logs \
  --workload-params "bulk_size:1000,number_of_replicas:1" \
  --test-iterations=1

# 3. Simulate AZ failure using AWS Console:
# - Go to EC2 > Instances
# - Filter instances by AZ
# - Stop all OpenSearch nodes in the target AZ
# - Monitor reallocation and recovery

# 4. Run continuous queries during failover
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=geonames \
  --test-iterations=5

# 5. Validate recovery:
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/allocation/explain
```

#### Test 2.2: Zone-to-Zone Latency
```bash
# 1. Create test index with zone awareness
curl -X PUT -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/test-latency -H 'Content-Type: application/json' -d '{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}'

# 2. Test cross-zone operations
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=geonames \
  --workload-params "bulk_size:0,number_of_replicas:1,index_settings.routing.allocation.awareness.attributes:zone" \
  --test-iterations=3

# 3. Monitor zone-specific metrics
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/indices?human | jq '.nodes[] | {name, zone: .attributes.zone, stats: .indices.search}'
```

#### Test 2.3: Zone Isolation
```bash
# 1. Create security group rules to simulate network partition
aws ec2 create-security-group \
  --group-name zone-isolation-test \
  --description "Temporary group for zone isolation testing"

# 2. Apply restrictive rules to isolate one AZ
aws ec2 authorize-security-group-ingress \
  --group-id SECURITY_GROUP_ID \
  --protocol tcp \
  --port 443 \
  --cidr SPECIFIC_AZ_CIDR

# 3. Run tests during isolation
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=http_logs \
  --workload-params "bulk_size:1000,number_of_replicas:1" \
  --test-iterations=2

# 4. Monitor cluster behavior
watch -n 5 'curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq'
```

### 3. Network Partition Tests

#### Test 3.1: Network Latency
```bash
# 1. Add latency between AZs using AWS Network Manager
# 2. Run performance test
opensearch-benchmark execute-test \
  --target-hosts=OPENSEARCH_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=pmc \
  --workload-params "bulk_size:0,number_of_replicas:1" \
  --test-iterations=3
```

### 4. Multi-Region Resilience Tests

#### Test 4.1: Cross-Region Replication
```bash
# 1. Setup cross-region replication
# 2. Run baseline test on primary
./run-dr-test.sh baseline

# 3. Monitor replication lag
watch -n 30 'curl -s -u admin:Admin123! --insecure https://PRIMARY_ENDPOINT/_cat/indices?v; echo "---"; curl -s -u admin:Admin123! --insecure https://DR_ENDPOINT/_cat/indices?v'

# 4. Test replication performance
opensearch-benchmark execute-test \
  --target-hosts=PRIMARY_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=http_logs \
  --workload-params "bulk_size:5000,number_of_replicas:1"
```

#### Test 4.2: Region Failover
```bash
# 1. Run load on primary region
opensearch-benchmark execute-test \
  --target-hosts=PRIMARY_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=default \
  --workload=geonames \
  --workload-params "bulk_size:1000,number_of_replicas:1" &

# 2. Simulate primary region failure
aws ec2 stop-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Name,Values=OpenSearch-Benchmark-*" --query "Reservations[].Instances[].InstanceId" --output text)

# 3. Execute failover procedure
./run-dr-test.sh failover

# 4. Validate DR cluster and measure RPO/RTO:
curl -s -u admin:Admin123! --insecure https://DR_ENDPOINT/_cat/indices?v
curl -s -u admin:Admin123! --insecure https://DR_ENDPOINT/_cat/recovery?v
```

#### Test 4.3: Cross-Region Performance
```bash
# 1. Test primary region performance
opensearch-benchmark execute-test \
  --target-hosts=PRIMARY_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=pmc \
  --test-iterations=3

# 2. Test DR region performance
opensearch-benchmark execute-test \
  --target-hosts=DR_ENDPOINT \
  --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
  --test-procedure=search-only \
  --workload=pmc \
  --test-iterations=3

# 3. Compare latencies and throughput
python3 -c '
import json
def load_results(file):
    with open(file) as f:
        return json.load(f)
primary = load_results("primary_results.json")
dr = load_results("dr_results.json")
print("Latency comparison:")
print(f"Primary: {primary['results']['latency']}")
print(f"DR: {dr['results']['latency']}")
'
```

## Multi-Zone and Multi-Region Monitoring Guide

### Zone-Specific Monitoring
```bash
# Create zone monitoring script
cat > monitor-zones.sh << 'EOL'
#!/bin/bash
while true; do
  echo "=== Zone Distribution ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/allocation?v

  echo -e "\n=== Cross-Zone Operations ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/transport?human | jq '.nodes[] | {name, zone: .attributes.zone, cross_zone_ops: .transport.rx_count}'

  echo -e "\n=== Zone Health ==="
  for zone in $(curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodeattrs | grep zone | awk '{print $3}' | sort -u); do
    echo "Zone: $zone"
    curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats?human | jq --arg zone "$zone" '.nodes[] | select(.attributes.zone == $zone) | {name, heap_used: .jvm.mem.heap_used_percent, cpu: .process.cpu.percent}'
  done

  sleep 10
done
EOL
chmod +x monitor-zones.sh
```

### Region-Specific Monitoring
```bash
# Create region monitoring script
cat > monitor-regions.sh << 'EOL'
#!/bin/bash
while true; do
  echo "=== Primary Region ==="
  curl -s -u admin:Admin123! --insecure https://PRIMARY_ENDPOINT/_cluster/health | jq

  echo -e "\n=== DR Region ==="
  curl -s -u admin:Admin123! --insecure https://DR_ENDPOINT/_cluster/health | jq

  echo -e "\n=== Replication Status ==="
  curl -s -u admin:Admin123! --insecure https://PRIMARY_ENDPOINT/_cat/indices?v
  echo "---"
  curl -s -u admin:Admin123! --insecure https://DR_ENDPOINT/_cat/indices?v

  sleep 30
done
EOL
chmod +x monitor-regions.sh
```

### 1. Cluster Health Monitoring
```bash
# Create monitoring script
cat > monitor-cluster.sh << 'EOL'
#!/bin/bash
while true; do
  echo "=== Cluster Health ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq
  
  echo "=== Node Status ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/nodes?v
  
  echo "=== Shard Allocation ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?v
  
  sleep 5
done
EOL
chmod +x monitor-cluster.sh
```

### 2. Performance Monitoring
```bash
# Create performance monitor
cat > monitor-performance.sh << 'EOL'
#!/bin/bash
while true; do
  echo "=== Thread Pools ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/thread_pool?v
  
  echo "=== JVM Stats ==="
  curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_nodes/stats/jvm | jq
  
  sleep 10
done
EOL
chmod +x monitor-performance.sh
```

### 3. CloudWatch Metrics to Monitor
- CPUUtilization
- FreeStorageSpace
- JVMMemoryPressure
- MasterReachableFromNode
- AutomatedSnapshotFailure
- ShardMigrationInProgress

## Validation Checklist

### 1. Cluster Health
- [ ] Cluster status is green
- [ ] All nodes are present and healthy
- [ ] No unassigned shards
- [ ] Master node is elected and stable

### 2. Data Integrity
- [ ] Document count matches pre-test state
- [ ] Search queries return expected results
- [ ] No corrupt indices

### 3. Performance
- [ ] Query latency within acceptable range
- [ ] Indexing throughput meets requirements
- [ ] JVM heap usage normal
- [ ] No thread pool rejections

### 4. Recovery
- [ ] All shards properly allocated
- [ ] Replication working correctly
- [ ] No stuck recoveries
- [ ] Recovery time within SLA

## Test Result Template

```markdown
# Resilience Test Report

## Test Details
- Test Type: [Node Failure/AZ Failure/Network Partition/DR]
- Date: YYYY-MM-DD
- Duration: HH:MM:SS
- Configuration: [Standard/Minimal/Coordinator]

## Initial State
- Cluster Health: [Green/Yellow/Red]
- Node Count: X
- Shard Count: X
- Document Count: X

## Test Execution
1. Action: [Description of test action]
   - Time: HH:MM:SS
   - Impact: [Observed immediate impact]

2. Recovery:
   - Start Time: HH:MM:SS
   - End Time: HH:MM:SS
   - Duration: MM:SS

## Metrics
- Recovery Time: MM:SS
- Data Loss: [None/X documents]
- Performance Impact: X%
- Error Rate: X%

## Validation Results
- [ ] Cluster Health Restored
- [ ] Data Integrity Verified
- [ ] Performance Returned to Baseline
- [ ] No Lingering Issues

## Issues Identified
1. [Issue description]
   - Impact:
   - Resolution:

## Recommendations
1. [Recommendation]
   - Priority: [High/Medium/Low]
   - Implementation:
```

## Common Issues and Solutions

### 1. Slow Recovery
- Check disk I/O
- Verify network bandwidth
- Adjust recovery throttling
- Monitor JVM heap usage

### 2. Stuck Shards
```bash
# Check shard allocation
curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/allocation/explain | jq

# Force allocation if needed
curl -X POST -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/reroute?retry_failed=true
```

### 3. Performance Degradation
- Check for hot nodes
- Verify shard distribution
- Monitor garbage collection
- Check for slow queries

## Best Practices

1. Test Preparation:
   - Take snapshots before testing
   - Document initial state
   - Prepare rollback plan
   - Monitor system resources

2. Test Execution:
   - Start with non-critical nodes
   - Monitor multiple metrics
   - Document all actions
   - Keep test duration reasonable

3. Validation:
   - Use automated checks
   - Verify data consistency
   - Compare performance metrics
   - Check logs for errors

4. Documentation:
   - Record all test steps
   - Document unexpected behavior
   - Track recovery times
   - Note any required manual intervention
