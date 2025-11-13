# VITAS Processing Stack - Comprehensive Analysis

## Executive Summary

The VITAS Processing Stack is an AWS-native healthcare data processing pipeline built with AWS CDK (Infrastructure as Code). It orchestrates medical audio transcription, SOAP report generation, and ICD-10 diagnosis classification through a sophisticated serverless architecture combining Step Functions, Lambda functions, and managed AWS services.

**Key Architecture Pattern**: Event-driven, fault-tolerant microservices with 24-hour SLA guarantee via Dead Letter Queue (DLQ) processing.

---

## 1. CDK Stack Structure

### Main Stack Entry Point
**File**: `/lib/vitas-processing-stack-stack.ts`
**Class**: `VitasProcessingStackStack extends cdk.Stack`

#### Stack Configuration
- **Account**: 197517026286
- **Region**: sa-east-1 (São Paulo)
- **Tags**: 
  - `client: vitas-clinic`
  - `stack: processing`

#### Core Components Created in Order

```typescript
// Resource Imports (Account-agnostic cross-stack references)
1. AppointmentsTable (imported as ITable) - "Appointments_Table_V2"
2. DiagnosisTable (imported as ITable) - "Diagnosis_Table_V2"
3. SNS Topic (imported via CloudFormation export) - "vitas-main-stack-NotificationTopic"
4. S3 Media Bucket (new) - "vitas-media-processing-{account}-v2"
5. SSM Parameters (new) - 5 parameters for secrets and prompts
6. Lambda Layer (new) - "ProcessingLayer" with Node.js dependencies
7. Lambda Functions (7 total)
8. Step Functions State Machines (2 workflows)
9. EventBridge Rules (DLQ retry scheduler)
10. CloudWatch Dashboard & Alarms
```

---

## 2. S3 Configuration

### Bucket Details
**Name Pattern**: `vitas-media-processing-{account}-v2`

#### Encryption & Transfer
- **Encryption**: S3-managed (SSE-S3)
- **Transfer Acceleration**: Enabled
- **Versioning**: Disabled
- **Removal Policy**: DESTROY (development)

#### CORS Configuration
```typescript
{
  allowedHeaders: ['*'],
  allowedMethods: [POST, PUT, GET],
  allowedOrigins: ['*']
}
```

#### Lifecycle Rules
```typescript
{
  id: 'DeleteOldFiles',
  expiration: 90 days,
  transitions: [
    {
      storageClass: INFREQUENT_ACCESS,
      transitionAfter: 30 days
    }
  ]
}
```

#### Folder Structure
```
s3://vitas-media-processing-{account}-v2/
├── audios/                          # Input audio files (.mp3)
├── transcripts/                     # Transcribed text files (.json)
├── soap-reports/                    # SOAP report outputs
├── icd-diagnoses/                   # ICD diagnosis outputs
└── transcribe-output/               # AWS Transcribe interim results
```

#### Event Triggers
Two S3 event notification triggers configured via Lambda destinations:

1. **Audio Files Trigger**
   - Event: `ObjectCreated`
   - Filter: `prefix: 'audios/', suffix: '.mp3'`
   - Target: `workflowTrigger` Lambda
   - Triggers: Transcription Workflow

2. **Transcript Files Trigger**
   - Event: `ObjectCreated`
   - Filter: `prefix: 'transcripts/', suffix: '.json'`
   - Target: `workflowTrigger` Lambda
   - Triggers: Analysis Workflow

---

## 3. Lambda Functions

### Overview
7 Lambda functions deployed with:
- **Runtime**: Node.js 20.x
- **Architecture**: ARM_64 (Graviton processors)
- **Layer**: ProcessingLayer (OpenAI, Anthropic, AWS SDK)

### 3.1 Transcription Function

**Location**: `/lambdas/transcription/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 15 minutes,
  memorySize: 1024 MB,
  runtime: NODEJS_20_X,
  architecture: ARM_64
}
```

#### Environment Variables
```typescript
{
  OPENAI_API_KEY_PARAMETER: '/vitas/openai-api-key',
  MEDIA_BUCKET_NAME: mediaBucket.bucketName,
  WEBHOOK_URL_PARAMETER: '/vitas/webhook-url',
  FAILURE_NOTIFICATION_FUNCTION_NAME: failureNotificationFunction.functionName
}
```

#### Functionality
1. **Input Event Processing**
   - Accepts S3 event or direct invocation with `{bucket, key}`
   - Extracts metadata from S3 object tags

2. **Audio Transcription Pipeline**
   ```
   OpenAI Whisper (Primary)
   └─ Fallback to AWS Transcribe (on non-retryable errors)
   ```

3. **Processing Logic**
   - Downloads audio file in 15MB chunks
   - Sends each chunk to OpenAI Whisper API (model: "whisper-1")
   - Language: Spanish (es)
   - Concatenates chunk transcriptions
   - Falls back to AWS Transcribe if OpenAI fails with non-retryable errors

4. **Output**
   - Saves to S3: `transcripts/transcript_{appointmentId}.json`
   - JSON structure:
   ```json
   {
     "transcription": "full text...",
     "service": "openai|aws_transcribe",
     "timestamp": "ISO8601",
     "metadata": {
       "doctorId": "...",
       "patientId": "...",
       "appointmentId": "..."
     }
   }
   ```

5. **Error Handling**
   - Retryable errors: rate_limit_exceeded, timeout, network_error
   - Triggers Step Functions retry (3 attempts, 10-30s intervals, 2.0 backoff)
   - Non-retryable: Sent to DLQ for scheduled retry
   - Calls failure-notification Lambda on final failure

#### Permissions Granted
- S3 read/write: Media bucket
- SSM read: `/vitas/openai-api-key`
- Transcribe: StartTranscriptionJob, GetTranscriptionJob
- Lambda invoke: Failure notification function
- CloudWatch: Logs

#### DLQ Configuration
**Queue**: `vitas-transcription-dlq-{account}`
- Retention: 14 days
- Max receive count: (default, triggers DLQ after Lambda retry exhaustion)

---

### 3.2 SOAP Report Function

**Location**: `/lambdas/soap-report/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 5 minutes,
  memorySize: 512 MB
}
```

#### Environment Variables
```typescript
{
  OPENAI_API_KEY_PARAMETER: '/vitas/openai-api-key',
  ANTHROPIC_API_KEY_PARAMETER: '/vitas/anthropic-api-key',
  APPOINTMENTS_TABLE_NAME: appointmentsTable.tableName,
  MEDIA_BUCKET_NAME: mediaBucket.bucketName,
  SNS_TOPIC_ARN: snsTopicArn,
  SOAP_PROMPT_PARAMETER: '/vitas/prompts/soap-report',
  WEBHOOK_URL_PARAMETER: '/vitas/webhook-url',
  FAILURE_NOTIFICATION_FUNCTION_NAME: failureNotificationFunction.functionName
}
```

#### Functionality
1. **Input Processing**
   - Reads transcription file from S3
   - Extracts metadata from S3 object

2. **SOAP Generation Pipeline**
   ```
   GPT-4 (gpt-4.1) with temperature=0.3 (Primary)
   └─ Fallback to Claude Haiku (claude-haiku-4-5-20251001)
   ```

