# VITAS Processing Stack - Quick Reference Guide

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    VITAS Medical Processing Pipeline              │
│                    (AWS CDK + Step Functions)                     │
└─────────────────────────────────────────────────────────────────┘

     [Frontend/Vercel App]
              │
              │ Upload Audio
              ▼
        [S3 Bucket]
      vitas-media-processing-{account}-v2
              │
              │ ObjectCreated Event
              ▼
    [Workflow Trigger Lambda]
              │
    ┌─────────┴──────────┐
    │                    │
    ▼                    ▼
[Transcription WF]  [Analysis WF (direct)]
(Standard, 20min)   (Express, 5min)
    │
    ├─→ [Transcription Lambda]
    │   - OpenAI Whisper (primary)
    │   - AWS Transcribe (fallback)
    │   └─→ S3: transcripts/
    │
    ├─→ [Retry: 3 attempts, 30s-120s backoff]
    │
    └─→ [Analysis WF] (Parallel execution)
        │
        ├─→ [SOAP Report Lambda]
        │   - GPT-4 (primary, temp=0.3)
        │   - Claude Haiku (fallback)
        │   └─→ S3: soap-reports/ + DynamoDB
        │
        └─→ [ICD Diagnosis Lambda]
            - GPT-4 (primary, temp=0.1)
            - Claude Haiku (fallback)
            └─→ DynamoDB
                
        └─→ [Notifications]
            - Success: Via webhook to Vercel
            - Failure: Via webhook to Vercel

[ERROR PATH - SLA Guarantee]

    [DLQ (SQS)] ← Failed messages
        │
        │ Every 2 hours (EventBridge)
        ▼
    [DLQ Processor Lambda]
        │
    ┌───┴────┬────────┬────────┐
    ▼        ▼        ▼        ▼
  <12h     12-24h   >24h   Publish
   Retry    Warn+   SLA    CloudWatch
           Retry   Viol    Metrics
```

## Lambda Functions Quick Lookup

| Function | Memory | Timeout | Purpose | Primary | Fallback |
|----------|--------|---------|---------|---------|----------|
| **Transcription** | 1024 MB | 15 min | Audio→Text | OpenAI Whisper | AWS Transcribe |
| **SOAP Report** | 512 MB | 5 min | Transcript→Report | GPT-4 (0.3°) | Claude Haiku |
| **ICD Diagnosis** | 512 MB | 5 min | Transcript→Diagnoses | GPT-4 (0.1°) | Claude Haiku |
| **Workflow Trigger** | 256 MB | 1 min | Route S3→Workflow | - | - |
| **Failure Notification** | 256 MB | 1 min | Error→Webhook | HTTP POST | - |
| **Success Notification** | 256 MB | 1 min | Success→Webhook | HTTP POST | - |
| **DLQ Processor** | 512 MB | 15 min | Retry logic | Scheduled | - |

## Step Functions State Machines

### TranscriptionWorkflow (STANDARD)
- **Timeout**: 20 minutes
- **Type**: Long-running (standard state machine)
- **Retry**: 3 attempts, 30s-120s backoff

```
TranscribeAudio
  ├─ [Retry 3x: 30s→60s→120s]
  ├─ Success → TriggerAnalysis → NotifySuccess → END
  └─ Failure → SendDLQ → NotifyFailure → FAIL
```

### AnalysisWorkflow (EXPRESS)
- **Timeout**: 5 minutes
- **Type**: Fast synchronous (express state machine)
- **Execution**: Parallel SOAP + ICD

```
ParallelAnalysis
  ├─ GenerateSOAP [Retry 3x, then DLQ]
  └─ GenerateICD [Retry 3x, then DLQ]

CheckSource
  ├─ From Transcription? → Skip notification
  └─ Direct call? → NotifySuccess
```

## S3 Folder Structure

```
vitas-media-processing-{account}-v2/
├── audios/
│   └── {appointmentId}.mp3              [Input: Triggers transcription]
├── transcripts/
│   └── transcript_{appointmentId}.json  [Output: Triggers analysis]
├── soap-reports/
│   └── soap-report-{appointmentId}.json [Output: SOAP reports]
├── icd-diagnoses/
│   └── icd-diagnosis-{appointmentId}.json [Output: ICD codes]
└── transcribe-output/
    └── vitas-transcribe-{ts}.json       [Interim: AWS Transcribe]
