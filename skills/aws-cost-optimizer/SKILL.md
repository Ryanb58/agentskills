---
name: aws-cost-optimizer
description: AWS cost optimization using CLI - identify waste, unattached resources, and savings opportunities
version: 1.0.0
author: agentskills
tags: [aws, cost, optimization, cloud, cli]
---

# AWS Cost Optimization Skill

Identify waste and optimize AWS costs using AWS CLI commands. Focus on safety-first approach with reporting before any modifications.

## Prerequisites

```bash
# Install AWS CLI v2
brew install awscli    # macOS
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"  # Linux

# Configure credentials
aws configure
aws sts get-caller-identity  # Verify access
```

## Cost Analysis

### Current Month Spend by Service

```bash
# Get current month's costs by service
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[*].[Keys[0],Metrics.BlendedCost.Amount]' \
  --output table

# Top 10 cost drivers
cost_explorer() {
  aws ce get-cost-and-usage \
    --time-period Start=$1,End=$2 \
    --granularity MONTHLY \
    --metrics "BlendedCost" \
    --group-by Type=DIMENSION,Key=SERVICE \
    --query "ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}" \
    --output json | jq -r '.[] | select(.Cost != "0") | "\(.Cost) \(.Service)"' | sort -rn | head -20
}
cost_explorer $(date -v-30d +%Y-%m-%d) $(date +%Y-%m-%d)
```

### Daily Cost Trends

```bash
# Last 14 days daily costs
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-14d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity DAILY \
  --metrics "BlendedCost" \
  --query 'ResultsByTime[*].[TimePeriod.Start,Metrics.BlendedCost.Amount]' \
  --output table
```

## Idle Resources

### Unattached Elastic IPs

```bash
# Find unattached Elastic IPs (cost money even when not in use)
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId,Domain]' \
  --output table

# Calculate waste ($0.005/hour = ~$3.60/month per unattached EIP)
unattached_count=$(aws ec2 describe-addresses --query 'length(Addresses[?AssociationId==null])' --output text)
echo "Unattached EIPs: $unattached_count"
echo "Monthly waste: \$$(echo "$unattached_count * 3.60" | bc)"
```

### Unattached EBS Volumes

```bash
# Find unattached volumes ( gp2/gp3 cost ~$0.10/GB/month)
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,VolumeType,CreateTime,State,AvailabilityZone]' \
  --output table

# Calculate total waste
total_gb=$(aws ec2 describe-volumes --filters Name=status,Values=available --query 'sum(Volumes[].Size)' --output text)
echo "Total unattached storage: ${total_gb} GB"
echo "Monthly waste: ~\$$(echo "${total_gb} * 0.10" | bc)"

# Old unattached volumes (>30 days)
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query "Volumes[?CreateTime<\`$(date -v-30d -u +%Y-%m-%dT%H:%M:%SZ)\`].[VolumeId,Size,CreateTime]" \
  --output table
```

### Unused NAT Gateways

```bash
# List all NAT Gateways with their data transfer (big cost driver)
aws ec2 describe-nat-gateways \
  --query 'NatGateways[*].[NatGatewayId,VpcId,State,ConnectivityType,Tags[?Key==`Name`].Value | [0]]' \
  --output table

# Check CloudWatch metrics for NAT Gateway usage (low DataProcessed = candidate for removal)
# This requires the NAT Gateway ID
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=nat-xxxxxxxxx \
  --start-time $(date -v-7d -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Sum \
  --query 'Datapoints[*].[Timestamp,Sum]' --output table
```

### Old EBS Snapshots

```bash
# List all snapshots older than 90 days
aws ec2 describe-snapshots \
  --owner-ids self \
  --query "Snapshots[?StartTime<\`$(date -v-90d -u +%Y-%m-%dT%H:%M:%SZ)\`].[SnapshotId,VolumeSize,StartTime,Description]" \
  --output table | head -20

# Calculate old snapshot costs (typically $0.05/GB/month)
old_snap_gb=$(aws ec2 describe-snapshots --owner-ids self --query "length(Snapshots[?StartTime<\`$(date -v-90d -u +%Y-%m-%dT%H:%M:%SZ)\`])" --output text)
echo "Snapshots older than 90 days: $old_snap_gb"
```

## Underutilized Compute

### Low CPU EC2 Instances

