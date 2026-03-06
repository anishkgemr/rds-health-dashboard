# RDS Health Dashboard for Cloud Intelligence Dashboards

A comprehensive Amazon RDS health and cost optimization dashboard that combines operational health metrics with AWS Cost and Usage Report (CUR) data.

## Overview

The RDS Health Dashboard provides:
- **RDS Operational Health**: Instance configurations, end-of-support tracking, backup status, maintenance windows, and resource tagging
- **Cost Analysis**: RDS spending patterns, Reserved Instance utilization, and cost optimization opportunities
- **Unified View**: Combines 5 RDS health data sources with CID cost data in a single QuickSight dashboard

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    RDS Health Dashboard                      │
│                     (QuickSight)                             │
└──────────────────┬──────────────────────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
┌───────▼────────┐   ┌────────▼─────────┐
│ RDS Health Data│   │  CID Cost Data   │
│  (5 views)     │   │ (summary_view)   │
│                │   │                  │
│ • Distribution │   │ • Cost trends    │
│ • Tags         │   │ • RI utilization │
│ • End-of-life  │   │ • Savings opps   │
│ • Backups      │   │                  │
│ • Maintenance  │   │                  │
└────────────────┘   └──────────────────┘
```

## Prerequisites

### Required
1. **CID Data Collection Deployed** - This dashboard requires the Cloud Intelligence Dashboards (CID) foundational infrastructure:
   - AWS Data Exports (CUR 2.0) configured
   - CID Data Collection stack deployed
   - Athena database: `optimization_data` (or custom name)
   - S3 bucket for CID data
   
   📖 [Deploy CID Data Collection](https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/deployment-in-global-regions.html)

2. **QuickSight Enterprise Edition** with:
   - SPICE capacity available (~100 GB recommended)
   - Athena access configured
   - At least 1 author user

3. **IAM Permissions**:
   - CloudFormation stack creation
   - Lambda function creation
   - QuickSight dashboard creation
   - Athena query execution

### Optional
- SNS email for data collection alerts
- Multiple AWS regions for RDS data collection

## Deployment Options

### Option 1: One-Click CloudFormation (Recommended)

**Step 1: Upload Templates to S3**

Since this is not yet on GitHub, upload the templates to your S3 bucket:

```bash
# Set your S3 bucket name
BUCKET_NAME="your-cid-bucket"
PREFIX="rds-health-dashboard"

# Upload templates
aws s3 cp cfn-templates/rds-health-one-click.yaml s3://${BUCKET_NAME}/${PREFIX}/
aws s3 cp dashboards/rds-health.yaml s3://${BUCKET_NAME}/${PREFIX}/dashboards/
aws s3 cp data-collection/module-rds-health.yaml s3://${BUCKET_NAME}/${PREFIX}/data-collection/
```

**Step 2: Get Required ARNs**

The RDS Health Dashboard reuses your existing CID Data Collection infrastructure. You need to provide 7 ARNs as parameters.

**Standard CID ARN Formats** (replace `ACCOUNT_ID` and `REGION` with your values):

```bash
# Get your Account ID and Region
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"  # Your CID deployment region

# Standard CID ARNs (verify these exist in your account)
CID_CRAWLER_ARN="arn:aws:states:${REGION}:${ACCOUNT_ID}:stateMachine:CID-DC-CrawlerExecution-StateMachine"
ACCOUNT_COLLECTOR_ARN="arn:aws:lambda:${REGION}:${ACCOUNT_ID}:function:CID-DC-account-collector-Lambda"
INVENTORY_LAMBDA_ARN="arn:aws:lambda:${REGION}:${ACCOUNT_ID}:function:CID-DC-inventory-Lambda"
INVENTORY_LAMBDA_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-inventory-LambdaRole"
GLUE_CRAWLER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-Glue-Crawler"
SCHEDULER_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-SchedulerExecutionRole"
STEPFUNCTION_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-StepFunctionExecutionRole"
```

**Verify ARNs exist:**
```bash
# Verify Step Functions
aws stepfunctions describe-state-machine --state-machine-arn ${CID_CRAWLER_ARN}

# Verify Lambda functions
aws lambda get-function --function-name CID-DC-account-collector-Lambda
aws lambda get-function --function-name CID-DC-inventory-Lambda

# Verify IAM roles
aws iam get-role --role-name CID-DC-inventory-LambdaRole
aws iam get-role --role-name CID-DC-Glue-Crawler
aws iam get-role --role-name CID-DC-SchedulerExecutionRole
aws iam get-role --role-name CID-DC-StepFunctionExecutionRole
```

If any verification fails, check your CID Data Collection stack outputs:
```bash
aws cloudformation describe-stacks \
  --stack-name CidDataCollectionStack \
  --query 'Stacks[0].Outputs'