3. **Processing Details**
   - Sends full transcription with SOAP prompt to GPT-4
   - Extracts structured output using XML tag parsing
   ```html
   <SOAP_Report>...</SOAP_Report>
   <Symptoms>...</Symptoms>
   ```

4. **Data Persistence**
   - **S3**: `soap-reports/soap-report-{appointmentId}.json`
   - **DynamoDB**: Appointments_Table_V2
   ```typescript
   {
     appointment_id: appointmentId (PK),
     patient_id: patientId,
     doctor_id: doctorId,
     created_at: ISO8601,
     soap_report_key: S3 path,
     symptoms: comma-separated string,
     transcription_key: S3 path,
     isFromSystem: true
   }
   ```

5. **Notifications**
   - SNS Publish to configured topic with doctor ID attribute
   - Success notification via webhook (via success-notification Lambda)
   - Failure notification via failure-notification Lambda

#### Permissions Granted
- S3 read/write: Media bucket
- DynamoDB write: Appointments table
- SNS publish: SNS topic ARN
- SSM read: All `/vitas/*` parameters
- Lambda invoke: Failure notification function

#### DLQ Configuration
**Queue**: `vitas-soap-report-dlq-{account}`
- Retention: 14 days

---

### 3.3 ICD Diagnosis Function

**Location**: `/lambdas/icd-diagnosis/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 5 minutes,
  memorySize: 512 MB
}
```

#### Environment Variables
```typescript
{
  OPENAI_API_KEY_PARAMETER: '/vitas/openai-api-key',
  ANTHROPIC_API_KEY_PARAMETER: '/vitas/anthropic-api-key',
  DIAGNOSIS_TABLE_NAME: diagnosisTable.tableName,
  ICD_PROMPT_PARAMETER: '/vitas/prompts/icd-diagnosis',
  WEBHOOK_URL_PARAMETER: '/vitas/webhook-url',
  FAILURE_NOTIFICATION_FUNCTION_NAME: failureNotificationFunction.functionName
}
```

#### Functionality
1. **ICD-10 Classification Pipeline**
   ```
   GPT-4 (gpt-4.1) with temperature=0.1 (Primary)
   └─ Fallback to Claude Haiku (claude-haiku-4-5-20251001)
   ```

2. **Processing Logic**
   - Analyzes transcription for medical conditions
   - Low temperature (0.1) for precise classification
   - Extracts diagnoses in format:
   ```
   DIAGNOSTICO_1: Enfermedad (CIE-10 código) | TIPO: Presuntivo|Definitivo|Repetitivo
   DIAGNOSTICO_2: ...
   DIAGNOSTICO_3: ...
   ```

3. **Diagnosis Types**
   - **Presuntivo**: Possible/probable conditions (uncertain)
   - **Definitivo**: Confirmed diagnoses (certain)
   - **Repetitivo**: Chronic/pre-existing conditions

4. **Data Persistence**
   - **DynamoDB**: Diagnosis_Table_V2
   ```typescript
   {
     appointment_id: appointmentId (PK),
     patient_id: patientId,
     diagnosis: pipe-separated string,
     created_at: ISO8601,
     isFromSystem: true
   }
   ```

5. **Output Format**
   - Returns `icdDiagnosisKey`: `icd-diagnoses/icd-diagnosis-{appointmentId}.json`

#### Permissions Granted
- S3 read: Media bucket (transcripts)
- DynamoDB write: Diagnosis table
- SSM read: All `/vitas/*` parameters
- Lambda invoke: Failure notification function

#### DLQ Configuration
**Queue**: `vitas-icd-diagnosis-dlq-{account}`
- Retention: 14 days

---

### 3.4 Workflow Trigger Function

**Location**: `/lambdas/workflow-trigger/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 1 minute,
  memorySize: 256 MB
}
```

#### Environment Variables
```typescript
{
  TRANSCRIPTION_WORKFLOW_ARN: transcriptionWorkflow.stateMachineArn,
  ANALYSIS_WORKFLOW_ARN: analysisWorkflow.stateMachineArn
}
```

#### Functionality
1. **Event Detection**
   - S3 event extraction and normalization
   - Supports S3 events and direct invocation

2. **Workflow Routing**
   ```typescript
   function determineWorkflow(key: string) {
     if (key.startsWith('audios/') && key.endsWith('.mp3')) 
       return 'transcription'
     else if (key.startsWith('transcripts/') && key.endsWith('.json')) 
       return 'analysis'
   }
   ```

3. **Step Functions Invocation**
   - Generates unique execution name: `{type}-{timestamp}-{randomSuffix}`
   - Passes S3 event details to workflow input

#### Permissions Granted
- Step Functions: StartExecution for both workflows
- CloudWatch: Logs

---

### 3.5 Failure Notification Function

**Location**: `/lambdas/failure-notification/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 1 minute,
  memorySize: 256 MB
}
```

#### Environment Variables
```typescript
{
  WEBHOOK_URL_PARAMETER: '/vitas/webhook-url',
  VERCEL_BYPASS_SECRET_PARAMETER: '/vitas/vercel-bypass-secret'
}
```

#### Functionality
1. **Event Input Structure**
   ```typescript
   interface NotificationEvent {
     error: string,
     service: string,
     metadata: {
       appointmentId?: string,
       patientId?: string,
       doctorId?: string,
       [key: string]: any
     },
     timestamp: string
   }
   ```

2. **Webhook Notification**
   - HTTP POST to configured webhook URL
   - Custom headers:
   ```
   Content-Type: application/json
   User-Agent: VITAS-Processing-Stack/1.0
   X-Source: aws-lambda
   X-Event-Type: processing-failure
   X-Webhook-Signature: vitas-processing-stack
   x-vercel-protection-bypass: {VERCEL_BYPASS_SECRET} (if available)
   ```

3. **Payload Structure**
   ```json
   {
     "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES",
     "service": "transcription|soap-report|icd-diagnosis",
     "metadata": { /* original event metadata */ },
     "timestamp": "ISO8601",
     "source": "vitas-processing-stack",
     "type": "failure"
   }
   ```

4. **Error Handling**
   - Logs and rethrows errors (causing Lambda failure, which triggers DLQ)
   - Vercel bypass secret is optional

#### Permissions Granted
- SSM read: `/vitas/webhook-url`, `/vitas/vercel-bypass-secret`
- CloudWatch: Logs

---

### 3.6 Success Notification Function

**Location**: `/lambdas/success-notification/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 1 minute,
  memorySize: 256 MB
}
```

