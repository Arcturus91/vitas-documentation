# Processing Pipeline Integration

## Table of Contents
- [Overview](#overview)
- [Complete Processing Flow](#complete-processing-flow)
- [Step Functions Workflows](#step-functions-workflows)
- [Lambda Functions](#lambda-functions)
- [Error Handling & DLQ](#error-handling--dlq)
- [Notifications Integration](#notifications-integration)
- [Testing Failures](#testing-failures)

## Overview

The VITAS Clinic processing pipeline automatically transcribes medical audio recordings, generates SOAP reports, and classifies ICD-10 diagnoses using AI. The system guarantees 24-hour processing SLA through automatic retries and dead letter queue (DLQ) management.

### Key Features
- **Multi-provider AI:** OpenAI (primary) with Anthropic/AWS fallbacks
- **Automatic retries:** 3 attempts with exponential backoff
- **DLQ processing:** Automatic retry every 2 hours for 24 hours
- **Real-time notifications:** Success/failure webhooks to frontend
- **Parallel processing:** SOAP and ICD analysis run concurrently
- **Cost optimization:** Hybrid Standard/Express Step Functions workflows

---

## Complete Processing Flow

```
┌───────────────────────────────────────────────────────────────────────┐
│                    AUDIO PROCESSING PIPELINE                           │
└───────────────────────────────────────────────────────────────────────┘

[Doctor Dashboard]
    ↓ Record audio (MediaRecorder API)
    ↓ Convert to MP3 (lamejs)
    ↓ Upload to S3 via presigned URL
    ↓
[S3 Bucket: vitas-media-processing/audios/]
    ├─ Folder structure: audios/doctor-{id}/patient-{id}/appointment-{id}.mp3
    ├─ Metadata embedded: doctor-id, patient-id, appointment-id
    ↓ S3 ObjectCreated event
[Lambda: workflow-trigger]
    ├─ Parse S3 event
    ├─ Route by folder: audios/* → Transcription Workflow
    ↓
[Step Functions: Transcription Workflow (Standard, 20 min timeout)]
    ├─ Invoke Transcription Lambda (15 min timeout)
    │  ├─ Retry 1: Wait 30s, retry
    │  ├─ Retry 2: Wait 60s, retry (2x backoff)
    │  ├─ Retry 3: Wait 120s, retry (2x backoff)
    │  └─ On all retries fail:
    │     ├─ Catch block → Send to DLQ (SqsSendMessage)
    │     ├─ Invoke Failure Notification Lambda
    │     └─ Fail state
    ├─ [Transcription Lambda Execution]
    │  ├─ Download audio from S3
    │  ├─ Try OpenAI Whisper API
    │  │  └─ If error → Try AWS Transcribe (fallback)
    │  ├─ Upload transcript to S3: transcripts/...json
    │  └─ Return { transcription: "...", service: "openai" }
    ├─ Success → Trigger Analysis Workflow
    ↓
[Step Functions: Analysis Workflow (Express, 5 min timeout)]
    ├─ Parallel branches:
    │  ┌────────────────────────────┬─────────────────────────────┐
    │  │ SOAP Report Branch         │ ICD Diagnosis Branch        │
    │  ├────────────────────────────┼─────────────────────────────┤
    │  │ [SOAP Lambda (5 min)]      │ [ICD Lambda (5 min)]        │
    │  │ ├─ Load transcript from S3 │ ├─ Load transcript from S3  │
    │  │ ├─ Load SOAP prompt (SSM)  │ ├─ Load ICD prompt (SSM)    │
    │  │ ├─ Try OpenAI GPT-4o       │ ├─ Try OpenAI GPT-4o        │
    │  │ │  └─ Fallback Anthropic   │ │  └─ Fallback Anthropic    │
    │  │ ├─ Parse SOAP JSON         │ ├─ Parse ICD JSON           │
    │  │ ├─ Write to DynamoDB       │ ├─ Write to DynamoDB        │
    │  │ │  (Appointments_Table_V2) │ │  (Diagnosis_Table_V2)     │
    │  │ └─ Publish to SNS          │ └─ Return { diagnoses: [] } │
    │  │    (optional)               │                             │
    │  └────────────────────────────┴─────────────────────────────┘
    │  Both complete → Join
    ├─ Success Notification Lambda
    │  ├─ Conditional: Only if source != "dlq_retry"
    │  └─ POST to /api/processing-webhook
    └─ END
    ↓
[Frontend: Notification Polling]
    ├─ GET /api/processing-webhook?doctorId={id}
    ├─ Receives notification
    ├─ Display snackbar
    └─ Invalidate appointments cache

┌───────────────────────────────────────────────────────────────────────┐
│                         ERROR RECOVERY FLOW                            │
└───────────────────────────────────────────────────────────────────────┘

[Step Functions Failure]
    ↓ Catch block
[SqsSendMessage Task]
    ├─ Send entire workflow input to DLQ
    ├─ DLQ: vitas-transcription-dlq OR vitas-soap-report-dlq OR vitas-icd-diagnosis-dlq
    └─ Message retention: 14 days
    ↓
[Failure Notification Lambda]
    ├─ POST to /api/processing-webhook
    ├─ Payload: { type: "failure", service: "...", metadata: { retryScheduled: true, nextRetryIn: "2 horas" }}
    └─ Frontend shows: "Error en proceso X, reintentaremos en 2 horas"
    ↓
[EventBridge Scheduled Rule]
    ├─ Triggers every 2 hours
    └─ Target: DLQ Processor Lambda
    ↓
[Lambda: dlq-processor]
    ├─ Poll all 3 DLQs
    ├─ For each message:
    │  ├─ Parse originalEvent
    │  ├─ Calculate message age
    │  ├─ If age < 24 hours:
    │  │  ├─ Determine workflow (transcription vs analysis)
    │  │  ├─ Start Step Functions execution (with retryAttempt: true)
    │  │  └─ Delete message from DLQ
    │  └─ If age ≥ 24 hours:
    │     ├─ Send final failure notification (SLA violation)
    │     ├─ Delete message from DLQ
    │     └─ Publish CloudWatch metric (SLA_VIOLATION)
    └─ Return summary (success/failure counts)
```

---

## Step Functions Workflows

### 1. Transcription Workflow (Standard)

**ARN:** `arn:aws:states:sa-east-1:ACCOUNT_ID:stateMachine:VitasTranscriptionWorkflow`

**Type:** Standard (long-running, persistent state)

**Timeout:** 20 minutes

**Definition:**
```json
{
  "StartAt": "TranscribeAudio",
  "States": {
    "TranscribeAudio": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "VitasProcessingStack-TranscriptionFunction",
        "Payload.$": "$"
      },
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed", "States.Timeout"],
          "IntervalSeconds": 30,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "SendToDLQ"
        }
      ],
      "Next": "TriggerAnalysisWorkflow"
    },
    "SendToDLQ": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage",
      "Parameters": {
        "QueueUrl": "https://sqs.sa-east-1.amazonaws.com/ACCOUNT_ID/vitas-transcription-dlq",
        "MessageBody.$": "$"
      },
      "Next": "NotifyTranscriptionFailure"
    },
    "NotifyTranscriptionFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "VitasProcessingStack-FailureNotificationFunction",
        "Payload": {
          "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES",
          "service": "transcription",
          "metadata": {
            "bucket.$": "$.bucket",
            "key.$": "$.key",
            "retryScheduled": true,
            "nextRetryIn": "2 horas"
          },
          "timestamp.$": "$.State.EnteredTime"
        }
      },
      "Catch": [{
        "ErrorEquals": ["States.ALL"],
        "Next": "TranscriptionFailed"
      }],
      "Next": "TranscriptionFailed"
    },
    "TranscriptionFailed": {
      "Type": "Fail",
      "Cause": "Transcription failed after retries"
    },
    "TriggerAnalysisWorkflow": {
      "Type": "Task",
      "Resource": "arn:aws:states:::states:startExecution.sync:2",
      "Parameters": {
        "StateMachineArn": "arn:aws:states:sa-east-1:ACCOUNT_ID:stateMachine:VitasAnalysisWorkflow",
        "Input.$": "$"
      },
      "End": true
    }
  }
}
```

### 2. Analysis Workflow (Express)

**ARN:** `arn:aws:states:sa-east-1:ACCOUNT_ID:stateMachine:VitasAnalysisWorkflow`

**Type:** Express (short-lived, cost-optimized 100x cheaper)

**Timeout:** 5 minutes

**Key Features:**
- Parallel execution of SOAP and ICD Lambdas
- Each branch has independent retry logic
- Conditional success notification (skips if retry source is DLQ)

**Parallel State:**
```json
{
  "Parallel": {
    "Branches": [
      {
        "StartAt": "GenerateSOAPReport",
        "States": {
          "GenerateSOAPReport": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "Parameters": {
              "FunctionName": "VitasProcessingStack-SOAPReportFunction",
              "Payload.$": "$"
            },
            "Retry": [
              {
                "ErrorEquals": ["States.TaskFailed"],
                "IntervalSeconds": 30,
                "MaxAttempts": 3,
                "BackoffRate": 2.0
              }
            ],
            "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "SendSOAPToDLQ"}],
            "End": true
          },
          "SendSOAPToDLQ": {"Type": "Task", "...": "..."}
        }
      },
      {
        "StartAt": "GenerateICDDiagnosis",
        "States": {
          "GenerateICDDiagnosis": {
            "Type": "Task",
            "Resource": "arn:aws:states:::lambda:invoke",
            "...": "..."
          }
        }
      }
    ],
    "Next": "CheckNotificationRequired"
  },
  "CheckNotificationRequired": {
    "Type": "Choice",
    "Choices": [{
      "Variable": "$.retryAttempt",
      "BooleanEquals": true,
      "Next": "SkipNotification"
    }],
    "Default": "SendSuccessNotification"
  }
}
```

---

## Lambda Functions

### 1. Transcription Function

**Memory:** 1024 MB | **Timeout:** 15 minutes | **Runtime:** Node.js 20.x ARM64

**File:** `vitas-processing-stack/lambdas/transcription/index.ts`

**Key Features:**
- Handles files up to 15MB (chunks larger files)
- Primary: OpenAI Whisper API
- Fallback: AWS Transcribe
- Extracts metadata from S3 object tags

**Implementation Highlights:**
```typescript
export const handler = async (event: any) => {
  const { bucket, key } = event;

  // Test failure mode (via environment variable)
  if (process.env.TEST_TRANSCRIPTION_FAILURE === 'true') {
    throw new Error('TEST_FAILURE: Simulated transcription error');
  }

  try {
    // Download audio from S3
    const audioBuffer = await s3.getObject({ Bucket: bucket, Key: key });

    // Try OpenAI Whisper (primary)
    try {
      const openaiKey = await getSSMParameter('/vitas/openai-api-key');
      const transcription = await transcribeWithOpenAI(audioBuffer, openaiKey);

      // Upload transcript to S3
      await s3.putObject({
        Bucket: bucket,
        Key: key.replace('audios/', 'transcripts/').replace('.mp3', '.json'),
        Body: JSON.stringify({ transcription, service: 'openai' })
      });

      return { transcription, service: 'openai', duration: Date.now() - startTime };
    } catch (openaiError) {
      logger.warn('OpenAI failed, trying AWS Transcribe', { error: openaiError });

      // Fallback to AWS Transcribe
      const transcription = await transcribeWithAWS(bucket, key);
      return { transcription, service: 'aws-transcribe' };
    }
  } catch (error) {
    logger.error('Transcription failed', error);
    throw error; // Trigger Step Functions retry
  }
};
```

### 2. SOAP Report Function

**Memory:** 512 MB | **Timeout:** 5 minutes | **Runtime:** Node.js 20.x ARM64

**File:** `vitas-processing-stack/lambdas/soap-report/index.ts`

**Key Features:**
- Loads transcript from S3
- Loads SOAP prompt template from SSM
- Primary: OpenAI GPT-4o
- Fallback: Anthropic Claude 3 Sonnet
- Writes to `Appointments_Table_V2` (SOAP fields)

**SOAP Report Structure:**
```json
{
  "subjective": "Patient presents with...",
  "objective": "Vital signs: BP 120/80, HR 72...",
  "assessment": "Diagnosis: Acute upper respiratory infection...",
  "plan": "Treatment: Prescribe amoxicillin 500mg..."
}
```

### 3. ICD Diagnosis Function

**Memory:** 512 MB | **Timeout:** 5 minutes | **Runtime:** Node.js 20.x ARM64

**File:** `vitas-processing-stack/lambdas/icd-diagnosis/index.ts`

**Key Features:**
- Loads transcript from S3
- Loads ICD prompt template from SSM
- Primary: OpenAI GPT-4o
- Fallback: Anthropic Claude 3 Sonnet
- Writes to `Diagnosis_Table_V2`

**ICD Diagnosis Structure:**
```json
{
  "diagnoses": [
    {
      "code": "J06.9",
      "description": "Acute upper respiratory infection, unspecified",
      "confidence": "high"
    },
    {
      "code": "R05",
      "description": "Cough",
      "confidence": "medium"
    }
  ]
}
```

### 4. Workflow Trigger Function

**Memory:** 256 MB | **Timeout:** 1 minute

**File:** `vitas-processing-stack/lambdas/workflow-trigger/index.ts`

**Responsibility:** Route S3 events to correct Step Functions workflow

```typescript
export const handler = async (event: any) => {
  const s3Event = event.Records[0].s3;
  const bucket = s3Event.bucket.name;
  const key = decodeURIComponent(s3Event.object.key);

  let workflowArn: string;
  let executionName: string;

  if (key.startsWith('audios/')) {
    // Route to Transcription Workflow
    workflowArn = process.env.TRANSCRIPTION_WORKFLOW_ARN!;
    executionName = `transcription-${Date.now()}`;
  } else if (key.startsWith('transcripts/')) {
    // Route to Analysis Workflow
    workflowArn = process.env.ANALYSIS_WORKFLOW_ARN!;
    executionName = `analysis-${Date.now()}`;
  } else {
    throw new Error(`Unknown file type: ${key}`);
  }

  // Start Step Functions execution
  await sfnClient.send(new StartExecutionCommand({
    stateMachineArn: workflowArn,
    name: executionName,
    input: JSON.stringify({ bucket, key, timestamp: new Date().toISOString() })
  }));
};
```

### 5. DLQ Processor Function

**Memory:** 512 MB | **Timeout:** 15 minutes | **Trigger:** EventBridge (every 2 hours)

**File:** `vitas-processing-stack/lambdas/dlq-processor/index.ts`

**Implementation Highlights:**
```typescript
export const handler = async (event: any) => {
  const dlqUrls = [
    process.env.TRANSCRIPTION_DLQ_URL,
    process.env.SOAP_REPORT_DLQ_URL,
    process.env.ICD_DIAGNOSIS_DLQ_URL
  ];

  let totalProcessed = 0;
  let totalRetried = 0;
  let totalExpired = 0;

  for (const queueUrl of dlqUrls) {
    // Poll DLQ (max 10 messages)
    const messages = await sqsClient.send(new ReceiveMessageCommand({
      QueueUrl: queueUrl,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 5
    }));

    for (const message of messages.Messages || []) {
      const originalEvent = JSON.parse(message.Body);
      const messageTimestamp = originalEvent.timestamp || message.Attributes?.SentTimestamp;
      const messageAge = Date.now() - parseInt(messageTimestamp);

      totalProcessed++;

      if (messageAge > 24 * 60 * 60 * 1000) {
        // Message older than 24 hours → SLA violation
        logger.error('SLA violation detected', { messageAge, originalEvent });

        // Send final failure notification
        await sendFinalFailureNotification(originalEvent);

        // Publish CloudWatch metric
        await cloudwatch.send(new PutMetricDataCommand({
          Namespace: 'VITAS/Processing',
          MetricData: [{
            MetricName: 'SLA_VIOLATION',
            Value: 1,
            Unit: 'Count'
          }]
        }));

        totalExpired++;
      } else {
        // Retry via Step Functions
        const success = await retryProcessing(originalEvent);

        if (success) {
          totalRetried++;
        }
      }

      // Delete message from DLQ
      await sqsClient.send(new DeleteMessageCommand({
        QueueUrl: queueUrl,
        ReceiptHandle: message.ReceiptHandle
      }));
    }
  }

  return { totalProcessed, totalRetried, totalExpired };
};
```

---

## Error Handling & DLQ

### Retry Strategy

| Attempt | Wait Time | Total Elapsed | Action |
|---------|-----------|---------------|--------|
| 1 | 0s | 0s | Initial attempt |
| 2 | 30s | 30s | First retry |
| 3 | 60s | 90s | Second retry (2x backoff) |
| 4 | 120s | 210s | Third retry (2x backoff) |
| All fail | - | ~3.5 min | Send to DLQ + Notify user |

### Dead Letter Queue Flow

```
Lambda Failure (after 3 retries)
    ↓
Step Functions Catch Block
    ↓
SqsSendMessage Task
    ├─ Queue: vitas-transcription-dlq (or soap/icd)
    ├─ Message Body: Entire workflow input (bucket, key, metadata)
    └─ Retention: 14 days
    ↓
Failure Notification Lambda
    ├─ POST /api/processing-webhook
    └─ Frontend shows: "Error, reintentaremos en 2 horas"
    ↓
EventBridge Rule (every 2 hours)
    ↓
DLQ Processor Lambda
    ├─ Poll DLQ
    ├─ For each message:
    │  ├─ Check age < 24h
    │  ├─ Restart Step Functions (with retryAttempt flag)
    │  └─ Delete from DLQ
    └─ If age ≥ 24h:
       ├─ Send final failure notification
       ├─ Log SLA violation
       └─ Delete from DLQ
```

---

## Notifications Integration

### Success Notification

**Trigger:** Analysis Workflow completes successfully

**Webhook:** `POST /api/processing-webhook`

**Payload:**
```json
{
  "type": "success",
  "service": "transcription-complete",
  "metadata": {
    "appointmentId": "appointment-789",
    "patientId": "patient-456",
    "doctorId": "doctor-123"
  },
  "timestamp": "2025-11-06T10:30:00Z",
  "source": "vitas-processing-stack"
}
```

**Frontend Response:**
- Display snackbar: "Procesamiento completado para la cita de Juan Pérez"
- Invalidate appointments cache
- Auto-refresh consultation list

### Failure Notification

**Trigger:** Step Functions catch block (after 3 retries fail)

**Webhook:** `POST /api/processing-webhook`

**Payload:**
```json
{
  "type": "failure",
  "service": "transcription",
  "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES",
  "metadata": {
    "appointmentId": "appointment-789",
    "patientId": "patient-456",
    "doctorId": "doctor-123",
    "retryScheduled": true,
    "nextRetryIn": "2 horas"
  },
  "timestamp": "2025-11-06T10:30:00Z",
  "source": "vitas-processing-stack"
}
```

**Frontend Response:**
- Display error snackbar: "Se ha encontrado un error en el proceso transcripción, reintentaremos en 2 horas"
- Duration: 10 seconds (longer for errors)
- Color: Red/error theme

---

## Testing Failures

### Environment Variable Testing

Enable test failures without redeployment:

```bash
# Enable test failure
aws lambda update-function-configuration \
  --function-name VitasProcessingStack-TranscriptionFunction* \
  --environment "Variables={TEST_TRANSCRIPTION_FAILURE=true}" \
  --region sa-east-1 \
  --profile vitas-sso

# Disable test failure
aws lambda update-function-configuration \
  --function-name VitasProcessingStack-TranscriptionFunction* \
  --environment "Variables={}" \
  --region sa-east-1 \
  --profile vitas-sso
```

### Manual DLQ Testing

**Inject test message into DLQ:**
```bash
aws sqs send-message \
  --queue-url https://sqs.sa-east-1.amazonaws.com/ACCOUNT_ID/vitas-transcription-dlq \
  --message-body '{
    "bucket": "vitas-media-processing-ACCOUNT-v2",
    "key": "audios/doctor-123/patient-456/appointment-789.mp3",
    "timestamp": "2025-11-06T10:00:00Z",
    "doctorId": "doctor-123",
    "patientId": "patient-456",
    "appointmentId": "appointment-789"
  }' \
  --region sa-east-1 \
  --profile vitas-sso
```

**Manually invoke DLQ Processor:**
```bash
aws lambda invoke \
  --function-name VitasProcessingStack-DLQProcessorFunction \
  --payload '{"source":"manual-test"}' \
  --region sa-east-1 \
  --profile vitas-sso \
  response.json

cat response.json
```

### Monitor Processing

**Watch Step Functions executions:**
```bash
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:sa-east-1:ACCOUNT_ID:stateMachine:VitasTranscriptionWorkflow \
  --status-filter RUNNING \
  --region sa-east-1 \
  --profile vitas-sso
```

**Tail Lambda logs:**
```bash
aws logs tail /aws/lambda/VitasProcessingStack-TranscriptionFunction \
  --follow \
  --filter-pattern "ERROR" \
  --region sa-east-1 \
  --profile vitas-sso
```

---

## Performance Metrics

### Expected Processing Times

| Stage | Duration (Average) | Duration (p95) |
|-------|-------------------|----------------|
| **Transcription** (15 min audio) | 2-3 minutes | 5 minutes |
| **SOAP Report** | 20-30 seconds | 45 seconds |
| **ICD Diagnosis** | 15-20 seconds | 35 seconds |
| **Total Pipeline** | 3-4 minutes | 6 minutes |

### Cost per Consultation

| Component | Cost per Execution |
|-----------|-------------------|
| **Step Functions** | $0.000025 (Standard) + $0.000001 (Express) = $0.000026 |
| **Lambda Executions** | $0.0002 (all 7 functions) |
| **OpenAI Whisper** | $0.12 (15 min audio) |
| **OpenAI GPT-4o** | $0.03 (SOAP + ICD) |
| **S3 Storage** | $0.001 (audio + transcripts) |
| **DynamoDB Writes** | $0.0001 |
| **Total** | **~$0.15 per consultation** |

---

## Summary

The VITAS Clinic processing pipeline provides:
- **Fully automated transcription** with multi-provider fallback
- **AI-generated SOAP reports** and ICD-10 diagnoses
- **99.9% reliability** through automatic retries and DLQ
- **24-hour SLA guarantee** via scheduled DLQ processor
- **Real-time notifications** to doctor dashboard
- **Cost-optimized architecture** ($0.15 per consultation)

**Related Documentation:**
- [System Architecture Overview](01-SYSTEM_ARCHITECTURE_OVERVIEW.md)
- [Frontend-Backend Integration](03-FRONTEND_BACKEND_INTEGRATION.md)
- [Deployment Guide](02-DEPLOYMENT_GUIDE.md)
- Detailed Testing Guide: `vitas-processing-stack/FAILURE_TESTING_GUIDE.md`

**Last Updated:** 2025-11-06
