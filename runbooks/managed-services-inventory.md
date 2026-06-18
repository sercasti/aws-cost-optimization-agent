# Runbook: Managed services inventory

**Goal**: detect always-billed managed services that are idle, over-configured for non-production, or missing free-tier networking optimizations. These services don't appear in standard EC2/EBS/RDS scans but can represent $200-1,000+/mo of avoidable cost.

**Typical capture**: $200-800/mo per account in mature organizations. High variance depending on services adopted.

**Risk profile**: low on the scan side (all read-only). Medium on action side -- each service has a different deletion/downgrade path and most require owner confirmation.

## Services covered

1. Transfer Family SFTP servers
2. Amazon MQ brokers
3. ECS Container Insights (non-prod over-enabling)
4. Lambda Insights inverted pyramid
5. NAT Gateway gateway endpoint gap (S3, DynamoDB)

Each has its own phase below.

---

## Phase 1: Transfer Family SFTP servers

AWS Transfer Family charges $0.30/hr per server regardless of usage. A server with zero transfers in 30 days costs $216/mo with nothing to show for it.

```bash
aws transfer list-servers --profile $PROFILE --region $REGION \
  --query 'Servers[].[ServerId,State,IdentityProviderType,Domain]' --output table
```

For each server in ONLINE state, check CloudWatch for file transfer activity:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Transfer \
  --metric-name FilesIn \
  --dimensions Name=ServerId,Value=$SERVER_ID \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 2592000 --statistics Sum \
  --region $REGION --profile $PROFILE \
  --query 'Datapoints[0].Sum'
```

Also check FilesOut with the same parameters. Zero on both = idle server.

**Owner conversation template**:

```
Found AWS Transfer Family SFTP server $SERVER_ID in $REGION.
State: ONLINE. Cost: ~$216/mo regardless of usage.

CloudWatch shows 0 files transferred in the past 30 days.

Two options:
1. Stop the server (state=OFFLINE, reduces billing to $0/hr while preserving config and users).
   `aws transfer stop-server --server-id $SERVER_ID --region $REGION --profile $PROFILE`
2. Delete the server if the SFTP integration is no longer needed.

Which path do you want?
```

Stopping is reversible and reduces cost immediately to ~$0/hr. Deletion is permanent. Let the owner choose.

**Engagement ref**: Traffilog `mbz-edi-repo` Transfer Family server = 0 transfers in 30 days, $216/mo.

---

## Phase 2: Amazon MQ brokers

Amazon MQ Active-Standby pairs charge 2x the instance rate 24/7 regardless of message volume. A 3x mq.m5.large pair costs ~$500/mo at zero utilization.

```bash
aws mq list-brokers --profile $PROFILE --region $REGION \
  --query 'BrokerSummaries[].[BrokerId,BrokerName,BrokerState,DeploymentMode,EngineType,HostInstanceType]' \
  --output table
```

For each broker, check 30-day message volume:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/AmazonMQ \
  --metric-name TotalMessageCount \
  --dimensions Name=Broker,Value=$BROKER_NAME \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 2592000 --statistics Sum \
  --region $REGION --profile $PROFILE \
  --query 'Datapoints[0].Sum'
```

Zero messages in 30 days = idle. Even for active brokers, evaluate whether ActiveMQ is still the right tool or whether SQS/SNS covers the use case at a fraction of the cost.

**Disposition buckets**:
- Zero messages, RUNNING state: idle candidate. Surface to owner.
- Low messages, ACTIVE_STANDBY: evaluate whether single-instance mode would suffice.
- Messages present, ACTIVE_STANDBY: keep. Note instance size for rightsizing consideration.

---

## Phase 3: ECS Container Insights (non-prod over-enabling)

Container Insights costs approximately $56/cluster/month (flat rate based on ~188 CloudWatch custom metrics per cluster). This is the same cost whether the cluster runs 2 tasks or 200. Enabling it on dev and uat clusters burns money on monitoring infrastructure that's rarely reviewed.

Scan for enabled clusters:

```bash
aws ecs list-clusters --profile $PROFILE --region $REGION \
  --query 'clusterArns' --output text \
  | tr '\t\r' '\n\n' | xargs -n 10 aws ecs describe-clusters \
      --include SETTINGS \
      --profile $PROFILE --region $REGION \
      --query 'clusters[?settings[?name==`containerInsights` && value==`enabled`]].[clusterName,status]' \
      --output table
```

Cross-reference the cluster name against environment naming conventions (typically `dev`, `uat`, `staging` are non-prod). Disable on non-prod:

```bash
aws ecs update-cluster-settings \
  --cluster $CLUSTER_NAME \
  --settings name=containerInsights,value=disabled \
  --profile $PROFILE --region $REGION
```

This is safe to do without downtime -- it removes custom metrics from CloudWatch and stops billing. Existing CloudWatch Insights dashboards stop receiving new data points but historical data is retained.

**Gotcha**: cost is cluster-level, not task-level. A dev cluster with 2 tasks and a prod cluster with 50 tasks cost the same $56/mo for Container Insights. The monitoring-to-traffic ratio is inverted in dev. Disable dev/uat; keep prod.

**Engagement ref**: Easytrack-Next dev + uat clusters with Container Insights = $112/mo eliminated.

---

## Phase 4: Lambda Insights inverted pyramid

Lambda Insights (the ENHANCED_MONITORING layer) costs approximately $0.30/metric/mo. Dev environments with many functions and Insights enabled can cost more for monitoring than production.

Check for the Lambda Insights layer in each account:

```bash
aws lambda list-functions --profile $PROFILE --region $REGION \
  --query 'Functions[?Layers!=null].[FunctionName,Layers[].LayerArn]' \
  --output json \
  | jq -r '.[] | . as [$fn, $layers] | 
      ($layers // [])[] | 
      select(contains("LambdaInsightsExtension")) | 
      "\($fn)"'
```