#### Similar to Failure Notification
- Same webhook infrastructure
- Event type: "success"
- Metadata includes processing metrics (bucket, keys, provider, processingTime)
- Non-fatal on error (doesn't rethrow)

#### Webhook Headers
```
X-Event-Type: processing-success
```

---

### 3.7 DLQ Processor Function

**Location**: `/lambdas/dlq-processor/index.ts`
**Handler**: `index.handler`

#### Configuration
```typescript
{
  timeout: 15 minutes,
  memorySize: 512 MB
}
```

#### Environment Variables
```typescript
{
  TRANSCRIPTION_DLQ_URL: transcriptionFunction.deadLetterQueue?.queueUrl,
  SOAP_REPORT_DLQ_URL: soapReportFunction.deadLetterQueue?.queueUrl,
  ICD_DIAGNOSIS_DLQ_URL: icdDiagnosisFunction.deadLetterQueue?.queueUrl,
  STACK_NAME: this.stackName,
  FAILURE_NOTIFICATION_FUNCTION_NAME: failureNotificationFunction.functionName,
  TRANSCRIPTION_WORKFLOW_ARN: workflows.transcriptionWorkflow.stateMachineArn,
  ANALYSIS_WORKFLOW_ARN: workflows.analysisWorkflow.stateMachineArn
}
```

#### Functionality
1. **DLQ Message Processing**
   - Polls up to 10 messages per queue
   - Evaluates message age against SLA thresholds

2. **SLA Management**
   - Warning threshold: 12 hours
   - Violation threshold: 24 hours
   - Max retention: 14 days (queue setting)

3. **Retry Logic**
   ```typescript
   if (messageAge > 24 hours) {
     // Send final failure notification
     // Delete message (stop retrying)
   } else if (messageAge > 12 hours) {
     // Log warning
     // Attempt retry
   } else {
     // Attempt retry
   }
   ```

4. **Retry Execution**
   - Invokes appropriate Step Functions workflow
   - Adds retry metadata to event:
   ```typescript
   {
     retryAttempt: true,
     retryTimestamp: ISO8601,
     ...originalEvent
   }
   ```

5. **CloudWatch Metrics**
   - Publishes SLA compliance metrics
   - Metric: `VITAS/Processing/FilesOlderThan12Hours`
   - Dimensions: `ProcessingStack: {stack_name}`

#### Permissions Granted
- SQS: ReceiveMessage, DeleteMessage, GetQueueAttributes (all DLQs)
- Step Functions: StartExecution (both workflows)
- CloudWatch: PutMetricData
- Lambda invoke: Failure notification function

#### Processing Results
Returns structured response:
```json
{
  "statusCode": 200,
  "body": {
    "message": "DLQ processing completed",
    "results": {
      "transcriptionDLQ": { "processed": 0, "failed": 0 },
      "soapReportDLQ": { "processed": 0, "failed": 0 },
      "icdDiagnosisDLQ": { "processed": 0, "failed": 0 }
    },
    "totalProcessed": 0,
    "totalFailed": 0
  }
}
```

---

## 4. Step Functions Workflows

### Workflow Overview
Two complementary state machines:
1. **TranscriptionWorkflow** (Standard type) - Long-running audio transcription
2. **AnalysisWorkflow** (Express type) - Fast parallel text analysis

### 4.1 Transcription Workflow (Standard)

**Type**: STANDARD (for long-running executions)
**Timeout**: 20 minutes
**Logs**: CloudWatch Log Group with ALL level, execution data included

#### State Machine Definition

```
TranscribeAudio
  ├─ Success
  │  └─ TriggerAnalysis (Express workflow invocation)
  │     └─ NotifyTranscriptionSuccess
  │        └─ Succeed
  └─ Retry (3 attempts)
     └─ Catch
        └─ SendTranscriptionToDLQ
           └─ NotifyTranscriptionFailure
              └─ Fail
```

#### Detailed States

1. **TranscribeAudio** (Lambda Invoke)
   - Function: `transcription`
   - Output path: `$.Payload`
   - Retry policy:
   ```typescript
   {
     errors: ['States.TaskFailed'],
     interval: 30 seconds,
     maxAttempts: 3,
     backoffRate: 2.0
   }
   ```

2. **SendTranscriptionToDLQ** (SQS Send Message)
   - Sends to `transcription.deadLetterQueue`
   - Message: `$.` (entire state input)

3. **NotifyTranscriptionFailure** (Lambda Invoke)
   - Function: `failureNotification`
   - Payload structure:
   ```json
   {
     "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES",
     "service": "transcription",
     "metadata": {
       "bucket": "$.bucket",
       "key": "$.key",
       "retryScheduled": true,
       "nextRetryIn": "2 horas"
     },
     "timestamp": "$$.State.EnteredTime"
   }
   ```

4. **TriggerAnalysis** (Step Functions Start Execution)
   - State machine: `analysisWorkflow`
   - Pattern: REQUEST_RESPONSE (synchronous)
   - Input:
   ```json
   {
     "bucket": "$.body.bucket",
     "key": "$.body.transcriptionKey",
     "source": "transcription_workflow"
   }
   ```
   - Result path: `$.analysisResult` (preserves original output)

5. **NotifyTranscriptionSuccess** (Lambda Invoke)
   - Function: `successNotification`
   - Payload:
   ```json
   {
     "service": "transcription-complete",
     "metadata": {
       "bucket": "$.body.bucket",
       "transcriptionKey": "$.body.transcriptionKey",
       "appointmentId": "$.body.appointmentId",
       "patientId": "$.body.patientId",
       "doctorId": "$.body.doctorId",
       "provider": "$.body.service",
       "processingTime": "$.body.processingTime"
     },
     "timestamp": "$$.State.EnteredTime"
   }
   ```

#### Input/Output Examples

**Input** (from S3 event):
```json
{
  "bucket": "vitas-media-processing-197517026286-v2",
  "key": "audios/appointment_123.mp3",
  "eventName": "ObjectCreated:Put",
  "eventTime": "2024-11-11T10:30:00Z",
  "workflowType": "transcription"
}
```

**Output** (from TranscribeAudio Lambda):
```json
{
  "statusCode": 200,
  "body": {
    "message": "Transcription completed successfully",
    "service": "openai",
    "processingTime": 45000,
    "appointmentId": "apt-123",
    "patientId": "pat-456",
    "doctorId": "doc-789",
    "bucket": "vitas-media-processing-197517026286-v2",
    "transcriptionKey": "transcripts/transcript_apt-123.json",
    "provider": "openai",
    "metadata": { /* ... */ }
  }
}
```

---

### 4.2 Analysis Workflow (Express)

**Type**: EXPRESS (fast synchronous processing)
**Timeout**: 5 minutes
**Logs**: CloudWatch Log Group with ALL level, execution data included

#### State Machine Definition

```
ParallelAnalysis
├─ Branch 1: GenerateSOAP
│  ├─ Retry (3 attempts)
│  └─ Catch
│     └─ SendSOAPToDLQ
│        └─ NotifySOAPFailure
│           └─ Pass (SOAPFailed)
└─ Branch 2: GenerateICD
   ├─ Retry (3 attempts)
   └─ Catch
      └─ SendICDToDLQ
         └─ NotifyICDFailure
            └─ Pass (ICDFailed)

After Parallel Completion:
CheckSource
├─ If source=='transcription_workflow'
│  └─ Succeed (AnalysisComplete)
└─ Otherwise
   └─ NotifyAnalysisSuccess
      └─ Succeed (AnalysisCompleteWithNotification)
```

#### Detailed States

1. **ParallelAnalysis** (Parallel state)
   - Executes both branches concurrently
   - Waits for both to complete

2. **GenerateSOAP** (Lambda Invoke)
   - Function: `soapReport`
   - Retry policy (same as transcription)
   - Input: `{ bucket, key, source }`

3. **SendSOAPToDLQ** (SQS Send Message)
   - Sends to `soapReport.deadLetterQueue`

4. **NotifySOAPFailure** (Lambda Invoke)
   - Payload:
   ```json
   {
     "error": "SOAP_GENERATION_FAILED_AFTER_RETRIES",
     "service": "soap-report",
     "metadata": {
       "bucket": "$.bucket",
       "key": "$.key",
       "retryScheduled": true,
       "nextRetryIn": "2 horas"
     },
     "timestamp": "$$.State.EnteredTime"
   }
   ```
   - Catch: Continue on error

5. **GenerateICD** (Lambda Invoke)
   - Function: `icdDiagnosis`
   - Similar structure to SOAP

6. **CheckSource** (Choice state)
   - Condition: `$.source == "transcription_workflow"`
   - Purpose: Determine if called from transcription workflow
   - If yes: Skip success notification (already sent by transcription workflow)
   - If no: Send success notification (direct analysis request)

7. **NotifyAnalysisSuccess** (Lambda Invoke)
   - Function: `successNotification`
   - Payload includes both SOAP and ICD results

#### Input/Output

**Input** (from Transcription Workflow):
```json
{
  "bucket": "vitas-media-processing-197517026286-v2",
  "key": "transcripts/transcript_apt-123.json",
  "source": "transcription_workflow"
}
```

**Output** (from both Lambda functions):
```json
[
  {
    "Payload": {
      "statusCode": 200,
      "body": {
        "soapReportKey": "soap-reports/soap-report-apt-123.json",
        "service": "openai",
        "processingTime": 12000,
        "metadata": { /* ... */ }
      }
    }
  },
  {
    "Payload": {
      "statusCode": 200,
      "body": {
        "icdDiagnosisKey": "icd-diagnoses/icd-diagnosis-apt-123.json",
        "diagnoses": "DIAGNOSTICO_1: ... || DIAGNOSTICO_2: ...",
        "service": "openai",
        "processingTime": 8000,
        "metadata": { /* ... */ }
      }
    }
  }
]
```

---

### Workflow Retry & Error Handling

#### Retry Strategy (Both Workflows)
```typescript
{
  errors: ['States.TaskFailed'],
  interval: cdk.Duration.seconds(10 or 30),
  maxAttempts: 3,
  backoffRate: 2.0  // Exponential backoff: 10->20->40s or 30->60->120s
}
```

#### Error Flow
```
Lambda Execution
└─ Step Functions Retry (3 attempts with backoff)
   └─ Failure (after exhaustion)
      └─ Send to DLQ
         └─ Notify failure via webhook
            └─ Mark as failed in Step Functions
               └─ EventBridge Scheduled DLQ Processor (every 2 hours)
                  └─ Retry workflow up to 24 hours
                     └─ Final failure notification after SLA violation
```

---

## 5. SQS Dead Letter Queues (DLQs)

### DLQ Configuration

#### Three DLQs Created (One per Processor Lambda)

1. **Transcription DLQ**
   - Queue: `vitas-transcription-dlq-{account}`
   - Retention: 14 days
   - Linked to: `transcriptionFunction`

2. **SOAP Report DLQ**
   - Queue: `vitas-soap-report-dlq-{account}`
   - Retention: 14 days
   - Linked to: `soapReportFunction`

3. **ICD Diagnosis DLQ**
   - Queue: `vitas-icd-diagnosis-dlq-{account}`
   - Retention: 14 days
   - Linked to: `icdDiagnosisFunction`

### DLQ Message Flow

```
Lambda Invocation Fails
└─ Step Functions Catch Block
   └─ SqsSendMessage (sends to DLQ)
      ├─ Message retained for 14 days
      └─ EventBridge scheduled rule (every 2 hours)
         └─ DLQ Processor Lambda
            ├─ Retrieves messages (max 10 per queue)
            ├─ Evaluates age:
            │  ├─ <12h: Retry workflow
            │  ├─ 12-24h: Log warning + retry
            │  └─ >24h: SLA violation + final notification
            ├─ Deletes on success
            └─ Leaves in queue on retry failure
```

### DLQ Configuration (config.ts)

```typescript
export const DLQ_CONFIG = {
  SLA: {
    WARNING_THRESHOLD_MS: 12 * 60 * 60 * 1000,      // 12 hours
    VIOLATION_THRESHOLD_MS: 24 * 60 * 60 * 1000,    // 24 hours
  },
  PROCESSING: {
    MAX_MESSAGES_PER_QUEUE: 10,
    WAIT_TIME_SECONDS: 5
  },
  METRICS: {
    NAMESPACE: 'VITAS/Processing',
    METRIC_NAME_FILES_OVER_12H: 'FilesOlderThan12Hours',
    METRIC_NAME_FILES_OVER_24H: 'FilesOlderThan24Hours'
  },
  RETRY: {
    MAX_ATTEMPTS: 12,          // With 2-hour intervals = 24 hours
    INTERVAL_HOURS: 2
  }
}
```

---

## 6. EventBridge Scheduled Rules

### DLQ Retry Rule

**Rule Name**: `vitas-dlq-retry-{account}`
**Schedule**: Every 2 hours (24-hour coverage)
**Pattern**: `rate(2 hours)`

#### Configuration
```typescript
{
  ruleName: 'vitas-dlq-retry-{account}',
  description: 'Retry failed processing every 2 hours for 24-hour SLA guarantee',
  schedule: Schedule.rate(Duration.hours(2))
}
```

#### Target
- **Function**: `DLQProcessorFunction`
- **Event Input**: RuleTargetInput.fromObject
```json
{
  "source": "eventbridge-scheduled",
  "retryType": "scheduled",
  "timestamp": "$$.time"
}
```

#### Retry Schedule
```
Hour 0:   Initial failure → DLQ
Hour 2:   Attempt 1 (via scheduled rule)
Hour 4:   Attempt 2
Hour 6:   Attempt 3
...
Hour 22:  Attempt 11
Hour 24:  Attempt 12 (final)
         If still failing → SLA violation notification
```

---

## 7. SSM Parameter Store Configuration

### Parameters Created

1. **OpenAI API Key**
   - Name: `/vitas/openai-api-key`
   - Type: String (manually change to SecureString post-deployment)
   - Initial Value: `PLACEHOLDER-SET-AFTER-DEPLOYMENT`
   - Used by: Transcription, SOAP Report, ICD Diagnosis

2. **Anthropic API Key**
   - Name: `/vitas/anthropic-api-key`
   - Type: String (manually change to SecureString post-deployment)
   - Initial Value: `PLACEHOLDER-SET-AFTER-DEPLOYMENT`
   - Used by: SOAP Report, ICD Diagnosis (fallbacks)

3. **Webhook URL**
   - Name: `/vitas/webhook-url`
   - Type: String
   - Default: `https://your-vercel-app.vercel.app/api/processing-webhook`
   - Used by: Notification Lambdas

4. **Vercel Bypass Secret**
   - Name: `/vitas/vercel-bypass-secret`
   - Type: String
   - Initial Value: `PLACEHOLDER-SET-AFTER-DEPLOYMENT`
   - Used by: Failure/Success notification Lambdas
   - Purpose: Bypass Vercel Protection for webhook calls

5. **SOAP Report Prompt**
   - Name: `/vitas/prompts/soap-report`
   - Type: String
   - Content: Loaded from `prompts/soap-report-prompt.txt` at deployment
   - Used by: SOAP Report Lambda

6. **ICD Diagnosis Prompt**
   - Name: `/vitas/prompts/icd-diagnosis`
   - Type: String
   - Content: Loaded from `prompts/icd-diagnosis-prompt.txt` at deployment
   - Used by: ICD Diagnosis Lambda

### IAM Permissions

All Lambdas have SSM parameter read access:
```typescript
actions: ['ssm:GetParameter'],
resources: [
  'arn:aws:ssm:{region}:{account}:parameter/vitas/*'
]
```

---

## 8. External Integrations

### 8.1 OpenAI Integration

#### Models Used

1. **Whisper-1** (Transcription)
   - Purpose: Audio transcription
   - Language: Spanish (es)
   - Response format: JSON
   - Processing: Chunked (15MB chunks)

2. **GPT-4 (gpt-4.1)** (Text Analysis)
   - SOAP Report Generation
     - Temperature: 0.3 (balanced)
     - Max tokens: Default
   - ICD Diagnosis Classification
     - Temperature: 0.1 (very deterministic)
     - Max tokens: Default

#### Configuration
```typescript
new OpenAI({
  apiKey: apiKeyFromSSM
})

// Whisper transcription
await openai.audio.transcriptions.create({
  file: file,
  model: "whisper-1",
  language: "es",
  response_format: "json"
})

// Chat completion for SOAP/ICD
await openai.chat.completions.create({
  model: "gpt-4.1",
  messages: [{ role: "user", content: prompt + transcription }],
  temperature: 0.3 or 0.1
})
```

#### Error Handling
- Retryable: rate_limit_exceeded, timeout, network_error
- Non-retryable: service_unavailable, auth errors
- Fallback triggered on non-retryable errors

### 8.2 Anthropic Claude Integration

#### Model Used
**Claude Haiku** (`claude-haiku-4-5-20251001`)
- Used as fallback for SOAP Report and ICD Diagnosis
- Max tokens: 2000 (SOAP), 1500 (ICD)

#### Configuration
```typescript
new Anthropic({
  apiKey: apiKeyFromSSM
})

// Message creation
await anthropic.messages.create({
  model: "claude-haiku-4-5-20251001",
  max_tokens: 2000 or 1500,
  messages: [{ role: "user", content: prompt + transcription }]
})
```

### 8.3 AWS Transcribe Integration

#### Purpose
Fallback transcription service (when OpenAI Whisper fails)

#### Configuration
```typescript
await transcribeClient.send(
  new StartTranscriptionJobCommand({
    TranscriptionJobName: `vitas-transcribe-${Date.now()}`,
    LanguageCode: "es-ES",
    Media: {
      MediaFileUri: `s3://bucket/audios/file.mp3`
    },
    OutputBucketName: mediaBucket,
    OutputKey: `transcribe-output/${jobName}.json`
  })
)
```

#### Processing
- Async job submission (start and poll)
- 10-second polling interval
- Output: JSON with transcription

### 8.4 Webhook Integration (Vercel)

#### Endpoints

1. **Processing Webhook**
   - URL: Configured via SSM `/vitas/webhook-url`
   - Default: `https://your-vercel-app.vercel.app/api/processing-webhook`
   - Accepts: Success and Failure notifications