```

**Step 3: Deploy the Stack**

```bash
# Set your values
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"
BUCKET_NAME="your-cid-bucket"
PREFIX="rds-health-dashboard"

aws cloudformation create-stack \
  --stack-name rds-health-dashboard \
  --template-url https://s3.amazonaws.com/${BUCKET_NAME}/${PREFIX}/rds-health-one-click.yaml \
  --parameters \
    ParameterKey=QuickSightUser,ParameterValue=Admin/your-username \
    ParameterKey=S3DataBucketName,ParameterValue=${BUCKET_NAME} \
    ParameterKey=DataCollectionDatabaseName,ParameterValue=optimization_data \
    ParameterKey=RegionsInScope,ParameterValue=us-east-1,us-west-2 \
    ParameterKey=DBClusterinfo,ParameterValue=no \
    ParameterKey=SNSEmailSubscription,ParameterValue=your-email@example.com \
    ParameterKey=CIDCrawlerExecutionStateMachineArn,ParameterValue=arn:aws:states:${REGION}:${ACCOUNT_ID}:stateMachine:CID-DC-CrawlerExecution-StateMachine \
    ParameterKey=AccountCollectorLambdaArn,ParameterValue=arn:aws:lambda:${REGION}:${ACCOUNT_ID}:function:CID-DC-account-collector-Lambda \
    ParameterKey=InventoryLambdaArn,ParameterValue=arn:aws:lambda:${REGION}:${ACCOUNT_ID}:function:CID-DC-inventory-Lambda \
    ParameterKey=InventoryLambdaRoleArn,ParameterValue=arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-inventory-LambdaRole \
    ParameterKey=GlueCrawlerRoleArn,ParameterValue=arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-Glue-Crawler \
    ParameterKey=SchedulerExecutionRoleArn,ParameterValue=arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-SchedulerExecutionRole \
    ParameterKey=StepFunctionExecutionRoleArn,ParameterValue=arn:aws:iam::${ACCOUNT_ID}:role/CID-DC-StepFunctionExecutionRole \
    ParameterKey=GitHubRepoURL,ParameterValue=https://s3.amazonaws.com/${BUCKET_NAME}/${PREFIX} \
  --capabilities CAPABILITY_IAM \
  --region ${REGION}
```

**Step 4: Monitor Deployment**

```bash
# Watch stack creation
aws cloudformation describe-stacks \
  --stack-name rds-health-dashboard \
  --query 'Stacks[0].StackStatus'

# Get dashboard URL once complete
aws cloudformation describe-stacks \
  --stack-name rds-health-dashboard \
  --query 'Stacks[0].Outputs[?OutputKey==`DashboardURL`].OutputValue' \
  --output text
```

---

### Option 2: Manual Deployment with cid-cmd

**Step 1: Deploy Data Collection**

```bash
aws cloudformation create-stack \
  --stack-name rds-health-data-collection \
  --template-body file://data-collection/module-rds-health.yaml \
  --parameters \
    ParameterKey=S3DataBucketName,ParameterValue=your-cid-bucket \
    ParameterKey=DataCollectionDatabaseName,ParameterValue=optimization_data \
    ParameterKey=RegionsInScope,ParameterValue=us-east-1,us-west-2 \
    ParameterKey=SNSEmailSubscription,ParameterValue=your-email@example.com \
    ParameterKey=CIDCrawlerExecutionStateMachineArn,ParameterValue=arn:aws:states:... \
    ParameterKey=AccountCollectorLambdaArn,ParameterValue=arn:aws:lambda:... \
    ParameterKey=InventoryLambdaArn,ParameterValue=arn:aws:lambda:... \
  --capabilities CAPABILITY_IAM
```

**Step 2: Wait for Data Collection**

The Lambda runs daily. To trigger manually:

```bash
# Get Step Functions ARN from stack outputs
STATE_MACHINE_ARN=$(aws cloudformation describe-stacks \
  --stack-name rds-health-data-collection \
  --query 'Stacks[0].Outputs[?OutputKey==`StepFunctionsArn`].OutputValue' \
  --output text)

# Start execution
aws stepfunctions start-execution \
  --state-machine-arn ${STATE_MACHINE_ARN}