```bash
# Find instances with low average CPU over last 14 days
# Requires jq for processing

get_low_cpu_instances() {
  local threshold=${1:-10}  # Default 10% CPU
  local days=${2:-14}
  
  aws ec2 describe-instances \
    --query 'Reservations[*].Instances[?State.Name==`running`].[InstanceId,InstanceType,Placement.AvailabilityZone,Tags[?Key==`Name`].Value | [0]]' \
    --output text | while read -r id type az name; do
      
      avg_cpu=$(aws cloudwatch get-metric-statistics \
        --namespace AWS/EC2 \
        --metric-name CPUUtilization \
        --dimensions Name=InstanceId,Value=$id \
        --start-time $(date -v-${days}d -u +%Y-%m-%dT%H:%M:%SZ) \
        --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
        --period 86400 \
        --statistics Average \
        --query 'Datapoints[*].Average | avg(@)' --output text)
      
      if (( $(echo "$avg_cpu < $threshold" | bc -l) )); then
        echo "$id $type $az $name ${avg_cpu}%"
      fi
    done
}

get_low_cpu_instances 10 14
```

### RDS Underutilization

```bash
# List RDS instances with low connections
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceClass,Engine,DBInstanceStatus,AllocatedStorage]' \
  --output table

# Check CloudWatch metrics for specific RDS instance
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=my-db \
  --start-time $(date -v-7d -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Average \
  --query 'Datapoints[*].[Timestamp,Average]' --output table
```

### ECS/Fargate Task Optimization

```bash
# List all ECS services with CPU/Memory settings
aws ecs list-clusters --query 'clusterArns[]' --output text | while read -r cluster; do
  echo "=== Cluster: $cluster ==="
  aws ecs list-services --cluster $cluster --query 'serviceArns[]' --output text | while read -r service; do
    aws ecs describe-services --cluster $cluster --services $service \
      --query 'services[*].[serviceName,launchType,desiredCount,runningCount,cpu,memory]' \
      --output table
  done
done

# Calculate potential Fargate savings
# Compare actual CPU usage vs provisioned
fargate_analysis() {
  local cluster=$1
  aws ecs list-tasks --cluster $cluster --desired-status RUNNING --query 'taskArns[]' --output text | while read -r task; do
    aws ecs describe-tasks --cluster $cluster --tasks $task \
      --query 'tasks[*].[taskArn,cpu,memory,containers[*].name]' --output table
  done
}
```

## AWS Recommendations

### Trusted Advisor Checks

```bash
# List cost optimization Trusted Advisor checks
aws support describe-trusted-advisor-checks \
  --language en \
  --query 'checks[?category==`cost_optimizing`].[id,name]' \
  --output table

# Get results for specific check (e.g., Underutilized EC2)
aws support describe-trusted-advisor-check-result \
  --check-id eW7HH0l7J9 \
  --query 'result.flaggedResources[*].[status,resourceId,metadata]' \
  --output table
```

### Compute Optimizer

```bash
# List Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[*].[instanceArn,instanceName,finding,recommendationOptions[0].instanceType]' \
  --output table | head -20

# Filter for over-provisioned instances only
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[?finding==`Overprovisioned`].[instanceArn,instanceName,recommendationOptions[0].instanceType,recommendationOptions[0].estimatedMonthlySavings]' \
  --output table
```

## Storage Waste

### S3 Bucket Analysis

```bash
# List buckets with size (requires CloudWatch or S3 Inventory)
aws s3api list-buckets --query 'Buckets[*].Name' --output text | while read -r bucket; do
  size_bytes=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/S3 \
    --metric-name BucketSizeBytes \
    --dimensions Name=BucketName,Value=$bucket Name=StorageType,Value=StandardStorage \
    --start-time $(date -v-1d -u +%Y-%m-%dT%H:%M:%SZ) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
    --period 86400 \
    --statistics Average \
    --query 'Datapoints[0].Average' --output text 2>/dev/null)
  
  if [ "$size_bytes" != "None" ] && [ -n "$size_bytes" ]; then
    size_gb=$(echo "scale=2; $size_bytes / 1024 / 1024 / 1024" | bc)
    echo "$bucket: ${size_gb} GB"
  fi
done

# Find incomplete multipart uploads (wasted storage)
aws s3api list-multipart-uploads --bucket my-bucket --query 'Uploads[*].[Key,UploadId,Initiated]' --output table
```

### Orphaned Load Balancers