#### Request Headers
```
Content-Type: application/json
User-Agent: VITAS-Processing-Stack/1.0
X-Source: aws-lambda
X-Event-Type: processing-success|processing-failure
X-Webhook-Signature: vitas-processing-stack
x-vercel-protection-bypass: {VERCEL_BYPASS_SECRET}
```

#### Payload Structure

**Success Notification**:
```json
{
  "service": "transcription-complete|analysis",
  "metadata": {
    "appointmentId": "apt-123",
    "patientId": "pat-456",
    "doctorId": "doc-789",
    "bucket": "vitas-media-processing-...",
    "transcriptionKey": "transcripts/transcript_apt-123.json",
    "soapReportKey": "soap-reports/soap-report-apt-123.json",
    "icdDiagnosisKey": "icd-diagnoses/icd-diagnosis-apt-123.json",
    "provider": "openai|anthropic",
    "processingTime": 45000
  },
  "timestamp": "2024-11-11T10:30:00Z",
  "source": "vitas-processing-stack",
  "type": "success"
}
```

**Failure Notification**:
```json
{
  "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES|SOAP_GENERATION_FAILED_AFTER_RETRIES|ICD_DIAGNOSIS_FAILED_AFTER_RETRIES|SLA_VIOLATION",
  "service": "transcription|soap-report|icd-diagnosis|dlq-processor",
  "metadata": {
    "appointmentId": "apt-123",
    "patientId": "pat-456",
    "doctorId": "doc-789",
    "bucket": "vitas-media-processing-...",
    "key": "audios/appointment-123.mp3|transcripts/transcript_apt-123.json",
    "retryScheduled": true,
    "nextRetryIn": "2 horas",
    "slaViolation": false,
    "finalFailure": false,
    "messageAgeHours": 18
  },
  "timestamp": "2024-11-11T10:30:00Z",
  "source": "vitas-processing-stack",
  "type": "failure"
}
```

