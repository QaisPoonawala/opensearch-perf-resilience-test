# OpenSearch Benchmark Infrastructure Setup

> **Note**: This infrastructure setup utilizes and references the official OpenSearch benchmark scripts from [opensearch-project/opensearch-benchmark](https://github.com/opensearch-project/opensearch-benchmark/tree/main) to simplify testing procedures. The original scripts have been integrated into our automated deployment process for seamless execution of benchmarks and resilience tests.

This guide provides instructions for setting up and running OpenSearch benchmarks using AWS CloudFormation.

## Prerequisites

1. AWS CLI installed and configured with appropriate credentials
2. An EC2 key pair for SSH access
3. AWS account with sufficient permissions to create:
   - VPC and associated networking resources
   - EC2 instances
   - OpenSearch domains
   - IAM roles and policies

## Setup Instructions

1. Deploy the CloudFormation stack:
   ```bash
   aws cloudformation create-stack \
     --stack-name opensearch-benchmark \
     --template-body file://opensearch-benchmark-cfn.yaml \
     --parameters \
       ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR_NAME \
     --capabilities CAPABILITY_IAM
   ```

2. Wait for the stack creation to complete (approximately 15-20 minutes):
   ```bash
   aws cloudformation wait stack-create-complete --stack-name opensearch-benchmark
   ```

3. Get the EC2 instance public IP and OpenSearch domain endpoint:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name opensearch-benchmark \
     --query 'Stacks[0].Outputs'
   ```

## Available Components and Scripts

The benchmark infrastructure includes the following components:

1. Core Components:
   - benchmark.py: Main benchmarking engine
   - client.py: OpenSearch client implementation
   - telemetry.py: Performance metrics collection
   - metrics.py: Metrics processing and analysis
   - aggregator.py: Results aggregation

2. Builder Module (/builder):
   - cluster_builder.py: OpenSearch cluster management
   - provisioner.py: Resource provisioning
   - launcher.py: Cluster deployment
   - java_resolver.py: Java runtime management

3. Worker Coordinator (/worker_coordinator):
   - worker_coordinator.py: Distributed test coordination
   - scheduler.py: Test scheduling and management
   - runner.py: Test execution

4. Workload Management:
   - workload/loader.py: Test workload loading
   - workload/params.py: Parameter management
   - workload_generator/: Custom workload generation

5. Utilities (/utils):
   - io.py: File operations
   - console.py: CLI interface
   - metrics processing
   - system statistics
   - network operations

## Client Guide: Running Benchmark Tests

### Initial Setup

1. Clone Repository and Package Scripts:
   ```bash
   # Clone repository
   git clone <repository-url>
   cd <repository-name>

   # Package scripts
   zip -r benchmark-scripts.zip ./*
   ```

2. Create S3 Buckets:
   ```bash
   # Set up variables
   PRIMARY_REGION="us-east-1"
   DR_REGION="us-west-2"
   BUCKET_NAME="opensearch-benchmark-scripts-$(date +%s)"

   # Create primary bucket
   aws s3 mb s3://$BUCKET_NAME --region $PRIMARY_REGION
   aws s3 cp benchmark-scripts.zip s3://$BUCKET_NAME/
   aws s3api put-bucket-versioning \
     --bucket $BUCKET_NAME \
     --versioning-configuration Status=Enabled

   # Create DR bucket
   aws s3 mb s3://$BUCKET_NAME-dr --region $DR_REGION
   ```

### Step 1: Deploy Infrastructure

1. Clone this repository:
   ```bash
   git clone <repository-url>
   cd <repository-name>
   ```

2. Deploy the CloudFormation stack:
   ```bash
   aws cloudformation create-stack \
     --stack-name opensearch-benchmark \
     --template-body file://opensearch-benchmark-cfn.yaml \
     --parameters ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR_NAME \
     --capabilities CAPABILITY_IAM
   ```

3. Wait for deployment completion (~15-20 minutes):
   ```bash
   aws cloudformation wait stack-create-complete --stack-name opensearch-benchmark
   ```

4. Get connection details:
   ```bash
   aws cloudformation describe-stacks \
     --stack-name opensearch-benchmark \
     --query 'Stacks[0].Outputs'
   ```
   Save the BenchmarkInstancePublicIP and OpenSearchDomainEndpoint values.

### Step 2: Connect to Benchmark Instances

1. Connect to Primary Instance:
   ```bash
   # Get primary instance IP
   PRIMARY_IP=$(aws cloudformation describe-stacks \
     --stack-name opensearch-benchmark \
     --query 'Stacks[0].Outputs[?OutputKey==`BenchmarkInstancePublicIP`].OutputValue' \
     --output text)

   # Connect via SSH
   ssh -i YOUR_KEY_PAIR.pem ec2-user@$PRIMARY_IP
   ```

2. Connect to DR Instance:
   ```bash
   # Get DR instance IP
   DR_IP=$(aws cloudformation describe-stacks \
     --stack-name opensearch-benchmark-dr \
     --query 'Stacks[0].Outputs[?OutputKey==`BenchmarkInstancePublicIP`].OutputValue' \
     --output text \
     --region $DR_REGION)

   # Connect via SSH
   ssh -i YOUR_KEY_PAIR.pem ec2-user@$DR_IP
   ```

1. Connect via SSH:
   ```bash
   ssh -i YOUR_KEY_PAIR.pem ec2-user@INSTANCE_PUBLIC_IP
   ```

### Step 3: Running Tests

1. Primary Region Tests:
   ```bash
   # Start monitoring
   ./monitoring-scripts/monitor-cluster.sh &
   ./monitoring-scripts/monitor-performance.sh &

   # Run benchmark tests
   ./run-benchmark.sh

   # Custom workload test
   opensearch-benchmark execute-test \
     --target-hosts=OPENSEARCH_ENDPOINT \
     --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
     --test-procedure=default \
     --workload=WORKLOAD_NAME \
     --workload-params "bulk_size:1000,number_of_replicas:1"
   ```

2. DR Region Tests:
   ```bash
   # Start monitoring
   ./monitoring-scripts/monitor-cluster.sh &
   ./monitoring-scripts/monitor-performance.sh &

   # Run DR tests
   ./run-dr-test.sh
   ```

3. Available Test Types:

1. Generate Sample Data:
   Each workload comes with built-in data generation capabilities. The benchmark tool automatically:
   - Downloads or generates sample data for the selected workload
   - Creates necessary indices in OpenSearch
   - Indexes the data before running performance tests

   Data characteristics by workload:
   - geonames: ~11M geographic records
   - nyc_taxis: ~1M taxi trip records
   - http_logs: ~2M HTTP server log entries
   - nested: ~1M records with nested fields
   - percolator: ~1M queries and documents
   - pmc: ~1M PubMed Central articles

2. Quick Test (Default Workload):
   ```bash
   ./run-benchmark.sh
   ```
   This runs the geonames workload with default settings.

2. Custom Workload Test:
   ```bash
   opensearch-benchmark execute-test \
     --target-hosts=OPENSEARCH_ENDPOINT \
     --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
     --test-procedure=default \
     --workload=WORKLOAD_NAME
   ```

   Available workloads and their test scenarios:
   - geonames:
     * Bulk indexing of geographic data
     * Name prefix queries
     * Phrase queries
     * Geographic bounding box queries
   - nyc_taxis:
     * Bulk indexing of trip data
     * Range queries on trip distance
     * Aggregations on payment types
     * Complex queries combining multiple fields
   - http_logs:
     * Bulk indexing of log entries
     * Full-text search on user agents
     * Date range queries
     * IP address queries
   - nested:
     * Indexing documents with nested objects
     * Nested field queries
     * Parent-child relationship queries
   - percolator:
     * Storing queries as documents
     * Document matching against stored queries
     * Query registration performance
   - pmc:
     * Full-text article indexing
     * Scientific term queries
     * Author and citation searches

3. Data Management:
   ```bash
   # View generated data location
   ls -l ~/.benchmark/benchmarks/data/

   # Check index statistics
   curl -X GET "https://OPENSEARCH_ENDPOINT/_cat/indices?v" \
     -u admin:Admin123! --insecure

   # Delete indices after testing (optional)
   curl -X DELETE "https://OPENSEARCH_ENDPOINT/test-*" \
     -u admin:Admin123! --insecure
   ```

4. Custom Test Parameters:
   ```bash
   opensearch-benchmark execute-test \
     --target-hosts=OPENSEARCH_ENDPOINT \
     --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
     --test-procedure=default \
     --workload=WORKLOAD_NAME \
     --workload-params "bulk_size:1000,number_of_replicas:1" \
     --test-iterations=3
   ```

   Common parameters:
   - bulk_size: Number of documents per bulk request
   - number_of_replicas: Number of replica shards
   - test-iterations: Number of times to repeat the test
   - client-options: Connection settings

### Step 4: Analyzing Results

1. View Real-time Metrics:
   ```bash
   # Check cluster health
   curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cluster/health | jq

   # Check indices
   curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/indices?v

   # Check shards
   curl -s -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/_cat/shards?v
   ```

2. View Test Results:

1. Real-time Results:
   - Test progress and metrics are displayed in the terminal
   - Key metrics include:
     * Throughput (ops/sec)
     * Latency (ms)
     * Error rate
     * Resource utilization

2. Detailed Reports:
   ```bash
   cd ~/.benchmark/benchmarks/results
   ls -l
   ```
   Results are stored in JSON format with timestamps.

3. Analyze Results:
   ```bash
   cat ~/.benchmark/benchmarks/results/TIMESTAMP/results.json
   ```
   This shows detailed metrics including:
   - Operation statistics
   - Latency percentiles
   - Error counts
   - System metrics

### Step 5: Clean Up Resources

1. Stop Running Tests:
   ```bash
   # Find and stop monitoring processes
   ps aux | grep monitor
   kill PROCESS_ID
   ```

2. Delete Test Data:
   ```bash
   # Delete test indices
   curl -X DELETE -u admin:Admin123! --insecure https://OPENSEARCH_ENDPOINT/test-*
   ```

3. Delete Infrastructure:
   ```bash
   # Delete primary stack
   aws cloudformation delete-stack --stack-name opensearch-benchmark
   aws cloudformation wait stack-delete-complete --stack-name opensearch-benchmark

   # Delete DR stack
   aws cloudformation delete-stack --stack-name opensearch-benchmark-dr --region $DR_REGION
   aws cloudformation wait stack-delete-complete --stack-name opensearch-benchmark-dr --region $DR_REGION

   # Delete S3 buckets
   aws s3 rb s3://$BUCKET_NAME --force
   aws s3 rb s3://$BUCKET_NAME-dr --force --region $DR_REGION
   ```

When finished testing:

1. Delete CloudFormation stack:
   ```bash
   aws cloudformation delete-stack --stack-name opensearch-benchmark
   aws cloudformation wait stack-delete-complete --stack-name opensearch-benchmark
   ```

2. Verify deletion in AWS Console:
   - Check CloudFormation stacks
   - Verify all resources are removed
   - Check for any remaining EBS volumes

## Multi-AZ and Disaster Recovery Testing

### Primary Region Configurations

1. Standard Multi-AZ (High Availability):
   ```bash
   aws cloudformation create-stack \
     --stack-name opensearch-benchmark-standard \
     --template-body file://opensearch-benchmark-cfn.yaml \
     --parameters \
       ParameterKey=ClusterConfig,ParameterValue=standard \
       ParameterKey=DataNodeCount,ParameterValue=4 \
       ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR_NAME \
     --capabilities CAPABILITY_IAM
   ```
   Features:
   - 4+ data nodes across AZs
   - 3 dedicated master nodes
   - Best for production workloads
   - Full high availability

2. Minimal Multi-AZ (Cost-Optimized):
   ```bash
   aws cloudformation create-stack \
     --stack-name opensearch-benchmark-minimal \
     --template-body file://opensearch-benchmark-cfn.yaml \
     --parameters \
       ParameterKey=ClusterConfig,ParameterValue=minimal \
       ParameterKey=DataNodeCount,ParameterValue=2 \
       ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR_NAME \
     --capabilities CAPABILITY_IAM
   ```
   Features:
   - 2+ data nodes across AZs
   - No dedicated master nodes
   - Cost-effective configuration
   - Basic high availability

3. Coordinator Node Multi-AZ:
   ```bash
   aws cloudformation create-stack \
     --stack-name opensearch-benchmark-coordinator \
     --template-body file://opensearch-benchmark-cfn.yaml \
     --parameters \
       ParameterKey=ClusterConfig,ParameterValue=coordinator \
       ParameterKey=DataNodeCount,ParameterValue=4 \
       ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR_NAME \
     --capabilities CAPABILITY_IAM
   ```
   Features:
   - 4+ data nodes across AZs
   - 3 dedicated master nodes
   - 2 coordinator nodes
   - Best for query-heavy workloads

### Testing Scenarios

For detailed resilience testing procedures, monitoring steps, and validation checklists, see [Resilience Testing Guide](resilience-testing-guide.md). The guide includes:
- Step-by-step test procedures for node failures, AZ failures, and DR scenarios
- Monitoring scripts and metrics to track
- Validation checklists for each test type
- Test result templates and reporting guidelines
- Common issues and solutions
- Best practices for resilience testing

1. High Availability Tests:
   ```bash
   # Test cluster stability during node failures
   opensearch-benchmark execute-test \
     --target-hosts=OPENSEARCH_ENDPOINT \
     --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
     --test-procedure=default \
     --workload=geonames \
     --workload-params "bulk_size:5000,number_of_replicas:1" \
     --test-iterations=5
   ```

2. Query Performance Tests:
   ```bash
   # Test query performance across configurations
   opensearch-benchmark execute-test \
     --target-hosts=OPENSEARCH_ENDPOINT \
     --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
     --test-procedure=search-only \
     --workload=pmc \
     --workload-params "bulk_size:0,number_of_replicas:1" \
     --test-iterations=3
   ```

3. Recovery Tests:
   ```bash
   # Test cluster recovery after AZ failure
   opensearch-benchmark execute-test \
     --target-hosts=OPENSEARCH_ENDPOINT \
     --client-options="use_ssl:true,verify_certs:false,basic_auth_user:admin,basic_auth_password:Admin123!" \
     --test-procedure=default \
     --workload=http_logs \
     --workload-params "bulk_size:10000,number_of_replicas:1" \
     --test-iterations=2
   ```

### Comparing Results

1. Performance Metrics:
   - Query latency across configurations
   - Indexing throughput
   - Recovery time after failures
   - Cross-AZ replication lag

2. Resilience Metrics:
   - Time to recover from node failures
   - Impact of AZ failures
   - Replication performance
   - Failover success rate

3. Cost-Performance Analysis:
   - Resource utilization
   - Cost per operation
   - Performance/cost ratio
   - Optimal configuration for workload

### Disaster Recovery Configuration

1. Deploy DR Infrastructure:
   ```bash
   # Get primary domain endpoint
   PRIMARY_ENDPOINT=$(aws cloudformation describe-stacks \
     --stack-name opensearch-benchmark \
     --query 'Stacks[0].Outputs[?OutputKey==`OpenSearchDomainEndpoint`].OutputValue' \
     --output text)

   # Deploy DR stack in different region
   aws cloudformation create-stack \
     --stack-name opensearch-benchmark-dr \
     --template-body file://opensearch-benchmark-dr.yaml \
     --parameters \
       ParameterKey=KeyPairName,ParameterValue=YOUR_KEY_PAIR_NAME \
       ParameterKey=PrimaryDomainEndpoint,ParameterValue=$PRIMARY_ENDPOINT \
       ParameterKey=PrimaryRegion,ParameterValue=us-east-1 \
     --capabilities CAPABILITY_IAM \
     --region us-west-2
   ```

2. DR Testing Scenarios:
   ```bash
   # Connect to DR instance
   ssh -i YOUR_KEY_PAIR.pem ec2-user@DR_INSTANCE_IP

   # Run DR tests
   ./run-dr-test.sh
   ```

   The DR test script performs:
   - Replication testing from primary to DR cluster
   - Search performance testing on DR cluster
   - Failover scenario testing
   - Cross-region latency measurements

3. DR Test Analysis:
   - Replication Metrics:
     * Data consistency between regions
     * Replication lag time
     * Cross-region bandwidth usage
   
   - Failover Metrics:
     * Recovery Point Objective (RPO)
     * Recovery Time Objective (RTO)
     * Data loss assessment
     * Service disruption duration

   - Performance Metrics:
     * Cross-region query latency
     * Search performance comparison
     * Index operation throughput
     * Resource utilization

4. DR Best Practices:
   - Regular replication testing
   - Periodic failover drills
   - Cross-region performance monitoring
   - Recovery procedure documentation
   - Automated failover testing

## Infrastructure Details

The CloudFormation template creates:

1. Network Infrastructure:
   - VPC with public and private subnets
   - Internet Gateway for public access
   - Route tables and security groups

2. Amazon OpenSearch Service Domain:
   - Fully managed OpenSearch service
   - Multi-AZ deployment for high availability:
     * 2 data nodes across different AZs
     * 3 dedicated master nodes for cluster management
     * Zone awareness enabled for automatic failover
   - 100GB GP3 EBS volume per node
   - Enterprise-grade features:
     * Encryption at rest and node-to-node encryption
     * HTTPS enforcement with TLS 1.2
     * Built-in monitoring and alerting
     * Automated backups and snapshots
     * Managed updates and maintenance
   - VPC deployment across multiple AZs
   - Basic authentication enabled
   - Service-managed scaling and high availability

3. Benchmark EC2 Instance:
   - Pre-installed with OpenSearch Benchmark tool
   - Configured with necessary permissions
   - Ready-to-use benchmark scripts

## Monitoring and Results

- Benchmark results will be displayed in the terminal after completion
- Results are stored in ~/.benchmark/benchmarks/results
- Monitor OpenSearch metrics through the AWS Console:
  - CPU Utilization
  - JVM Memory Pressure
  - Cluster Health
  - Request Latency

## Amazon OpenSearch Service Features

1. High Availability Architecture:
   - Multi-AZ deployment:
     * Data nodes distributed across AZs
     * Dedicated master nodes for cluster management
     * Automatic failover between AZs
   - Resilience features:
     * Cross-zone replication
     * Automatic recovery
     * Load balancing across zones
   - Disaster recovery capabilities:
     * Automated snapshots
     * Point-in-time recovery
     * Cross-zone redundancy

2. Managed Service Benefits:
   - Automated software patching
   - Hardware provisioning and management
   - Built-in monitoring and alerting
   - Automated backups and snapshots
   - Easy scaling options
   - High availability features
   - Managed security updates

2. Service Integration:
   - Native AWS service integration
   - CloudWatch metrics and dashboards
   - AWS IAM authentication
   - VPC integration
   - KMS encryption support

3. Performance and Resilience Features:
   - Multi-AZ deployment for high availability
   - Automatic failover and recovery
   - Cross-zone replication
   - UltraWarm storage for cost-effective storage
   - Auto-Tune for performance optimization
   - Query performance optimization
   
4. Resilience Testing Capabilities:
   - Test scenarios across multiple AZs:
     * Node failure simulation
     * AZ failure recovery testing
     * Cross-zone latency measurement
     * Load distribution verification
   - Failover testing:
     * Master node election
     * Data node recovery
     * Shard reallocation testing
   - Performance testing:
     * Cross-zone query performance
     * Replication lag measurement
     * Recovery time objectives (RTO)
     * Recovery point objectives (RPO)

## Security Features

1. Network Security:
   - OpenSearch domain in private subnet
   - EC2 instance in public subnet with SSH access
   - Security groups restricting access

2. Data Security:
   - Encryption at rest enabled
   - Node-to-node encryption enabled
   - Basic authentication required
   - VPC deployment

3. Access Control:
   - IAM role for EC2 instance
   - Limited permissions to OpenSearch domain
   - SSH key pair authentication

## Cleanup

To delete all created resources:
```bash
aws cloudformation delete-stack --stack-name opensearch-benchmark
aws cloudformation wait stack-delete-complete --stack-name opensearch-benchmark
```

## Troubleshooting

1. Connection Issues:
   - Verify OpenSearch domain status is "Active"
   - Check security group rules
   - Ensure EC2 instance has proper IAM role
   - Verify VPC endpoints and routing

2. Benchmark Failures:
   - Check ~/.benchmark/logs/benchmark.log
   - Verify Python dependencies
   - Ensure sufficient disk space
   - Check OpenSearch cluster health

3. Performance Issues:
   - Monitor JVM memory pressure
   - Check EBS volume throughput
   - Verify instance sizing is appropriate
   - Review concurrent request settings

## Cost Tracking and Optimization

1. Resource Tagging:
   - All resources are tagged with:
     * Name: Unique identifier for each resource
     * Project: OpenSearch-Benchmark
   - Use these tags in AWS Cost Explorer to track expenses
   - Filter costs by component (EC2, OpenSearch, Network, etc.)
   - Monitor resource-specific costs

2. Instance Sizing:
   - Start with r6g.large.search for OpenSearch
   - Use t3.large for benchmark instance
   - Scale up if needed based on results

2. Storage:
   - GP3 volumes provide good baseline performance
   - Adjust volume size based on workload
   - Consider cleanup of benchmark data

3. Testing Duration:
   - Run shorter tests during development
   - Use full benchmarks for production validation
   - Clean up resources after testing

## Best Practices

1. Testing:
   - Start with small datasets
   - Gradually increase workload size
   - Monitor cluster health during tests
   - Allow warm-up period before tests

2. Security:
   - Change default passwords
   - Restrict SSH access to known IPs
   - Regularly rotate credentials
   - Monitor security logs

3. Operations:
   - Take snapshots before large tests
   - Monitor resource utilization
   - Keep benchmark tool updated
   - Document custom configurations

## Support

For issues or questions:
1. Check AWS CloudFormation console for stack events
2. Review OpenSearch domain logs
3. Check EC2 instance system logs
4. Consult OpenSearch Benchmark documentation