```bash
# Find ALBs/NLBs with low request count
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[*].[LoadBalancerArn,LoadBalancerName,Type,State.Code]' \
  --output table

# Check request count for specific ALB
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name RequestCount \
  --dimensions Name=LoadBalancer,Value=app/my-alb/xxxx \
  --start-time $(date -v-7d -u +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 86400 \
  --statistics Sum \
  --query 'Datapoints[*].[Timestamp,Sum]' --output table
```

## Automation Scripts

### Complete Cost Audit

```bash
#!/bin/bash
# full_cost_audit.sh - Comprehensive AWS cost audit

echo "=== AWS Cost Optimization Report ==="
echo "Generated: $(date)"
echo ""

# 1. Unattached EIPs
echo "## Unattached Elastic IPs"
unattached_eips=$(aws ec2 describe-addresses --query 'Addresses[?AssociationId==null].[PublicIp]' --output text)
if [ -n "$unattached_eips" ]; then
  echo "$unattached_eips"
  count=$(echo "$unattached_eips" | wc -l)
  echo "💰 Monthly waste: \$$(echo "$count * 3.60" | bc)"
else
  echo "✓ No unattached EIPs found"
fi
echo ""

# 2. Unattached EBS
echo "## Unattached EBS Volumes"
unattached_vols=$(aws ec2 describe-volumes --filters Name=status,Values=available --query 'Volumes[*].[VolumeId,Size]' --output text)
if [ -n "$unattached_vols" ]; then
  echo "$unattached_vols"
  total_gb=$(aws ec2 describe-volumes --filters Name=status,Values=available --query 'sum(Volumes[].Size)' --output text)
  echo "💰 Monthly waste: ~\$$(echo "$total_gb * 0.10" | bc)"
else
  echo "✓ No unattached volumes"
fi
echo ""

# 3. Old Snapshots
echo "## Snapshots older than 90 days"
old_snaps=$(aws ec2 describe-snapshots --owner-ids self --query "Snapshots[?StartTime<\`$(date -v-90d -u +%Y-%m-%dT%H:%M:%SZ)\`].[SnapshotId,VolumeSize,StartTime]" --output text)
if [ -n "$old_snaps" ]; then
  echo "$old_snaps"
  count=$(echo "$old_snaps" | wc -l)
  echo "Found $count old snapshots - review for deletion"
else
  echo "✓ No old snapshots"
fi
echo ""

# 4. Top Cost Services (last 30 days)
echo "## Top Cost Services (Last 30 Days)"
aws ce get-cost-and-usage \
  --time-period Start=$(date -v-30d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query "ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}" \
  --output json | jq -r '.[] | select(.Cost != "0") | "\(.Cost)\t\(.Service)"' | sort -rn | head -10

echo ""
echo "=== Report Complete ==="
```

## Safety First - Always Review Before Action

```bash
# Before deleting anything, always:
# 1. Verify the resource is truly unused
# 2. Create snapshots/backups if needed
# 3. Get approval for deletion

# Example: Safe EIP release workflow
release_unused_eip() {
  local allocation_id=$1
  
  # Double-check it's still unattached
  attached=$(aws ec2 describe-addresses --allocation-ids $allocation_id --query 'Addresses[0].AssociationId' --output text)
  
  if [ "$attached" == "None" ] || [ -z "$attached" ]; then
    echo "EIP $allocation_id confirmed unattached"
    echo "Releasing..."
    # aws ec2 release-address --allocation-id $allocation_id  # UNCOMMENT TO ACTUALLY DELETE
    echo "Would release: aws ec2 release-address --allocation-id $allocation_id"
  else
    echo "⚠️  EIP $allocation_id is now attached! Skipping."
  fi
}
```

## Best Practices

1. **Start with reporting only** - Never delete resources on first run
2. **Tag everything** - Use tags to mark resources for review: `CostReview=true`
3. **Right-size gradually** - Downsize instances in stages (t3.large → t3.medium)
4. **Monitor after changes** - Watch metrics for 1-2 weeks after optimization
5. **Use Savings Plans** - For predictable workloads, commit to 1-3 year savings
6. **Enable AWS Budgets** - Set alerts for unexpected cost increases

## Resources

- [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/)
- [AWS Trusted Advisor](https://aws.amazon.com/premiumsupport/technology/trusted-advisor/)
- [AWS Compute Optimizer](https://aws.amazon.com/compute-optimizer/)
- [AWS Pricing Calculator](https://calculator.aws/)