---

## 9. IAM Roles & Permissions

### Lambda Execution Roles

#### Transcription Lambda Permissions
```typescript
{
  's3:GetObject': mediaBucket + audios/
  's3:PutObject': mediaBucket + transcripts/
  's3:HeadObject': mediaBucket
  'ssm:GetParameter': /vitas/openai-api-key
  'transcribe:StartTranscriptionJob': *
  'transcribe:GetTranscriptionJob': *
  'lambda:InvokeFunction': failureNotificationFunction
  'logs:CreateLogGroup': *
  'logs:CreateLogStream': *
  'logs:PutLogEvents': *
}
```

#### SOAP Report Lambda Permissions
```typescript
{
  's3:GetObject': mediaBucket + transcripts/
  's3:PutObject': mediaBucket + soap-reports/
  's3:HeadObject': mediaBucket
  'dynamodb:PutItem': Appointments_Table_V2
  'sns:Publish': snsTopicArn
  'ssm:GetParameter': /vitas/* (all parameters)
  'lambda:InvokeFunction': failureNotificationFunction
  'logs:*': *
}
```

#### ICD Diagnosis Lambda Permissions
```typescript
{
  's3:GetObject': mediaBucket + transcripts/
  's3:HeadObject': mediaBucket
  'dynamodb:PutItem': Diagnosis_Table_V2
  'ssm:GetParameter': /vitas/* (all parameters)
  'lambda:InvokeFunction': failureNotificationFunction
  'logs:*': *
}
```

#### Workflow Trigger Lambda Permissions
```typescript
{
  'states:StartExecution': [
    transcriptionWorkflow.stateMachineArn,
    analysisWorkflow.stateMachineArn
  ]
  'logs:*': *
}
```

#### Notification Lambdas Permissions
```typescript
{
  'ssm:GetParameter': [
    /vitas/webhook-url
    /vitas/vercel-bypass-secret
  ]
  'logs:*': *
}
```

