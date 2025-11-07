# Monitoring & Observability

## Table of Contents
- [CloudWatch Dashboards](#cloudwatch-dashboards)
- [CloudWatch Alarms](#cloudwatch-alarms)
- [Logging](#logging)
- [Metrics](#metrics)
- [Tracing](#tracing)

## CloudWatch Dashboards

### VITAS Medical Processing Pipeline Dashboard

**Dashboard Name:** `VITAS-Medical-Processing-Pipeline`

**Widgets:**
1. **Workflow Executions** (Line graph)
   - Transcription Workflow starts
   - Analysis Workflow starts
   - Success/failure counts

2. **Lambda Performance** (Multi-metric)
   - Invocations per function
   - Errors per function
   - Duration (p50, p95, p99)
   - Throttles

3. **DLQ Depth** (Number widgets)
   - vitas-transcription-dlq message count
   - vitas-soap-report-dlq message count
   - vitas-icd-diagnosis-dlq message count

4. **Processing Duration** (Heatmap)
   - Transcription time distribution
   - SOAP generation time
   - ICD classification time

5. **SLA Compliance** (Gauge)
   - Files < 12 hours old (warning)
   - Files < 24 hours old (SLA target)
   - SLA violations count

6. **Cost Tracking** (Number)
   - Estimated daily cost
   - Cost per consultation

---

## CloudWatch Alarms

### Critical Alarms

#### 1. Transcription High Error Rate
```yaml
Metric: AWS/Lambda/Errors
Statistic: Sum
Period: 5 minutes
Threshold: > 5 errors
Evaluation: 2 consecutive periods
Actions: SNS notification to operations team
```

#### 2. Transcription Long Duration
```yaml
Metric: AWS/Lambda/Duration
Statistic: Average
Period: 5 minutes
Threshold: > 600000 ms (10 minutes)
Evaluation: 1 period
Actions: SNS notification
```

#### 3. DLQ Message Detected
```yaml
Metric: AWS/SQS/ApproximateNumberOfMessagesVisible
Statistic: Sum
Period: 1 minute
Threshold: > 0
Evaluation: 1 period
Actions: SNS notification (high priority)
```

#### 4. Analysis Workflow Failures
```yaml
Metric: AWS/States/ExecutionsFailed
Statistic: Sum
Period: 5 minutes
Threshold: > 3 failures
Evaluation: 1 period
Actions: SNS notification
```

#### 5. SLA Compliance Risk
```yaml
Metric: Custom/VITAS/ProcessingAgeTooHigh
Statistic: Sum
Period: 1 hour
Threshold: > 0 (files > 12 hours old)
Evaluation: 1 period
Actions: SNS notification (warning)
```

---

## Logging

### Log Groups

| Service | Log Group | Retention |
|---------|-----------|-----------|
| Transcription Lambda | `/aws/lambda/VitasProcessingStack-TranscriptionFunction` | 7 days |
| SOAP Report Lambda | `/aws/lambda/VitasProcessingStack-SOAPReportFunction` | 7 days |
| ICD Diagnosis Lambda | `/aws/lambda/VitasProcessingStack-ICDDiagnosisFunction` | 7 days |
| Workflow Trigger | `/aws/lambda/VitasProcessingStack-WorkflowTriggerFunction` | 7 days |
| DLQ Processor | `/aws/lambda/VitasProcessingStack-DLQProcessorFunction` | 14 days |
| Failure Notification | `/aws/lambda/VitasProcessingStack-FailureNotificationFunction` | 7 days |
| Success Notification | `/aws/lambda/VitasProcessingStack-SuccessNotificationFunction` | 7 days |
| Chat Integration | `/aws/lambda/VitasChatbotStack-ChatIntegration` | 7 days |
| Agent Webhook | `/aws/lambda/VitasChatbotStack-AgentWebhook` | 7 days |
| Step Functions | `/aws/vendedlogs/states/VitasTranscriptionWorkflow` | 7 days |
| Step Functions | `/aws/vendedlogs/states/VitasAnalysisWorkflow` | 7 days |

### Log Insights Queries

#### Query 1: Processing Duration Analysis
```
fields @timestamp, @message
| filter @message like /Processing completed/
| parse @message "duration: * ms" as duration
| stats avg(duration), max(duration), min(duration) by bin(5m)
```

#### Query 2: Error Rate by Service
```
fields @timestamp, service, error
| filter level = "ERROR"
| stats count() by service
| sort count desc
```

#### Query 3: DLQ Retry Success Rate
```
fields @timestamp, @message
| filter @message like /DLQ message retry/
| parse @message "status: *" as status
| stats count() by status
```

---

## Metrics

### Custom CloudWatch Metrics

#### Namespace: `VITAS/Processing`

| Metric Name | Unit | Description |
|-------------|------|-------------|
| `ProcessingDuration` | Milliseconds | End-to-end processing time |
| `TranscriptionDuration` | Milliseconds | Transcription Lambda duration |
| `SOAPGenerationDuration` | Milliseconds | SOAP Report Lambda duration |
| `ICDClassificationDuration` | Milliseconds | ICD Diagnosis Lambda duration |
| `ProviderUsage` | Count | AI provider used (OpenAI/Anthropic/AWS) |
| `DLQRetrySuccess` | Count | Successful DLQ retries |
| `DLQRetryFailure` | Count | Failed DLQ retries |
| `SLAViolation` | Count | Files exceeding 24-hour SLA |
| `NotificationDelivery` | Count | Webhook notification success/failure |

#### Dimensions
- `Service`: transcription | soap-report | icd-diagnosis
- `Provider`: openai | anthropic | aws-transcribe
- `Status`: success | failure
- `RetryAttempt`: 1 | 2 | 3 | dlq

### Publishing Custom Metrics

**Example (TypeScript):**
```typescript
import { CloudWatchClient, PutMetricDataCommand } from '@aws-sdk/client-cloudwatch';

const cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });

await cloudwatch.send(new PutMetricDataCommand({
  Namespace: 'VITAS/Processing',
  MetricData: [
    {
      MetricName: 'ProcessingDuration',
      Value: durationMs,
      Unit: 'Milliseconds',
      Dimensions: [
        { Name: 'Service', Value: 'transcription' },
        { Name: 'Provider', Value: 'openai' }
      ],
      Timestamp: new Date()
    }
  ]
}));
```

---

## Tracing

### LangSmith (LangGraph Monitoring)

**Platform:** https://smith.langchain.com

**Metrics Tracked:**
- Agent execution traces
- Tool call success rates
- Response latency
- Token usage per conversation
- Error rates by agent type (doctor/patient)
- Cost per conversation

**Configuration:**
```bash
# Environment variables
LANGSMITH_API_KEY=your-key
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=vitas-chatbot
```

**Example Trace:**
```
Conversation: thread-123
├─ User Input: "Quiero agendar una cita"
├─ Agent: patient_agent
│  ├─ Reasoning: 5 tokens, 120ms
│  ├─ Tool Call: get_available_doctors()
│  │  └─ Duration: 350ms, Success
│  ├─ Reasoning: 8 tokens, 100ms
│  └─ Response: 45 tokens, 200ms
└─ Total: 58 tokens, 770ms, $0.001
```

### AWS X-Ray (Optional)

Enable X-Ray tracing for Lambda functions to visualize:
- Lambda invocation chains
- DynamoDB query latencies
- S3 operation durations
- External API call times (OpenAI, Anthropic)

**Enable in CDK:**
```typescript
const transcriptionFunction = new lambda.Function(this, 'Transcription', {
  // ... other props
  tracing: lambda.Tracing.ACTIVE
});
```

---

## Monitoring Best Practices

### 1. Set Up Alerts Proactively
- Configure SNS topics for critical alarms
- Use different severity levels (critical, warning, info)
- Test alert delivery regularly

### 2. Regular Dashboard Reviews
- Daily: Check DLQ depth, error rates
- Weekly: Review SLA compliance, cost trends
- Monthly: Analyze performance trends, optimize resources

### 3. Log Retention Strategy
- Critical logs: 14 days
- Standard logs: 7 days
- Archive to S3 for long-term storage

### 4. Cost Monitoring
- Set up AWS Budgets for spending alerts
- Track custom metrics for cost per consultation
- Optimize Lambda memory based on CloudWatch Insights

### 5. Performance Optimization
- Monitor Lambda cold starts
- Adjust timeout based on p99 duration
- Use provisioned concurrency for critical functions

---

**Commands:**

**Tail Lambda logs:**
```bash
aws logs tail /aws/lambda/FUNCTION_NAME --follow --region sa-east-1 --profile vitas-sso
```

**Query logs:**
```bash
aws logs start-query \
  --log-group-name /aws/lambda/FUNCTION_NAME \
  --start-time $(date -u -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter level = "ERROR"' \
  --region sa-east-1 \
  --profile vitas-sso
```

**Get alarm status:**
```bash
aws cloudwatch describe-alarms --alarm-names VitasTranscriptionHighErrorRate --region sa-east-1 --profile vitas-sso
```

---

**Last Updated:** 2025-11-06