```

## DynamoDB Tables (Imported)

### Appointments_Table_V2
```
appointment_id (PK) ← doctor-id, patient-id, appointment-id metadata
├── patient_id
├── doctor_id
├── created_at
├── soap_report_key (S3 path)
├── symptoms (comma-separated)
├── transcription_key (S3 path)
└── isFromSystem (true if auto-generated)
```

### Diagnosis_Table_V2
```
appointment_id (PK)
├── patient_id
├── diagnosis (pipe-separated ICD codes)
├── created_at
└── isFromSystem (true if auto-generated)
```

## SSM Parameters

| Parameter | Type | Value | Used By |
|-----------|------|-------|---------|
| `/vitas/openai-api-key` | SecureString | sk-... | Transcription, SOAP, ICD |
| `/vitas/anthropic-api-key` | SecureString | sk-ant-... | SOAP, ICD (fallback) |
| `/vitas/webhook-url` | String | Vercel API | Notifications |
| `/vitas/vercel-bypass-secret` | SecureString | ... | Notifications |
| `/vitas/prompts/soap-report` | String | XML template | SOAP Lambda |
| `/vitas/prompts/icd-diagnosis` | String | Instructions | ICD Lambda |

## SQS Dead Letter Queues

| Queue | Retention | Purpose |
|-------|-----------|---------|
| `vitas-transcription-dlq-{account}` | 14 days | Transcription failures |
| `vitas-soap-report-dlq-{account}` | 14 days | SOAP report failures |
| `vitas-icd-diagnosis-dlq-{account}` | 14 days | ICD diagnosis failures |

## SLA Timeline

```
Hour 0:   Upload audio → Transcription starts
Hour 0+:  Failure → Message to DLQ
Hour 2:   DLQ Processor runs (attempt 1)
Hour 4:   DLQ Processor runs (attempt 2)
...
Hour 22:  DLQ Processor runs (attempt 11)
Hour 24:  DLQ Processor runs (attempt 12)
         If still failing → SLA violation notification sent
         Message deleted from DLQ
```

## CloudWatch Monitoring

### Alarms
1. **Transcription High Error Rate** (>3 errors/5min)
2. **Transcription Long Duration** (>10 min)
3. **Analysis Workflow Failures** (>1 failure/5min)
4. **DLQ Messages Detected** (>0 messages)
5. **SLA Compliance Risk** (>0 files >12h)

### Dashboard
- Total Executions (24h)
- Success Rate (24h)
- Average Duration
- Failed Executions (24h)
- Execution Trends
- Lambda Invocations & Duration
- Error Rates & Memory Usage

### Log Groups
```
/aws/stepfunctions/AnalysisWorkflow-{account}
(Retention: 7 days, Level: ALL with execution data)

/aws/lambda/{function-name}
(Retention: Never expire, Format: JSON structured)
```

## External API Integrations

### OpenAI
- **Whisper**: Audio transcription (Spanish)
- **GPT-4**: SOAP reports & ICD classification
- **Model**: gpt-4.1
- **Error Handling**: Non-retryable → Anthropic fallback

### Anthropic Claude
- **Claude Haiku**: claude-haiku-4-5-20251001
- **Purpose**: Fallback for SOAP & ICD
- **Max Tokens**: 2000 (SOAP), 1500 (ICD)
- **Trigger**: OpenAI failure (non-retryable)

### AWS Transcribe
- **Purpose**: Fallback transcription (when Whisper fails)
- **Language**: es-ES
- **Processing**: Async job + polling
- **Output**: S3 JSON

### Vercel Webhook
- **Endpoint**: `/api/processing-webhook` (configurable)
- **Method**: HTTP POST
- **Headers**: Custom (X-Event-Type, x-vercel-protection-bypass)
- **Payload**: Success/Failure notifications

## Key Configuration Values

```typescript
// Transcription chunking
CHUNK_SIZE = 15 * 1024 * 1024  // 15 MB chunks

// Step Functions retry
INTERVAL = 30 seconds (transcription), 10 seconds (analysis)
MAX_ATTEMPTS = 3
BACKOFF_RATE = 2.0  // Exponential

// DLQ processing
DLQ_RETRY_INTERVAL = Every 2 hours
DLQ_WARNING_THRESHOLD = 12 hours
DLQ_SLA_VIOLATION = 24 hours
DLQ_RETENTION = 14 days
MAX_MESSAGES_PER_POLL = 10

// Temperatures (LLM)
SOAP_TEMPERATURE = 0.3  // Balanced
ICD_TEMPERATURE = 0.1   // Deterministic