#### DLQ Processor Lambda Permissions
```typescript
{
  'sqs:ReceiveMessage': [
    transcriptionDLQ,
    soapReportDLQ,
    icdDiagnosisDLQ
  ]
  'sqs:DeleteMessage': [same DLQs]
  'sqs:GetQueueAttributes': [same DLQs]
  'states:StartExecution': [
    transcriptionWorkflow.stateMachineArn,
    analysisWorkflow.stateMachineArn
  ]
  'cloudwatch:PutMetricData': *
  'lambda:InvokeFunction': failureNotificationFunction
  'logs:*': *
}
```

### Cross-Workflow Permissions

```typescript
// Allow Analysis Workflow to be triggered by Transcription Workflow
analysisWorkflow.grantStartExecution(transcriptionWorkflow)

// Allow Step Functions to send messages to DLQs
transcription.deadLetterQueue.grantSendMessages(transcriptionWorkflow)
soapReport.deadLetterQueue.grantSendMessages(analysisWorkflow)
icdDiagnosis.deadLetterQueue.grantSendMessages(analysisWorkflow)
```

---

## 10. Monitoring & Observability

### CloudWatch Dashboard

**Name**: `VITAS-Medical-Processing-Pipeline`
**Default Interval**: 1 hour

#### Dashboard Widgets

**Row 1: Processing Overview** (4 widgets)
1. Total Executions (24h) - SingleValueWidget
2. Success Rate (24h) - SingleValueWidget
3. Average Duration - SingleValueWidget
4. Failed Executions (24h) - SingleValueWidget

**Row 2: Step Functions Performance** (2 widgets)
1. Workflow Executions Over Time - GraphWidget
   - Transcription Started/Succeeded
   - Analysis Started/Succeeded
2. Workflow Duration Trends - GraphWidget
   - Transcription Duration
   - Analysis Duration

**Row 3: Lambda Function Performance** (2 widgets)
1. Lambda Invocations - GraphWidget
   - Transcription, SOAP Report, ICD Diagnosis, Workflow Trigger
2. Lambda Duration - GraphWidget
   - Same functions

**Row 4: Error Monitoring** (2 widgets)
1. Lambda Errors - GraphWidget
   - Error count by function
2. Lambda Memory Utilization - GraphWidget
   - Memory usage by function

### CloudWatch Alarms

#### Alarm 1: Transcription High Error Rate
- **Name**: `VITAS-Transcription-High-Error-Rate`
- **Metric**: Transcription Lambda errors
- **Threshold**: 3 errors in 5 minutes
- **Evaluation**: 2 periods

#### Alarm 2: Transcription Long Duration
- **Name**: `VITAS-Transcription-Long-Duration`
- **Metric**: Transcription Lambda duration
- **Threshold**: 10 minutes (600,000 ms)
- **Evaluation**: 1 period

#### Alarm 3: Analysis Workflow Failures
- **Name**: `VITAS-Analysis-Workflow-Failures`
- **Metric**: Analysis workflow failures
- **Threshold**: 1 failure in 5 minutes
- **Evaluation**: 1 period

#### Alarm 4: DLQ Messages Detected
- **Name**: `VITAS-DLQ-Messages-Detected`
- **Metric**: Sum of all DLQ visible messages
- **Threshold**: 1+ messages
- **Evaluation**: 1 period
- **Calculation**: `transcription + soap + icd`

#### Alarm 5: SLA Compliance Risk
- **Name**: `VITAS-SLA-Compliance-Risk`
- **Metric**: `VITAS/Processing/FilesOlderThan12Hours`
- **Threshold**: 1+ files
- **Evaluation**: 1 period
- **Source**: Published by DLQ Processor

### CloudWatch Logs

#### Log Groups Created

1. **Analysis Workflow Logs**
   - Name: `/aws/stepfunctions/AnalysisWorkflow-{account}`
   - Retention: 7 days
   - Includes: Execution data, state transitions

2. **Transcription Workflow Logs**
   - Name: Implicit (standard Step Functions logging)
   - Retention: Default
   - State machine includes logs configuration

3. **Lambda Function Logs**
   - Implicit per function
   - Retention: Default (never expire)
   - Format: JSON structured logs (from StructuredLogger)

#### Log Format

All Lambda logs use structured JSON format:
```json
{
  "timestamp": "2024-11-11T10:30:00Z",
  "level": "INFO|WARN|ERROR|DEBUG",
  "service": "transcription|soap-report|icd-diagnosis|workflow-trigger|dlq-processor",
  "message": "Operation description",
  "context": {
    "appointmentId": "apt-123",
    "patientId": "pat-456",
    "doctorId": "doc-789",
    "bucket": "vitas-media-processing-...",
    "key": "audios/file.mp3"
  },
  "event": "operation_start|operation_complete|api_call|service_initialization|fallback_triggered",
  "operation": "audio_transcription|soap_generation|icd_diagnosis_generation",
  "provider": "openai|anthropic|aws_transcribe",
  "model": "gpt-4.1|claude-haiku-4-5-20251001|whisper-1",
  "metrics": {
    "duration": 45000,
    "tokens": { "prompt": 150, "completion": 200 },
    "size": 1024000
  },
  "error": {
    "type": "Error",
    "message": "Error description",
    "stack": "...",
    "code": "ENOENT"
  }
}
```

#### CloudWatch Logs Insights Queries

**Example queries for common scenarios**:

```sql
-- Find all transcription errors
fields @timestamp, message, error.message
| filter service = "transcription" and level = "ERROR"

-- Calculate average processing time
fields @timestamp, metrics.duration
| filter event = "operation_complete"
| stats avg(metrics.duration) as avg_duration

-- Find SLA violations
fields @timestamp, message, metrics
| filter service = "dlq-processor" and message like /SLA violation/

-- Identify provider usage
fields @timestamp, provider
| stats count() as invocations by provider
```

---

## 11. Infrastructure as Code Patterns

### CDK Constructs & Patterns Used

#### 1. Table Imports (Cross-Stack References)
```typescript
// Importing existing tables from main stack
const appointmentsTable = dynamodb.Table.fromTableName(
  this, 
  'AppointmentsTable', 
  'Appointments_Table_V2'
);
```

#### 2. SNS Topic Import (CloudFormation Export)
```typescript
const snsTopicArn = cdk.Fn.importValue('vitas-main-stack-NotificationTopic');
```

#### 3. Lambda Layer
```typescript
const processingLayer = new lambda.LayerVersion(this, 'ProcessingLayer', {
  code: lambda.Code.fromAsset('layers/processing-layer'),
  compatibleRuntimes: [lambda.Runtime.NODEJS_20_X],
  compatibleArchitectures: [lambda.Architecture.ARM_64]
});
```

#### 4. Lambda with DLQ
```typescript
new lambda.Function(this, 'TranscriptionFunction', {
  // ...
  deadLetterQueue: new sqs.Queue(this, 'TranscriptionDLQ', {
    queueName: `vitas-transcription-dlq-${this.account}`,
    retentionPeriod: cdk.Duration.days(14)
  })
});
```

#### 5. Step Functions State Machine
```typescript
new stepfunctions.StateMachine(this, 'TranscriptionWorkflow', {
  definition: stepfunctions.Chain.start(
    new tasks.LambdaInvoke(...)
      .addRetry({...})
      .addCatch(...)
      .next(...)
  ),
  stateMachineType: stepfunctions.StateMachineType.STANDARD,
  timeout: cdk.Duration.minutes(20),
  logs: {
    destination: new logs.LogGroup(...),
    level: stepfunctions.LogLevel.ALL
  }
});
```

