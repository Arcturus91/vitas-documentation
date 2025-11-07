# System Architecture Overview

## Table of Contents
- [High-Level Architecture](#high-level-architecture)
- [Technology Stack](#technology-stack)
- [AWS Infrastructure Stacks](#aws-infrastructure-stacks)
- [Frontend Applications](#frontend-applications)
- [LangGraph AI System](#langgraph-ai-system)
- [Data Flow](#data-flow)
- [Integration Points](#integration-points)
- [Service Inventory](#service-inventory)

## High-Level Architecture

The VITAS Clinic platform consists of **4 AWS CDK stacks**, **2 Next.js frontend applications**, and **1 LangGraph AI service**, all working together to provide a complete healthcare automation solution.

```
┌────────────────────────────────────────────────────────────────────────┐
│                         VITAS CLINIC ECOSYSTEM                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐       │
│  │   Doctor Web    │  │  Patient Web    │  │    WhatsApp      │       │
│  │   Dashboard     │  │     Portal      │  │    Business      │       │
│  │   (Next.js)     │  │   (Next.js)     │  │    Messages      │       │
│  └────────┬────────┘  └────────┬────────┘  └────────┬─────────┘       │
│           │                     │                     │                  │
│           │                     │                     │                  │
│           └─────────────────────┴─────────────────────┘                  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │                    API GATEWAY LAYER                         │      │
│  ├──────────────────────────────────────────────────────────────┤      │
│  │  • Main API (/auth/*, /doctors/*, /patients/*)              │      │
│  │  • OTP API (/otp/*, /chatbot/*)                             │      │
│  │  • Webhooks (/processing-webhook, /confirm-pay)             │      │
│  └──────────────────┬───────────────────────────────────────────┘      │
│                     │                                                    │
│         ┌───────────┴────────────┬──────────────────┐                  │
│         ▼                        ▼                   ▼                  │
│  ┌──────────────┐      ┌──────────────────┐  ┌─────────────────┐     │
│  │ VITAS-MAIN   │      │  VITAS-AUTH-OTP  │  │ VITAS-PROCESS   │     │
│  │   STACK      │      │      STACK       │  │    STACK        │     │
│  ├──────────────┤      ├──────────────────┤  ├─────────────────┤     │
│  │ • Auth       │      │ • OTP Verify     │  │ • Transcription │     │
│  │ • Doctors    │◄────►│ • Chat Gateway   │  │ • SOAP Reports  │     │
│  │ • Patients   │      │ • WhatsApp Int.  │  │ • ICD Coding    │     │
│  │ • Profile    │      │ • Payment Conf.  │  │ • Step Funcs    │     │
│  │   Pictures   │      └────────┬─────────┘  │ • DLQ Handler   │     │
│  └──────┬───────┘               │            └────────┬────────┘     │
│         │                       │                     │                │
│         │                       ▼                     │                │
│         │              ┌──────────────────┐          │                │
│         │              │  LANGGRAPH AI    │          │                │
│         │              │     CLOUD        │          │                │
│         │              ├──────────────────┤          │                │
│         │              │ • Doctor Agent   │          │                │
│         │              │ • Patient Agent  │          │                │
│         │              │ • ReAct Pattern  │          │                │
│         │              │ • GPT-4o-mini    │          │                │
│         │              └──────────────────┘          │                │
│         │                                             │                │
│         ▼                                             ▼                │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │                  AWS FOUNDATIONAL SERVICES                │        │
│  ├──────────────────────────────────────────────────────────┤        │
│  │  • DynamoDB (12 tables)   • S3 (3 buckets)               │        │
│  │  • CloudFront CDN         • Systems Manager (secrets)     │        │
│  │  • CloudWatch Monitoring  • EventBridge Scheduling        │        │
│  │  • Step Functions         • SQS Dead Letter Queues        │        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │               EXTERNAL INTEGRATIONS                       │        │
│  ├──────────────────────────────────────────────────────────┤        │
│  │  • OpenAI (Whisper, GPT-4o)  • Anthropic (Claude)        │        │
│  │  • Twilio (WhatsApp API)      • MercadoPago (Payments)    │        │
│  │  • Deepgram (Speech-to-Text) • Upstash Redis (Rate Limit)│        │
│  └──────────────────────────────────────────────────────────┘        │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

## Technology Stack

### Frontend Technologies
| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Framework** | Next.js | 14.2.7 | React framework with SSR and API routes |
| **UI Library** | React | 18.x | Component-based UI |
| **Language** | TypeScript | 5.x | Type-safe development |
| **Styling** | Tailwind CSS | 3.4.1 | Utility-first CSS |
| **Component Library** | Material-UI | 7.1.0 | Pre-built UI components |
| **State Management** | React Context | Built-in | Global state (user, notifications) |
| **API Client** | Axios + fetch | - | HTTP requests |
| **Real-time** | Server-Sent Events | - | Streaming AI responses |
| **Notifications** | Notistack | 3.0.2 | Toast notifications |

### Backend Technologies (AWS)
| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Runtime** | Node.js | 20.x | Lambda execution environment |
| **Architecture** | ARM64 | Graviton2 | Cost-optimized Lambda |
| **IaC** | AWS CDK | 2.215.0 | Infrastructure as Code |
| **Language** | TypeScript | 5.x | CDK stack definitions |
| **Authentication** | JWT + jose | 5.9.3 | Token-based auth |
| **Validation** | Zod | 3.25.76 | Schema validation |
| **Database SDK** | AWS SDK v3 | 3.670.0 | DynamoDB operations |
| **Storage SDK** | AWS SDK v3 | 3.670.0 | S3 operations |

### AI & LangGraph Technologies
| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Framework** | LangGraph | 0.6.0 | Stateful AI agent orchestration |
| **LLM Library** | LangChain | 0.2.14+ | LLM abstractions |
| **Primary LLM** | OpenAI GPT-4o-mini | - | Conversational AI |
| **Fallback LLM** | Anthropic Claude 3 | - | Backup AI provider |
| **Language** | Python | 3.11+ | LangGraph runtime |
| **Pattern** | ReAct | - | Reasoning + Acting agent pattern |
| **Deployment** | LangGraph Cloud | - | Managed hosting |

### External Services
| Service | Provider | Purpose |
|---------|----------|---------|
| **Messaging** | Twilio WhatsApp Business API | Send/receive WhatsApp messages |
| **Payments** | MercadoPago | Payment processing |
| **Transcription** | OpenAI Whisper API | Audio-to-text conversion |
| **Transcription Fallback** | AWS Transcribe | Backup transcription |
| **AI Reports** | OpenAI GPT-4o | SOAP/ICD report generation |
| **AI Fallback** | Anthropic Claude 3 Sonnet | Backup AI provider |
| **Real-time STT** | Deepgram | Live audio transcription |
| **Rate Limiting** | Upstash Redis | Distributed rate limiting |
| **Monitoring** | LangSmith | AI agent performance tracking |

## AWS Infrastructure Stacks

### 1. VitasMainStack

**Purpose:** Core authentication, user management, and profile storage

**Location:** `vitas-main-stack/lib/vitas-main-stack.ts`

**Key Resources:**

#### Lambda Functions (1)
- **VitasAuthFunction**
  - Runtime: Node.js 20.x ARM64
  - Memory: 256 MB
  - Timeout: 30 seconds
  - Role: Doctor authentication, profile management, S3 presigned URL generation

#### DynamoDB Tables (7)
| Table Name | Partition Key | Sort Key | GSI | Purpose |
|------------|---------------|----------|-----|---------|
| `Patients_Table_V2` | patient_id | phone_number | phone_number-index | Patient profiles |
| `PatientDoctors_Table_V2` | doctor_id | patient_id | patient_id-index | Doctor-patient relationships |
| `Doctors_Table_V2` | doctor_id | - | email-index, active_for_ai_booking-index | Doctor profiles |
| `Appointments_Table_V2` | patient_id | appointment_id | appointment_id-index, doctor_id-created_at-index | Appointments with SOAP/ICD |
| `Diagnosis_Table_V2` | appointment_id | patient_id | - | ICD-10 diagnoses |
| `Vitas_Appointment` | doctor_id | appt | appointment_id-index | Available appointment slots (TTL) |
| `Vitas_Schedule_Config` | doctor_id | config | - | Doctor schedule configuration |

#### S3 Buckets (2)
- **vitas-profile-pictures-v2** - Doctor/patient profile pictures
- **vitas-profile-pictures-v2-cloudfront-logs** - CloudFront access logs

#### CloudFront Distribution
- Origin: S3 bucket with Origin Access Control (OAC)
- Domain: `d2k0hzw5g3avku.cloudfront.net`
- Protocol: HTTPS redirect
- Caching: Optimized for images

#### API Gateway
- **Name:** Vitas Auth API V2
- **Endpoint:** `https://q36l2q61i5.execute-api.sa-east-1.amazonaws.com/prod/`
- **Rate Limiting:** 100 req/sec, 200 burst, 10k/day quota
- **Routes:** /auth/signin, /auth/signup, /auth/profile, /auth/upload-url, /health

#### SSM Parameters
- `/vitas/auth/jwt-secret` - JWT signing key (SecureString)

---

### 2. VitasProcessingStack

**Purpose:** Medical audio processing with AI-powered transcription and analysis

**Location:** `vitas-processing-stack/lib/vitas-processing-stack-stack.ts`

**Key Resources:**

#### Lambda Functions (7)
| Function | Memory | Timeout | Purpose | DLQ |
|----------|--------|---------|---------|-----|
| **Transcription** | 1024 MB | 15 min | Audio → Text (OpenAI/AWS) | Yes |
| **SOAP Report** | 512 MB | 5 min | Generate clinical notes | Yes |
| **ICD Diagnosis** | 512 MB | 5 min | Medical coding (ICD-10) | Yes |
| **Workflow Trigger** | 256 MB | 1 min | S3 event → Step Functions | No |
| **Failure Notification** | 256 MB | 1 min | Send error webhooks | No |
| **Success Notification** | 256 MB | 1 min | Send success webhooks | No |
| **DLQ Processor** | 512 MB | 15 min | Retry failed jobs (2h schedule) | No |

#### Step Functions Workflows (2)
| Workflow | Type | Timeout | Purpose |
|----------|------|---------|---------|
| **Transcription Workflow** | Standard | 20 min | Audio transcription with retries |
| **Analysis Workflow** | Express | 5 min | Parallel SOAP + ICD generation |

**Workflow Features:**
- 3 automatic retries with exponential backoff (30s, 60s, 120s)
- Intelligent fallback (OpenAI → Anthropic)
- Manual DLQ population on failure
- Failure notification webhook

#### S3 Bucket
- **vitas-media-processing-{account}-v2**
  - Folders: `audios/`, `transcripts/`, `soap-reports/`, `icd-diagnosis/`
  - Event triggers: ObjectCreated → Workflow Trigger Lambda

#### SQS Dead Letter Queues (3)
- `vitas-transcription-dlq-{account}` - Retention: 14 days
- `vitas-soap-report-dlq-{account}` - Retention: 14 days
- `vitas-icd-diagnosis-dlq-{account}` - Retention: 14 days

#### EventBridge Rule
- **Schedule:** Every 2 hours (cron)
- **Target:** DLQ Processor Lambda
- **Purpose:** Automatic retry for failed jobs (24-hour SLA)

#### CloudWatch
- **Dashboard:** VITAS-Medical-Processing-Pipeline
- **Alarms:** High error rates, long durations, DLQ depth, SLA violations

#### SSM Parameters
- `/vitas/openai-api-key` - OpenAI API key
- `/vitas/anthropic-api-key` - Anthropic API key
- `/vitas/webhook-url` - Frontend notification webhook
- `/vitas/vercel-bypass-secret` - Vercel Protection bypass
- `/vitas/prompts/soap-report` - SOAP generation prompt
- `/vitas/prompts/icd-diagnosis` - ICD classification prompt

---

### 3. VitasAuthOtpStack (VitasChatbotStack)

**Purpose:** OTP authentication and WhatsApp chatbot gateway

**Location:** `vitas-auth-otp/lib/vitas-auth-otp-stack.ts`

**Key Resources:**

#### Lambda Functions (9)
| Function | Endpoint | Purpose | Token Expiry |
|----------|----------|---------|--------------|
| **send-otp** | POST /otp/send | Generate payment OTP | - |
| **verify-otp** | POST /otp/verify | Verify payment OTP | 24 hours |
| **chatbot-send-otp** | POST /chatbot/send-otp | Generate chatbot OTP | - |
| **chatbot-verify-otp** | POST /chatbot/verify-otp | Verify chatbot OTP | 2 hours |
| **get-appointment** | POST /appointments/get | Fetch appointment details | JWT required |
| **chat-integration** | POST /chat-webhook | Twilio webhook handler | - |
| **agent-webhook** | POST /agent-webhook | LangGraph callback | - |
| **confirm-pay** | POST /confirm-pay | MercadoPago webhook | - |
| **paylink** | - | Generate short URL tokens | - |

#### DynamoDB Tables (3 new)
| Table | Partition Key | TTL | Purpose |
|-------|---------------|-----|---------|
| `vitas-otp` | phoneNumber | expiresAt (5 min) | Temporary OTP storage |
| `wa_sessions` | pk (phone) | expiresAt (15 min idle) | WhatsApp session management |
| `vitas-paylinks` | pk | expiresAt | Payment link tokens |

**External Table References:**
- Patients_Table_V2 (read/write)
- PatientDoctors_Table_V2 (write)
- Vitas_Appointment (read/write via GSI)
- Doctors_Table_V2 (read-only)

#### API Gateway
- **Endpoints:** 9 routes for OTP, chat, payment
- **Authentication:** JWT validation, Twilio signature verification
- **Rate Limiting:** OTP limited to 3 attempts per phone

#### SSM Parameters
- `/vitas-clinic/twilio-config` - Twilio credentials
- `/vitas-clinic/jwt-secret` - JWT signing key

#### External Integrations
- **Twilio:** WhatsApp message send/receive
- **MercadoPago:** Payment webhook processing
- **LangGraph:** AI agent API calls

---

### 4. VitasChatbot (LangGraph Service)

**Purpose:** AI conversational assistant with role-based agents

**Location:** `vitas-chatbot/src/react_agent/`

**Architecture:**

#### StateGraph Components
```python
StateGraph
├── START → role_condition → doctor_agent OR load_patient
├── doctor_agent → END
├── load_patient → check_information → patient_agent OR upsert_patient
└── patient_agent → END
```

#### Agents (2)
| Agent | Role | Tools | Purpose |
|-------|------|-------|---------|
| **Doctor Agent** | doctor | get_patients, get_consultations, search_medical_info | Access patient records, schedule |
| **Patient Agent** | patient | get_doctors, find_slots, schedule_appointment, update_profile | Book appointments, manage profile |

#### LLM Configuration
- **Primary:** OpenAI GPT-4o-mini (temperature=0)
- **Mode:** Streaming enabled
- **Context:** State-based with message history

#### Tool Implementations
**Patient Tools:**
- `get_available_doctors(specialty)` - Search doctors
- `find_free_slots_for_doctor(doctor_id, date)` - Check availability
- `schedule_patient_appointment(...)` - Book appointment
- `upsert_patient_profile(...)` - Update patient info

**Doctor Tools:**
- `get_patients_by_doctor()` - List patients
- `get_scheduled_consultations_by_doctor(...)` - View schedule
- `get_consultation_details(...)` - Access medical records
- `search_medical_information(query)` - Medical knowledge lookup

#### Database Integration
**Python DynamoDB Client:** `src/react_agent/aws/dynamo_client.py`

Tables accessed:
- AWS_DOCTORS_TABLE (Doctors_Table_V2)
- AWS_PATIENTSDOCTORS_TABLE (PatientDoctors_Table_V2)
- AWS_PATIENTS_DETAILS_TABLE (Patients_Table_V2)
- AWS_VIRTUAL_APPOINTMENTS_TABLE (Vitas_Appointment)
- AWS_SCHEDULE_APPOINTMENTS_TABLE (Vitas_Schedule_Config)

#### Deployment
- **Platform:** LangGraph Cloud
- **Configuration:** `langgraph.json`
- **Auth:** Custom JWT validation (`src/security/auth.py`)
- **Monitoring:** LangSmith integration

---

## Frontend Applications

### 1. Doctor Dashboard (vitas-client)

**Purpose:** Doctor-facing web application for consultations and patient management

**Location:** `vitas-client/src/app/`

**Key Features:**
- Voice recording and audio upload
- Real-time transcription (Deepgram)
- SOAP report viewing and editing
- ICD-10 diagnosis management
- Patient consultation history
- AI chatbot sidebar
- Processing notifications (polling every 10 seconds)

**Key Components:**
| Component | Location | Purpose |
|-----------|----------|---------|
| **Chatbot** | `medinotas/components/Chatbot.tsx` | LangGraph AI assistant |
| **Voice Recorder** | `medinotas/voice-record/` | Audio capture and upload |
| **Notifications Hook** | `hooks/useProcessingNotifications.ts` | Polling for async processing updates |
| **User Context** | `contexts/UserContext.tsx` | Global user state |

**API Integration:**
- `/api/chat/[...path]` - Proxy to LangGraph
- `/api/processing-webhook` - Notification webhook (POST/GET)
- `/api/audio-to-s3-presigned-url` - S3 upload
- All other `/api/*` routes proxy to AWS API Gateway

**Deployment:**
- **Platform:** Vercel
- **Domain:** (configured in Vercel)
- **Environment:** Edge functions

### 2. Patient Portal (vitas-patients-front)

**Purpose:** Patient-facing web application for appointment booking

**Location:** `vitas-patients-front/src/`

**Key Features:**
- OTP authentication
- Chatbot for appointment booking
- Payment via MercadoPago
- Appointment history

**Deployment:**
- **Platform:** Vercel
- **Environment:** Edge functions

---

## Data Flow

### 1. Audio Processing Flow

```
Doctor Records Audio
    ↓
[Frontend] Upload to S3 (presigned URL)
    ↓
[S3] audios/doctor-{id}/patient-{id}/appointment-{id}.mp3
    ↓
[S3 Event] → Workflow Trigger Lambda
    ↓
[Step Functions] Start Transcription Workflow (Standard)
    ↓
[Transcription Lambda]
    ├─ Try OpenAI Whisper (primary)
    │  └─ Retry 3x (30s, 60s, 120s backoff)
    ├─ Fallback to AWS Transcribe
    └─ On failure → DLQ + Failure Notification
    ↓
[S3] transcripts/doctor-{id}/patient-{id}/appointment-{id}.json
    ↓
[Step Functions] Start Analysis Workflow (Express)
    ↓
    ┌─────────────────────┴─────────────────────┐
    ▼                                             ▼
[SOAP Lambda]                             [ICD Lambda]
├─ Try OpenAI GPT-4o                      ├─ Try OpenAI GPT-4o
├─ Fallback Anthropic Claude              ├─ Fallback Anthropic Claude
└─ Write to Appointments_Table_V2         └─ Write to Diagnosis_Table_V2
    ↓                                             ↓
    └─────────────────────┬─────────────────────┘
                          ↓
              [Success Notification Lambda]
                          ↓
              POST to /api/processing-webhook
                          ↓
              [Frontend Polling] → Show snackbar
```

### 2. WhatsApp Chatbot Flow

```
Patient sends WhatsApp message
    ↓
[Twilio] Webhook to /chat-webhook
    ↓
[chat-integration Lambda]
    ├─ Validate Twilio signature
    ├─ Parse phone number & message
    ├─ Load/create wa_session (with JWT)
    ├─ Auto-create patient in Patients_Table_V2
    └─ Renew session (15min idle, 1h absolute)
    ↓
[LangGraph Cloud] POST /threads/{threadId}/runs
    ├─ Headers: Authorization (JWT), assistant_id, webhook URL
    ├─ Payload: {messages: [message], source: "patient"}
    └─ Async processing
    ↓
[LangGraph] role_condition
    ├─ If role=doctor → doctor_agent
    └─ If role=patient → load_patient → patient_agent
    ↓
[Agent] Execute tools (DynamoDB queries)
    ├─ get_available_doctors()
    ├─ find_free_slots_for_doctor()
    └─ schedule_patient_appointment()
    ↓
[LangGraph] Generate AI response
    ↓
[LangGraph] Callback to /agent-webhook
    ↓
[agent-webhook Lambda]
    ├─ Extract AI message
    ├─ Convert markdown to WhatsApp format
    └─ Send via Twilio WhatsApp API
    ↓
Patient receives WhatsApp reply
```

### 3. Appointment Booking Flow (Web)

```
Patient visits web portal
    ↓
[Frontend] POST /otp/send (phone number)
    ↓
[send-otp Lambda]
    ├─ Generate 6-digit OTP
    ├─ Store in vitas-otp (TTL: 5 min)
    ├─ Rate limit: 3 attempts
    └─ Send via Twilio WhatsApp
    ↓
Patient enters OTP
    ↓
[Frontend] POST /otp/verify (phone, OTP)
    ↓
[verify-otp Lambda]
    ├─ Validate OTP
    ├─ Create/load patient in Patients_Table_V2
    ├─ Generate JWT (24h expiry)
    └─ Return JWT + patient data
    ↓
[Frontend] Select doctor, date, time
    ↓
[Frontend] POST /appointments/book (JWT in header)
    ↓
Create appointment in Vitas_Appointment (status=pending, TTL=15min)
    ↓
[Frontend] Redirect to MercadoPago
    ↓
Patient completes payment
    ↓
[MercadoPago] Webhook to /confirm-pay
    ↓
[confirm-pay Lambda]
    ├─ Validate payment via MercadoPago API
    ├─ Update appointment (status=confirmed, paid=true)
    ├─ Remove TTL (persist appointment)
    ├─ Create PatientDoctors_Table_V2 entry
    └─ Send WhatsApp confirmation
    ↓
Patient receives confirmation
```

## Integration Points

### 1. Cross-Stack Data Sharing

| Source Stack | Target Stack | Mechanism | Resources |
|--------------|--------------|-----------|-----------|
| vitas-main-stack | vitas-processing-stack | CloudFormation Exports | DynamoDB table names, SNS topic ARN |
| vitas-main-stack | vitas-auth-otp | Direct table name references | All 7 DynamoDB tables |
| vitas-auth-otp | vitas-chatbot | HTTP API + JWT | LangGraph Cloud endpoint |
| vitas-processing-stack | vitas-client | HTTP Webhook | /api/processing-webhook |

### 2. External Service Integration

| Service | Integration Type | Stacks Using It |
|---------|-----------------|-----------------|
| **OpenAI** | REST API | vitas-processing-stack, vitas-chatbot |
| **Anthropic** | REST API | vitas-processing-stack |
| **Twilio** | Webhook (inbound), REST API (outbound) | vitas-auth-otp |
| **MercadoPago** | Webhook | vitas-auth-otp |
| **Deepgram** | WebSocket | vitas-client (frontend) |
| **Upstash Redis** | REST API | vitas-client (API routes) |
| **LangGraph Cloud** | HTTP API + SSE | vitas-client (frontend), vitas-auth-otp (backend) |

### 3. Authentication Flow Integration

```
[Frontend] Login/OTP
    ↓
[vitas-auth-otp] Generate JWT
    ↓
[Frontend] Store JWT in HTTP-only cookie
    ↓
[Frontend] API calls include JWT
    ↓
[vitas-main-stack] Validate JWT in API Gateway/Lambda
    OR
[vitas-auth-otp] Validate JWT in Lambda
    OR
[vitas-chatbot] Validate JWT in LangGraph auth.py
```

## Service Inventory

### AWS Services Used

| Service | Count | Purpose |
|---------|-------|---------|
| **Lambda** | 17 | Serverless compute |
| **DynamoDB** | 12 tables | NoSQL database |
| **S3** | 3 buckets | Object storage |
| **API Gateway** | 2 | REST API management |
| **Step Functions** | 2 workflows | Orchestration |
| **EventBridge** | 1 rule | Scheduled triggers |
| **SQS** | 3 queues | Dead letter queues |
| **CloudFront** | 1 distribution | CDN |
| **CloudWatch** | Multiple | Logs, metrics, alarms, dashboard |
| **Systems Manager** | 8+ parameters | Secrets storage |
| **IAM** | Multiple | Access control |

### Lambda Functions Summary

| Stack | Lambda Count | Total Memory | Total Timeout |
|-------|--------------|--------------|---------------|
| vitas-main-stack | 1 | 256 MB | 30s |
| vitas-processing-stack | 7 | 4.5 GB | 53 min total |
| vitas-auth-otp | 9 | ~2.3 GB | 4.5 min total |
| **Total** | **17** | **~7 GB** | **~58 min** |

### DynamoDB Tables Summary

| Stack | Table Count | Billing Mode | Features |
|-------|-------------|--------------|----------|
| vitas-main-stack | 7 | PAY_PER_REQUEST | 5 GSI indexes, Point-in-time recovery |
| vitas-processing-stack | 0 (imports from main) | - | - |
| vitas-auth-otp | 3 new + 4 imported | PAY_PER_REQUEST | TTL enabled on all 3 new tables |
| **Total** | **12 unique** | **PAY_PER_REQUEST** | **5 GSI, 3 TTL** |

### S3 Buckets Summary

| Bucket | Stack | Size Estimate | Purpose |
|--------|-------|---------------|---------|
| vitas-profile-pictures-v2 | vitas-main-stack | ~10 GB | Profile pictures |
| vitas-profile-pictures-v2-cloudfront-logs | vitas-main-stack | ~1 GB | Access logs |
| vitas-media-processing-{account}-v2 | vitas-processing-stack | ~500 GB | Audio, transcripts, reports |

## Architecture Patterns

### 1. Microservices Architecture
Each AWS stack represents a bounded context with clear responsibilities and minimal coupling.

### 2. Event-Driven Processing
S3 events trigger workflows, webhooks deliver notifications, EventBridge schedules retries.

### 3. CQRS (Command Query Responsibility Segregation)
DynamoDB GSI indexes optimize read patterns separately from write patterns.

### 4. Retry with Exponential Backoff
Step Functions implement 3 retries with increasing delays (30s, 60s, 120s).

### 5. Circuit Breaker (Provider Fallback)
Primary providers (OpenAI) automatically fall back to secondary providers (Anthropic, AWS).

### 6. Dead Letter Queue (DLQ)
Failed messages are captured for later retry, ensuring no data loss.

### 7. Asynchronous Messaging
Webhooks decouple frontend from backend processing, enabling long-running workflows.

### 8. API Gateway Pattern
Centralized API management with authentication, rate limiting, and request validation.

### 9. Repository Pattern
DynamoDB client abstractions in LangGraph (`dynamo_client.py`) separate data access from business logic.

### 10. Agent Pattern (ReAct)
LangGraph agents use Reasoning (think) + Acting (tool use) pattern for conversational AI.

## Performance Characteristics

### Latency
- **API Calls:** <500ms (p95)
- **Transcription:** ~3 minutes for 15-minute audio
- **SOAP Report:** ~30 seconds
- **ICD Coding:** ~20 seconds
- **Chatbot Response:** <2 seconds (streaming)

### Throughput
- **API Gateway:** 10,000 requests/second (AWS limit)
- **Lambda Concurrent Executions:** 1,000 (default quota)
- **DynamoDB:** Auto-scales to handle demand
- **S3:** Unlimited throughput

### Scalability
- **Horizontal:** Automatic Lambda scaling
- **Vertical:** Configurable memory (256 MB - 10 GB per Lambda)
- **Database:** DynamoDB auto-scaling with PAY_PER_REQUEST

## Resilience & Reliability

### High Availability
- **Multi-AZ:** All AWS services deployed across multiple availability zones
- **Regional:** Single region (sa-east-1) with future multi-region roadmap

### Disaster Recovery
- **RPO (Recovery Point Objective):** <1 hour (DynamoDB point-in-time recovery)
- **RTO (Recovery Time Objective):** <4 hours (manual stack recreation)

### Fault Tolerance
- **Automatic Retries:** Step Functions retry failed tasks 3 times
- **Dead Letter Queues:** Capture failed messages for up to 14 days
- **DLQ Processor:** Automatic retry every 2 hours for 24 hours
- **Provider Fallback:** Multiple AI providers prevent single point of failure

### Monitoring & Alerting
- **CloudWatch Alarms:** 5+ alarms for critical metrics
- **Dashboard:** Real-time system health visualization
- **Log Aggregation:** Centralized CloudWatch Logs
- **Error Tracking:** Automated error notifications

---

**Next Steps:**
- [Deployment Guide](02-DEPLOYMENT_GUIDE.md) - Learn how to deploy all stacks
- [Frontend-Backend Integration](03-FRONTEND_BACKEND_INTEGRATION.md) - Understand API flows
- [Database Schema](07-DATABASE_SCHEMA.md) - Explore data models

**Last Updated:** 2025-11-06