Count functions with Insights enabled per environment (use function name prefix or tag filtering). If dev/non-prod has more Insights-enabled functions than prod, you have an inverted monitoring pyramid.

**Flag pattern**: dev function count with Insights > prod function count with Insights. This is backwards. Monitoring cost should scale with traffic importance, not with the number of functions deployed.

**Resolution**: remove the LambdaInsightsExtension layer from dev/uat functions, or use a Terraform `count` toggle per workspace.

**Engagement ref**: Easytrack-Next dev Lambda Insights = $254/mo (94 functions) vs. prod = $59/mo (22 functions). Dev was 4x the monitoring cost of prod.

---

## Phase 5: NAT Gateway gateway endpoint gap

S3 and DynamoDB calls from within a VPC through NAT Gateway cost $0.045/GB of data transfer. The same calls through S3 or DynamoDB gateway VPC endpoints are free. A VPC with a NAT Gateway and no S3/DynamoDB gateway endpoint is routing avoidable traffic through NAT.

**Step 1**: Find all VPCs with NAT Gateways:

```bash
aws ec2 describe-nat-gateways --profile $PROFILE --region $REGION \
  --filter Name=state,Values=available \
  --query 'NatGateways[].[NatGatewayId,VpcId,Tags]' --output table
```

**Step 2**: Find which VPCs already have S3/DynamoDB gateway endpoints:

```bash
aws ec2 describe-vpc-endpoints --profile $PROFILE --region $REGION \
  --filters Name=vpc-endpoint-type,Values=Gateway Name=state,Values=available \
  --query 'VpcEndpoints[].[VpcId,ServiceName]' --output table
```

Service names to look for: `com.amazonaws.<region>.s3` and `com.amazonaws.<region>.dynamodb`.

**Step 3**: Cross-reference. Any VPC that has a NAT Gateway but lacks either endpoint is a gap candidate.

If the NAT Gateway shows high BytesOutToDestination in CloudWatch, the gap is likely material:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name BytesOutToDestination \
  --dimensions Name=NatGatewayId,Value=$NGW_ID \
  --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 2592000 --statistics Sum \
  --region $REGION --profile $PROFILE \
  --query 'Datapoints[0].Sum'
```

GB = sum / 1073741824. Cost at $0.045/GB.

**Creating the missing endpoints** (non-Terraform accounts):

```bash
# S3 gateway endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.$REGION.s3 \
  --route-table-ids $ROUTE_TABLE_IDS \
  --profile $PROFILE --region $REGION

# DynamoDB gateway endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.$REGION.dynamodb \
  --route-table-ids $ROUTE_TABLE_IDS \
  --profile $PROFILE --region $REGION
```

Gateway endpoints are free. Route tables associated with the endpoint get an automatic route entry for the service prefix list. All S3/DynamoDB traffic from that VPC routes through the endpoint at zero cost instead of NAT.

**For Terraform-managed VPCs**: add `aws_vpc_endpoint` resources per the customer's IaC patterns and open a draft PR (see `runbooks/tf-pr-draft-pattern.md`).

**Gotcha**: adding a gateway endpoint doesn't change existing connection state. It only affects new connections. If a Lambda or ECS task resolves S3 endpoints at DNS query time and caches the result, the change may take a connection refresh to take effect.

**Engagement ref**: Tecnocontrol and Easytrack each showed $133-145/mo in VPC data transfer costs attributable to S3/DynamoDB calls routing through NAT.

---

## Cross-account sweep template

Run all five checks across accounts in a single session:

```bash
for account in "profile-1:us-east-1:AccountA" "profile-2:us-west-2:AccountB"; do
  PROFILE=$(echo $account | cut -d: -f1)
  REGION=$(echo $account  | cut -d: -f2)
  NAME=$(echo $account    | cut -d: -f3)
  echo "=== $NAME ($PROFILE $REGION) ==="

  echo "--- Transfer Family ---"
  aws transfer list-servers --profile $PROFILE --region $REGION \
    --query 'Servers[].[ServerId,State]' --output text

  echo "--- Amazon MQ ---"
  aws mq list-brokers --profile $PROFILE --region $REGION \
    --query 'BrokerSummaries[].[BrokerName,BrokerState,DeploymentMode]' --output text

  echo "--- ECS Container Insights (enabled clusters) ---"
  for arn in $(aws ecs list-clusters --profile $PROFILE --region $REGION --query 'clusterArns[]' --output text | tr '\t\r' '\n\n'); do
    aws ecs describe-clusters --clusters "$arn" --include SETTINGS \
      --profile $PROFILE --region $REGION \
      --query 'clusters[?settings[?name==`containerInsights`&&value==`enabled`]].clusterName' \
      --output text
  done

  echo "--- NAT GW without S3/DDB gateway endpoint ---"
  aws ec2 describe-nat-gateways --profile $PROFILE --region $REGION \
    --filter Name=state,Values=available \
    --query 'NatGateways[].[NatGatewayId,VpcId]' --output text
  echo "(compare VpcIds above against:)"
  aws ec2 describe-vpc-endpoints --profile $PROFILE --region $REGION \
    --filters Name=vpc-endpoint-type,Values=Gateway Name=state,Values=available \
    --query 'VpcEndpoints[].[VpcId,ServiceName]' --output text
done
```

Prioritize findings by dollar impact: Transfer Family and MQ are flat costs ($216/mo and $200-500/mo respectively) and visible immediately. Container Insights and Lambda Insights are proportional to cluster/function count. NAT gateway endpoint gaps require CloudWatch data to size.