#### 6. EventBridge Rule
```typescript
new events.Rule(this, 'DLQRetryRule', {
  schedule: events.Schedule.rate(cdk.Duration.hours(2))
});

rule.addTarget(new targets.LambdaFunction(dlqProcessorFunction, {
  event: events.RuleTargetInput.fromObject({...})
}));
```

#### 7. S3 Event Notifications
```typescript
mediaBucket.addEventNotification(
  s3.EventType.OBJECT_CREATED,
  new s3n.LambdaDestination(workflowTrigger),
  { prefix: 'audios/', suffix: '.mp3' }
);
```

#### 8. IAM Policy Statements
```typescript
const ssmPolicy = new iam.PolicyStatement({
  effect: iam.Effect.ALLOW,
  actions: ['ssm:GetParameter'],
  resources: [
    `arn:aws:ssm:${this.region}:${this.account}:parameter/vitas/*`
  ]
});

lambdaFunction.addToRolePolicy(ssmPolicy);
```

#### 9. CloudWatch Dashboard & Widgets
```typescript
const dashboard = new cloudwatch.Dashboard(this, 'VitasProcessingDashboard', {
  dashboardName: 'VITAS-Medical-Processing-Pipeline'
});

dashboard.addWidgets(
  new cloudwatch.SingleValueWidget({
    title: 'Total Executions (24h)',
    metrics: [workflow.metricStarted()]
  })
);
```

#### 10. CloudWatch Alarms
```typescript
new cloudwatch.Alarm(this, 'DLQMessagesAlarm', {
  metric: new cloudwatch.MathExpression({
    expression: 'transcription + soap + icd',
    usingMetrics: { /* metrics */ }
  }),
  threshold: 1,
  evaluationPeriods: 1
});
```

---

## 12. Deployment Configuration

### Stack Properties
```typescript
new VitasProcessingStackStack(app, "VitasProcessingStack", {
  env: {
    account: "197517026286",
    region: "sa-east-1"
  }
});
```

### Package.json Scripts
```json
{
  "scripts": {
    "build": "tsc",
    "watch": "tsc -w",
    "test": "jest",
    "cdk": "cdk"
  }
}
```

### Build & Deploy Commands
```bash
# Build TypeScript
npm run build

# Synthesize CloudFormation template
npm run cdk -- synth

# Deploy stack
npm run cdk -- deploy

# Watch for changes
npm run watch
```

### Dependencies
- **aws-cdk-lib**: 2.215.0
- **constructs**: 10.0.0
- **TypeScript**: ~5.6.3

### Dev Dependencies (Lambda Runtimes)
- **@anthropic-ai/sdk**: ^0.67.0
- **openai**: ^6.7.0
- **@aws-sdk/client-***: ^3.918.0 (multiple services)
- **@types/aws-lambda**: ^8.10.156
- **jest**: ^29.7.0 + ts-jest for testing

---

## 13. Data Flow Architecture

### Complete End-to-End Flow

```
1. USER UPLOADS AUDIO
   └─ S3: audios/appointment-123.mp3
      ├─ Metadata tags: doctor-id, patient-id, appointment-id
      └─ Triggers S3 ObjectCreated event

2. S3 EVENT NOTIFICATION
   └─ workflowTrigger Lambda invoked
      ├─ Extracts S3 event
      ├─ Determines workflow type: 'transcription'
      └─ Starts TranscriptionWorkflow via Step Functions

3. TRANSCRIPTION WORKFLOW (Standard, 20 min timeout)
   └─ State 1: TranscribeAudio Lambda
      ├─ Downloads audio file in chunks
      ├─ Calls OpenAI Whisper API (Spanish)
      ├─ If fails: Fall back to AWS Transcribe
      └─ Saves to S3: transcripts/transcript-apt-123.json
      
   └─ Step Functions Retry (if needed)
      └─ 3 attempts: 30s → 60s → 120s

4. ERROR: CAUGHT BY STEP FUNCTIONS
   └─ SendTranscriptionToDLQ (SQS)
      └─ Message retained 14 days
      
   └─ NotifyTranscriptionFailure webhook
      └─ Async HTTP POST to Vercel

5. CROSS-WORKFLOW EXECUTION
   └─ State 2: TriggerAnalysis (Synchronous)
      └─ Invokes AnalysisWorkflow (Express)

6. ANALYSIS WORKFLOW (Express, 5 min timeout)
   └─ ParallelAnalysis (concurrent execution)
      ├─ Branch 1: GenerateSOAP Lambda
      │  ├─ Reads transcript
      │  ├─ Calls OpenAI GPT-4.1 or Claude Haiku
      │  └─ Saves to S3: soap-reports/soap-report-apt-123.json
      │  └─ Writes to DynamoDB: Appointments_Table_V2
      │
      └─ Branch 2: GenerateICD Lambda
         ├─ Reads transcript
         ├─ Calls OpenAI GPT-4.1 or Claude Haiku
         └─ Saves diagnosis to DynamoDB: Diagnosis_Table_V2
         
   └─ CheckSource State
      ├─ If source == 'transcription_workflow'
      │  └─ Skip notification (Transcription workflow sends it)
      └─ Otherwise
         └─ NotifyAnalysisSuccess webhook

7. SUCCESS NOTIFICATION (Async)
   └─ successNotification Lambda
      └─ HTTP POST to Vercel webhook
         ```json
         {
           "type": "success",
           "service": "transcription-complete",
           "metadata": { /* s3 paths, IDs, timings */ }
         }
         ```

8. COMPLETION
   └─ Step Functions execution completes successfully
   └─ User receives webhook notification in Vercel frontend
   └─ Appointments and Diagnosis tables updated
   └─ S3 contains all artifacts

[FAULT PATH: SLA GUARANTEE]

9. ASYNC: DLQ PROCESSING (every 2 hours)
   └─ EventBridge scheduled rule
      └─ DLQ Processor Lambda
         ├─ Polls all DLQs (max 10 messages each)
         ├─ For each message:
         │  ├─ Calculate age
         │  ├─ If <12h: Retry workflow (re-execute)
         │  ├─ If 12-24h: Log warning + retry
         │  └─ If >24h: SLA violation
         │     ├─ Send final failure notification
         │     └─ Delete message
         │
         └─ Publish CloudWatch metrics
            └─ FilesOlderThan12Hours count

10. SLA VIOLATION (24 hours exceeded)
    └─ Final failure notification sent
       ```json
       {
         "type": "failure",
         "error": "SLA_VIOLATION",
         "metadata": {
           "messageAgeHours": 24+,
           "slaViolation": true,
           "finalFailure": true
         }
       }
       ```
```

### Data Persistence Strategy

```
DynamoDB (Real-time queries)
├─ Appointments_Table_V2
│  ├─ appointment_id (PK) ← from S3 metadata
│  ├─ patient_id
│  ├─ doctor_id
│  ├─ created_at
│  ├─ soap_report_key (S3 path)
│  ├─ symptoms (extracted)
│  ├─ transcription_key (S3 path)
│  └─ isFromSystem (true if from processing stack)
│
└─ Diagnosis_Table_V2
   ├─ appointment_id (PK)
   ├─ patient_id
   ├─ diagnosis (ICD codes)
   ├─ created_at
   └─ isFromSystem (true if from processing stack)