// S3 Lifecycle
TRANSITION_TO_IA = 30 days
EXPIRATION = 90 days
```

## Data Flow Payload Examples

### S3 Event (Trigger)
```json
{
  "Records": [{
    "s3": {
      "bucket": { "name": "vitas-media-processing-..." },
      "object": { "key": "audios/apt-123.mp3" }
    },
    "eventName": "ObjectCreated:Put",
    "eventTime": "2024-11-11T10:30:00Z"
  }]
}
```

### Lambda Response (Success)
```json
{
  "statusCode": 200,
  "body": {
    "message": "Processing completed",
    "service": "openai|anthropic",
    "processingTime": 45000,
    "appointmentId": "apt-123",
    "bucket": "vitas-media-...",
    "transcriptionKey": "transcripts/transcript_apt-123.json"
  }
}
```

### Webhook Notification (Success)
```json
{
  "type": "success",
  "service": "transcription-complete",
  "metadata": {
    "appointmentId": "apt-123",
    "processingTime": 45000,
    "provider": "openai"
  },
  "timestamp": "2024-11-11T10:45:00Z",
  "source": "vitas-processing-stack"
}
```

### Webhook Notification (Failure)
```json
{
  "type": "failure",
  "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES",
  "service": "transcription",
  "metadata": {
    "appointmentId": "apt-123",
    "retryScheduled": true,
    "nextRetryIn": "2 horas"
  },
  "timestamp": "2024-11-11T10:45:00Z",
  "source": "vitas-processing-stack"
}
```

## Deployment Commands

```bash
# Build TypeScript
npm run build

# Synthesize CloudFormation
npm run cdk -- synth

# Deploy stack
npm run cdk -- deploy

# Watch for changes
npm run watch

# Run tests
npm test

# Deploy specific function
npm run cdk -- deploy --hotswap
```

## Post-Deployment Setup

```bash
# Update SSM parameters with actual values
aws ssm put-parameter --name /vitas/openai-api-key \
  --value "sk-..." --type SecureString --overwrite

aws ssm put-parameter --name /vitas/anthropic-api-key \
  --value "sk-ant-..." --type SecureString --overwrite

aws ssm put-parameter --name /vitas/vercel-bypass-secret \
  --value "..." --type SecureString --overwrite

aws ssm put-parameter --name /vitas/webhook-url \
  --value "https://your-app.vercel.app/api/processing-webhook" \
  --type String --overwrite
```

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Lambda timeout | Large audio file | Increase timeout, check chunking |
| Webhook fails | Invalid URL | Verify SSM parameter |
| DLQ growing | Persistent API failures | Check OpenAI/Anthropic status |
| High costs | Excessive API calls | Monitor dashboard, adjust retry |
| Slow processing | Cold starts | Add reserved concurrency |
| Missing metadata | S3 tags not set | Ensure object tags in upload |

## Performance Benchmarks

- **Transcription**: 15-60 min (audio duration dependent)
- **SOAP Generation**: 2-5 minutes
- **ICD Diagnosis**: 2-4 minutes
- **Total E2E**: 20-70 minutes (most time in transcription)
- **Lambda cold start**: <2 seconds (ARM_64 Graviton)
- **DLQ processing**: <5 minutes per 10 messages

## Cost Estimates (Monthly)

- **Lambda invocations**: $0.20 per 1M calls (low volume)
- **Step Functions**: $0.25 per 1M state transitions
- **S3 storage**: $0.023/GB (30 days retention)
- **OpenAI API**: Variable, ~$0.01-0.05 per transcription
- **Anthropic API**: Variable, ~$0.001-0.003 per message
- **CloudWatch**: ~$0.50 for logs + $1-5 for custom metrics

**Estimated Total**: $50-150/month for low volume clinic operations

## File Locations Summary

```
/lib/vitas-processing-stack-stack.ts      - Main CDK stack
/bin/vitas-processing-stack.ts            - Stack entry point
/lambdas/transcription/index.ts           - Audio transcription
/lambdas/soap-report/index.ts             - SOAP generation
/lambdas/icd-diagnosis/index.ts           - ICD classification
/lambdas/workflow-trigger/index.ts        - Workflow routing
/lambdas/failure-notification/index.ts    - Error webhooks
/lambdas/success-notification/index.ts    - Success webhooks
/lambdas/dlq-processor/index.ts           - Retry processor
/lambdas/dlq-processor/config.ts          - DLQ configuration
/prompts/soap-report-prompt.txt           - SOAP prompt template
/prompts/icd-diagnosis-prompt.txt         - ICD prompt template
/layers/processing-layer/                 - Shared dependencies
```

---

**Last Updated**: 2024-11-11
**AWS CDK Version**: 2.215.0
**Node Runtime**: 20.x
**Architecture**: Serverless (Lambda, Step Functions, SQS, S3)