```

**Step 3: Create RDS Cost Summary View**

```sql
CREATE OR REPLACE VIEW optimization_data.rds_cost_summary_view AS
WITH resource_agg AS (
    SELECT 
        linked_account_id, usage_date, region, product_code, usage_type,
        ARRAY_JOIN(ARRAY_AGG(DISTINCT resource_id), ', ') as resource_ids
    FROM cid_data_export.resource_view
    WHERE product_code = 'AmazonRDS'
    GROUP BY linked_account_id, usage_date, region, product_code, usage_type
)
SELECT 
    a.account_id, a.account_name, s.payer_account_id, s.billing_period,
    s.usage_date, r.resource_ids as resource_id, s.product_code, s.service,
    s.product_name, s.product_family, s.region, s.availability_zone,
    s.instance_type_family, s.instance_type, s.processor, s.processor_features,
    s.current_generation, s.platform, s.database_engine, s.usage_type,
    s.operation, s.charge_type, s.charge_category, s.purchase_option,
    s.pricing_unit, s.usage_quantity, s.unblended_cost, s.amortized_cost,
    s.public_cost, s.ri_sp_arn, s.ri_sp_upfront_fees, s.ri_sp_trueup,
    s.year, s.month
FROM cid_data_export.summary_view s
INNER JOIN cid_data_export.account_map a ON s.linked_account_id = a.account_id
LEFT JOIN resource_agg r ON s.linked_account_id = r.linked_account_id
    AND s.usage_date = r.usage_date AND s.region = r.region
    AND s.product_code = r.product_code AND s.usage_type = r.usage_type
WHERE s.product_code = 'AmazonRDS';
```

**Step 4: Deploy Dashboard with cid-cmd**

```bash
# Install cid-cmd
pip3 install --upgrade cid-cmd

# Deploy dashboard
cid-cmd deploy \
  --resources dashboards/rds-health.yaml \
  --dashboard-id rds-health-dashboard \
  --athena-database optimization_data \
  --athena-workgroup CID \
  --quicksight-user Admin/your-username \
  --share-with-account \
  --yes
```

---

## Post-Deployment

### 1. Access the Dashboard

Navigate to QuickSight:
```
https://<region>.quicksight.aws.amazon.com/sn/dashboards/rds-health-dashboard
```

### 2. Verify Data

The dashboard has 3 sheets:
- **Sheet 1-2**: RDS Health Metrics (5 datasets from Lambda collection)
- **Sheet 3**: RDS Cost Analysis (from CID cost data)

### 3. Schedule Refresh

Data collection runs daily. QuickSight datasets refresh automatically via SPICE.

---

## Troubleshooting

### No Data in Dashboard

**Check data collection:**
```bash
# Check if Lambda ran successfully
aws logs tail /aws/lambda/rds-health-data-collection --follow

# Check S3 for collected data
aws s3 ls s3://your-cid-bucket/rds-health-data/
```

**Check Athena views:**
```sql
-- Verify views exist
SHOW VIEWS IN optimization_data;

-- Test RDS health data
SELECT COUNT(*) FROM optimization_data.rds_analysis_dist_view;

-- Test cost data
SELECT COUNT(*) FROM optimization_data.rds_cost_summary_view;
```

### CloudFormation Stack Failed

Check stack events:
```bash
aws cloudformation describe-stack-events \
  --stack-name rds-health-dashboard \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### QuickSight Permissions

Ensure QuickSight can access Athena:
1. Go to QuickSight → Manage QuickSight → Security & permissions
2. Add Athena access
3. Add S3 bucket access for CID data

---

## Cleanup

### Delete Dashboard Only
```bash
cid-cmd delete --dashboard-id rds-health-dashboard --yes
```

### Delete Everything
```bash
# Delete CloudFormation stack (includes data collection)
aws cloudformation delete-stack --stack-name rds-health-dashboard

# Or delete data collection separately
aws cloudformation delete-stack --stack-name rds-health-data-collection
```

---

## Cost Estimate

| Component | Monthly Cost |
|-----------|--------------|
| Lambda (data collection) | $1-5 |
| S3 (RDS metadata storage) | $1-3 |
| Glue Crawler | $1-2 |
| Athena queries | $2-5 |
| QuickSight SPICE | $10-20 |
| **Total** | **~$15-35/month** |

*Note: Excludes existing CID infrastructure costs*

---

## Support

For issues or questions:
1. Check [CID Documentation](https://docs.aws.amazon.com/guidance/latest/cloud-intelligence-dashboards/)
2. Review [Troubleshooting](#troubleshooting) section
3. Open an issue in this repository

---

## License

MIT-0 License - See LICENSE file