S3 (Long-term storage, version control)
├─ audios/
│  └─ appointment-ID.mp3 (input)
├─ transcripts/
│  └─ transcript-ID.json (from Whisper/Transcribe)
├─ soap-reports/
│  └─ soap-report-ID.json (from GPT-4/Claude)
├─ icd-diagnoses/
│  └─ icd-diagnosis-ID.json (from GPT-4/Claude)
└─ transcribe-output/
   └─ vitas-transcribe-TIMESTAMP.json (AWS Transcribe results)
```

---

## 14. API Contract Examples

### Webhook Payload Examples

#### Successful Transcription Completion
```json
POST /api/processing-webhook HTTP/1.1
Host: your-vercel-app.vercel.app
Content-Type: application/json
X-Event-Type: processing-success
x-vercel-protection-bypass: {secret}

{
  "service": "transcription-complete",
  "metadata": {
    "appointmentId": "apt-20241111-001",
    "patientId": "pat-20241110-002",
    "doctorId": "doc-20241101-003",
    "bucket": "vitas-media-processing-197517026286-v2",
    "transcriptionKey": "transcripts/transcript_apt-20241111-001.json",
    "provider": "openai",
    "processingTime": 48233
  },
  "timestamp": "2024-11-11T10:45:23Z",
  "source": "vitas-processing-stack",
  "type": "success"
}
```

#### Transcription Failure with Scheduled Retry
```json
POST /api/processing-webhook HTTP/1.1
Host: your-vercel-app.vercel.app
Content-Type: application/json
X-Event-Type: processing-failure
x-vercel-protection-bypass: {secret}

{
  "error": "TRANSCRIPTION_FAILED_AFTER_RETRIES",
  "service": "transcription",
  "metadata": {
    "appointmentId": "apt-20241111-001",
    "patientId": "pat-20241110-002",
    "doctorId": "doc-20241101-003",
    "bucket": "vitas-media-processing-197517026286-v2",
    "key": "audios/appointment-123.mp3",
    "retryScheduled": true,
    "nextRetryIn": "2 horas"
  },
  "timestamp": "2024-11-11T10:45:23Z",
  "source": "vitas-processing-stack",
  "type": "failure"
}
```

#### SLA Violation (24-hour threshold exceeded)
```json
POST /api/processing-webhook HTTP/1.1
Host: your-vercel-app.vercel.app
Content-Type: application/json
X-Event-Type: processing-failure
x-vercel-protection-bypass: {secret}

{
  "error": "SLA_VIOLATION",
  "message": "Processing failed after 24 hours - SLA violation",
  "service": "dlq-processor",
  "metadata": {
    "messageAgeHours": 24,
    "slaViolation": true,
    "finalFailure": true,
    "originalEvent": {
      "bucket": "vitas-media-processing-197517026286-v2",
      "key": "audios/appointment-123.mp3"
    }
  },
  "timestamp": "2024-11-11T10:45:23Z",
  "source": "vitas-processing-stack",
  "type": "failure"
}
```

---

## 15. Configuration & Best Practices

### Pre-Deployment Checklist

1. **SSM Parameters Setup**
   ```bash
   # After deployment, manually update parameters:
   aws ssm put-parameter \
     --name /vitas/openai-api-key \
     --value "sk-..." \
     --type SecureString \
     --overwrite
   
   aws ssm put-parameter \
     --name /vitas/anthropic-api-key \
     --value "sk-ant-..." \
     --type SecureString \
     --overwrite
   
   aws ssm put-parameter \
     --name /vitas/vercel-bypass-secret \
     --value "..." \
     --type SecureString \
     --overwrite
   
   aws ssm put-parameter \
     --name /vitas/webhook-url \
     --value "https://your-vercel-app.vercel.app/api/processing-webhook" \
     --type String \
     --overwrite
   ```

2. **IAM Role Verification**
   - Ensure Lambda execution roles have proper permissions
   - Verify cross-account access if applicable

3. **Webhook Endpoint**
   - Test webhook endpoint before deployment
   - Ensure Vercel protection rules allow processing stack IP ranges

4. **S3 Bucket Policy**
   - Verify CORS settings if frontend uploads directly
   - Configure lifecycle rules as needed

5. **DynamoDB Tables**
   - Ensure tables exist and have correct schema
   - Verify cross-stack import names match

### Operational Best Practices

1. **Monitoring**
   - Set up SNS topic for CloudWatch alarm notifications
   - Review dashboard daily for anomalies
   - Monitor DLQ depth regularly

2. **Cost Optimization**
   - Use Lambda reserved concurrency to prevent runaway costs
   - Monitor API call costs (OpenAI, Anthropic)
   - S3 lifecycle transitions to IA after 30 days (configured)

3. **Error Handling**
   - Review CloudWatch Logs regularly for error patterns
   - Adjust retry parameters based on failure analysis
   - Consider implementing circuit breaker pattern for external APIs

4. **Security**
   - Rotate API keys periodically
   - Use SecureString for sensitive parameters
   - Enable VPC endpoints for private API access if needed
   - Implement API rate limiting

5. **Disaster Recovery**
   - Backup DynamoDB tables regularly
   - Test failover procedures
   - Keep S3 versioning enabled for audit trails

---

## Summary Table

| Component | Type | Count | Key Details |
|-----------|------|-------|-------------|
| S3 Buckets | Storage | 1 | Media processing bucket with lifecycle |
| Lambda Functions | Compute | 7 | Multi-purpose pipeline processors |
| Step Functions | Orchestration | 2 | Transcription (Standard) + Analysis (Express) |
| SQS Queues | Messaging | 3 | One DLQ per processor Lambda |
| DynamoDB Tables | Database | 2 | Appointments + Diagnosis (imported) |
| EventBridge Rules | Scheduling | 1 | DLQ retry every 2 hours |
| SSM Parameters | Configuration | 5 | API keys + prompts + webhook |
| CloudWatch Alarms | Monitoring | 5 | Error rates, duration, DLQ depth, SLA |
| Lambda Layers | Dependencies | 1 | OpenAI + Anthropic + AWS SDK |
| IAM Roles | Security | 7+ | Custom policies per function |

---

## Key Technical Highlights

1. **Dual-Provider Strategy**: OpenAI primary with Anthropic fallback
2. **24-Hour SLA Guarantee**: Scheduled DLQ processing every 2 hours
3. **Structured Logging**: JSON format for CloudWatch Logs Insights
4. **Parallel Processing**: SOAP + ICD analysis runs concurrently
5. **Cross-Stack Integration**: Imports tables and SNS from main stack
6. **Event-Driven Architecture**: S3 events trigger workflows automatically
7. **Error Recovery**: Comprehensive retry + DLQ + webhook notification system
8. **Multi-Language Support**: Spanish language configuration for Whisper/Transcribe
9. **Cost-Optimized**: ARM_64 Graviton processors, Express state machines for fast paths
10. **Production-Ready**: Comprehensive monitoring, alarms, and observability

