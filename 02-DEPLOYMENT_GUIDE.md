# Deployment Guide

## Table of Contents
- [Prerequisites](#prerequisites)
- [Deployment Order](#deployment-order)
- [Stack Deployment Instructions](#stack-deployment-instructions)
- [Frontend Deployment](#frontend-deployment)
- [LangGraph Deployment](#langgraph-deployment)
- [Post-Deployment Verification](#post-deployment-verification)
- [Environment Variables](#environment-variables)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Required Software
- **Node.js:** v20.x or higher
- **npm:** v10.x or higher
- **AWS CLI:** v2.x configured with credentials
- **AWS CDK:** v2.215.0 or higher
- **Python:** 3.11+ (for LangGraph)
- **LangGraph CLI:** Latest version

### AWS Account Setup
1. **AWS Account:** Active AWS account with appropriate permissions
2. **AWS Region:** sa-east-1 (São Paulo, Brazil)
3. **AWS CLI Profile:** Configured profile (example: `vitas-sso`)
4. **IAM Permissions:** Administrator access or permissions for:
   - Lambda, DynamoDB, S3, API Gateway, CloudFront
   - Step Functions, EventBridge, SQS, CloudWatch
   - Systems Manager Parameter Store, IAM

### External Service Accounts
- **OpenAI:** API key for Whisper and GPT-4o
- **Anthropic:** API key for Claude 3 Sonnet
- **Twilio:** Account SID, Auth Token, WhatsApp Business API access
- **MercadoPago:** Access token for payment processing
- **Upstash Redis:** API credentials for rate limiting
- **Deepgram:** API key for real-time transcription (optional)
- **LangSmith:** API key for LangGraph monitoring (optional)

## Deployment Order

**IMPORTANT:** Stacks must be deployed in this specific order due to dependencies.

```
1. vitas-main-stack         (Core infrastructure)
   ↓
2. vitas-processing-stack   (Depends on main-stack tables, SNS)
   ↓
3. vitas-auth-otp          (Depends on main-stack tables)
   ↓
4. vitas-chatbot           (Depends on auth-otp JWT validation)
   ↓
5. vitas-client            (Depends on all backend APIs)
   ↓
6. vitas-patients-front    (Depends on auth-otp and chatbot)
```

### Dependency Graph
```
vitas-main-stack
├── Exports: DynamoDB tables, SNS topic
│
├─→ vitas-processing-stack (imports tables, SNS)
│   └── Exports: None
│
└─→ vitas-auth-otp (imports all 7 main tables)
    ├── Exports: API Gateway URL
    │
    └─→ vitas-chatbot (calls auth-otp for JWT)
        └── Exports: LangGraph URL
            │
            ├─→ vitas-client (calls all APIs)
            │
            └─→ vitas-patients-front (calls chatbot API)
```

## Stack Deployment Instructions

### 1. Deploy vitas-main-stack

**Purpose:** Core authentication, user management, and profile storage

**Location:** `vitas-main-stack/`

#### Step 1.1: Install Dependencies
```bash
cd vitas-main-stack
npm install
```

#### Step 1.2: Configure Environment Variables
Create `.env` file in `vitas-main-stack/`:
```bash
# JWT Secret (generate secure random string)
JWT_SECRET=your-secure-jwt-secret-minimum-32-characters

# S3 Bucket Name (must be globally unique)
PROFILE_PICTURE_BUCKET=vitas-profile-pictures-v2

# SNS Topic Name
SNS_TOPIC_NAME=VitasNotificationsV2

# AWS Region
AWS_REGION=sa-east-1
```

#### Step 1.3: Create SSM Parameters
```bash
# Store JWT secret in SSM Parameter Store
aws ssm put-parameter \
  --name "/vitas/auth/jwt-secret" \
  --value "your-secure-jwt-secret-minimum-32-characters" \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso
```

#### Step 1.4: Bootstrap CDK (first time only)
```bash
npx cdk bootstrap aws://ACCOUNT_ID/sa-east-1 --profile vitas-sso
```

#### Step 1.5: Deploy Stack
```bash
npx cdk deploy --profile vitas-sso --require-approval never
```

#### Step 1.6: Capture Outputs
After deployment, note these outputs:
- API Gateway URL (e.g., `https://q36l2q61i5.execute-api.sa-east-1.amazonaws.com/prod/`)
- CloudFront Distribution Domain (e.g., `d2k0hzw5g3avku.cloudfront.net`)
- DynamoDB table names (all 7 tables)

---

### 2. Deploy vitas-processing-stack

**Purpose:** Medical audio processing with AI-powered transcription and analysis

**Location:** `vitas-processing-stack/`

#### Step 2.1: Install Dependencies
```bash
cd vitas-processing-stack
npm install

# Install Lambda layer dependencies
cd layers/processing-layer/nodejs
npm install
cd ../../..
```

#### Step 2.2: Configure Environment Variables
Create `.env` file in `vitas-processing-stack/`:
```bash
# AWS Region
AWS_REGION=sa-east-1

# DynamoDB Tables (from vitas-main-stack)
APPOINTMENTS_TABLE=Appointments_Table_V2
DIAGNOSIS_TABLE=Diagnosis_Table_V2

# SNS Topic ARN (from vitas-main-stack CloudFormation exports)
SNS_TOPIC_ARN=arn:aws:sns:sa-east-1:ACCOUNT_ID:VitasNotificationsV2

# S3 Bucket for media processing
MEDIA_BUCKET=vitas-media-processing-ACCOUNT_ID-v2
```

#### Step 2.3: Create SSM Parameters
```bash
# OpenAI API Key
aws ssm put-parameter \
  --name "/vitas/openai-api-key" \
  --value "sk-proj-..." \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso

# Anthropic API Key
aws ssm put-parameter \
  --name "/vitas/anthropic-api-key" \
  --value "sk-ant-..." \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso

# Frontend Webhook URL (will update after vitas-client deployment)
aws ssm put-parameter \
  --name "/vitas/webhook-url" \
  --value "https://your-frontend-domain.vercel.app/api/processing-webhook" \
  --type "String" \
  --region sa-east-1 \
  --profile vitas-sso

# Vercel Protection Bypass Secret (optional, for Vercel deployments)
aws ssm put-parameter \
  --name "/vitas/vercel-bypass-secret" \
  --value "your-vercel-protection-bypass-token" \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso

# SOAP Report Prompt
aws ssm put-parameter \
  --name "/vitas/prompts/soap-report" \
  --value "$(cat prompts/soap-report-prompt.txt)" \
  --type "String" \
  --region sa-east-1 \
  --profile vitas-sso

# ICD Diagnosis Prompt
aws ssm put-parameter \
  --name "/vitas/prompts/icd-diagnosis" \
  --value "$(cat prompts/icd-diagnosis-prompt.txt)" \
  --type "String" \
  --region sa-east-1 \
  --profile vitas-sso
```

#### Step 2.4: Build Lambda Functions
```bash
# Transpile TypeScript to JavaScript
npm run build

# Build each Lambda function
cd lambdas/transcription && npm install && cd ../..
cd lambdas/soap-report && npm install && cd ../..
cd lambdas/icd-diagnosis && npm install && cd ../..
cd lambdas/workflow-trigger && npm install && cd ../..
cd lambdas/failure-notification && npm install && cd ../..
cd lambdas/success-notification && npm install && cd ../..
cd lambdas/dlq-processor && npm install && cd ../..
```

#### Step 2.5: Deploy Stack
```bash
npx cdk deploy --profile vitas-sso --require-approval never
```

#### Step 2.6: Capture Outputs
After deployment, note these outputs:
- S3 bucket name for media processing
- Step Functions workflow ARNs (Transcription, Analysis)
- Lambda function names (7 functions)
- DLQ queue URLs (3 queues)

---

### 3. Deploy vitas-auth-otp

**Purpose:** OTP authentication and WhatsApp chatbot gateway

**Location:** `vitas-auth-otp/`

#### Step 3.1: Install Dependencies
```bash
cd vitas-auth-otp
npm install
```

#### Step 3.2: Configure Environment Variables
Create `.env` file in `vitas-auth-otp/`:
```bash
# AWS Region
AWS_REGION=sa-east-1

# DynamoDB Tables (from vitas-main-stack)
PATIENTS_TABLE=Patients_Table_V2
PATIENTDOCTORS_TABLE=PatientDoctors_Table_V2
DOCTORS_TABLE=Doctors_Table_V2
APPOINTMENTS_TABLE=Appointments_Table_V2
VITAS_APPOINTMENT_TABLE=Vitas_Appointment
SCHEDULE_CONFIG_TABLE=Vitas_Schedule_Config

# MercadoPago
MP_ACCESS_TOKEN=your-mercadopago-access-token

# LangGraph Configuration
LANGGRAPH_URL=https://your-langgraph-deployment.langchain.com
LANGSMITH_ASSISTANT_ID=your-assistant-id
AGENT_API_KEY=your-langgraph-api-key
LANGGRAPH_CALLBACK_URL=https://your-api-gateway-url.amazonaws.com/prod/agent-webhook
```

#### Step 3.3: Create SSM Parameters
```bash
# Twilio Configuration (JSON format)
aws ssm put-parameter \
  --name "/vitas-clinic/twilio-config" \
  --value '{"accountSid":"ACxxxx","authToken":"xxxx","fromNumber":"whatsapp:+14155238886"}' \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso

# JWT Secret (same as main-stack or separate)
aws ssm put-parameter \
  --name "/vitas-clinic/jwt-secret" \
  --value "your-secure-jwt-secret-minimum-32-characters" \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso
```

#### Step 3.4: Build Lambda Functions
```bash
# Build each Lambda function
cd lambda-functions/send-otp && npm install && npx tsc && cd ../..
cd lambda-functions/verify-otp && npm install && npx tsc && cd ../..
cd lambda-functions/chatbot-send-otp && npm install && npx tsc && cd ../..
cd lambda-functions/chatbot-verify-otp && npm install && npx tsc && cd ../..
cd lambda-functions/get-appointment && npm install && npx tsc && cd ../..
cd lambda-functions/chat-integration && npm install && npx tsc && cd ../..
cd lambda-functions/agent-webhook && npm install && npx tsc && cd ../..
cd lambda-functions/confirm-pay && npm install && npx tsc && cd ../..
cd lambda-functions/paylink && npm install && npx tsc && cd ../..
```

#### Step 3.5: Deploy Stack
```bash
npx cdk deploy --profile vitas-sso --require-approval never
```

#### Step 3.6: Configure Twilio Webhooks
After deployment, configure Twilio WhatsApp webhook:
1. Log in to Twilio Console
2. Navigate to Programmable Messaging → WhatsApp Sandbox
3. Set webhook URL to: `https://{api-gateway-url}/prod/chat-webhook`
4. Set HTTP method to POST

#### Step 3.7: Configure MercadoPago Webhooks
1. Log in to MercadoPago Dashboard
2. Navigate to Webhooks configuration
3. Add webhook URL: `https://{api-gateway-url}/prod/confirm-pay`
4. Select event types: Payment (approved, rejected)

#### Step 3.8: Capture Outputs
After deployment, note these outputs:
- API Gateway URL (e.g., `https://xxxxx.execute-api.sa-east-1.amazonaws.com/prod/`)
- Lambda function names (9 functions)
- DynamoDB table names (3 new tables)

---

### 4. Deploy vitas-chatbot (LangGraph)

**Purpose:** AI conversational assistant with role-based agents

**Location:** `vitas-chatbot/`

#### Step 4.1: Install Dependencies
```bash
cd vitas-chatbot
pip install -e .
```

#### Step 4.2: Configure Environment Variables
Create `.env` file in `vitas-chatbot/`:
```bash
# OpenAI Configuration
OPENAI_API_KEY=sk-proj-...

# Anthropic Configuration (optional fallback)
ANTHROPIC_API_KEY=sk-ant-...

# LangSmith Monitoring (optional)
LANGSMITH_API_KEY=your-langsmith-key
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=vitas-chatbot

# DynamoDB Tables (from vitas-main-stack)
AWS_DOCTORS_TABLE=Doctors_Table_V2
AWS_PATIENTSDOCTORS_TABLE=PatientDoctors_Table_V2
AWS_PATIENTS_DETAILS_TABLE=Patients_Table_V2
AWS_VIRTUAL_APPOINTMENTS_TABLE=Vitas_Appointment
AWS_SCHEDULE_APPOINTMENTS_TABLE=Vitas_Schedule_Config

# AWS Region
AWS_REGION=sa-east-1

# JWT Secret (same as auth-otp)
JWT_SECRET=your-secure-jwt-secret-minimum-32-characters
```

#### Step 4.3: Test Locally (optional)
```bash
# Start LangGraph development server
langgraph dev
```

#### Step 4.4: Deploy to LangGraph Cloud
```bash
# Login to LangGraph Cloud
langgraph login

# Deploy
langgraph deploy
```

#### Step 4.5: Capture Deployment URL
After deployment, note:
- LangGraph deployment URL (e.g., `https://your-deployment.langchain.com`)
- Assistant ID from LangSmith dashboard

#### Step 4.6: Update vitas-auth-otp Environment
Update the Lambda environment variables in vitas-auth-otp:
```bash
# Update chat-integration Lambda
aws lambda update-function-configuration \
  --function-name VitasChatbotStack-ChatIntegration* \
  --environment "Variables={LANGGRAPH_URL=https://your-deployment.langchain.com,LANGSMITH_ASSISTANT_ID=your-assistant-id,AGENT_API_KEY=your-api-key}" \
  --region sa-east-1 \
  --profile vitas-sso
```

---

## Frontend Deployment

### 5. Deploy vitas-client (Doctor Dashboard)

**Purpose:** Doctor-facing web application

**Location:** `vitas-client/`

#### Step 5.1: Install Dependencies
```bash
cd vitas-client
npm install
```

#### Step 5.2: Configure Environment Variables
Create `.env.local` file in `vitas-client/`:
```bash
# API Gateway URLs (from AWS deployments)
NEXT_PUBLIC_API_BASE_URL=https://q36l2q61i5.execute-api.sa-east-1.amazonaws.com/prod
NEXT_PUBLIC_AUTH_API_URL=https://xxxxx.execute-api.sa-east-1.amazonaws.com/prod

# CloudFront Domain
NEXT_PUBLIC_CLOUDFRONT_DOMAIN=d2k0hzw5g3avku.cloudfront.net

# LangGraph Configuration
LANGGRAPH_DEPLOY_URL=https://your-deployment.langchain.com
LANGSMITH_API_KEY=your-langsmith-key

# Frontend URL (will be Vercel domain)
NEXT_PUBLIC_FRONTEND_URL=https://your-domain.vercel.app

# JWT Secret (server-side only)
JWT_SECRET=your-secure-jwt-secret-minimum-32-characters

# Deepgram API Key (optional)
DEEPGRAM_API_KEY=your-deepgram-key

# Upstash Redis (for rate limiting)
UPSTASH_REDIS_REST_URL=your-upstash-url
UPSTASH_REDIS_REST_TOKEN=your-upstash-token

# AWS Region
AWS_REGION=sa-east-1

# AWS Credentials (for server-side S3/DynamoDB access)
AWS_ACCESS_KEY_ID=your-access-key
AWS_SECRET_ACCESS_KEY=your-secret-key
```

#### Step 5.3: Build and Test Locally
```bash
# Development server
npm run dev

# Production build (test)
npm run build
npm start
```

#### Step 5.4: Deploy to Vercel
```bash
# Install Vercel CLI (if not installed)
npm install -g vercel

# Login to Vercel
vercel login

# Deploy
vercel --prod

# Or connect GitHub repository for automatic deployments
```

#### Step 5.5: Configure Vercel Environment Variables
In Vercel dashboard, add all environment variables from `.env.local`

#### Step 5.6: Update Webhook URL in Processing Stack
After Vercel deployment, update the webhook URL:
```bash
aws ssm put-parameter \
  --name "/vitas/webhook-url" \
  --value "https://your-domain.vercel.app/api/processing-webhook" \
  --type "String" \
  --overwrite \
  --region sa-east-1 \
  --profile vitas-sso
```

---

### 6. Deploy vitas-patients-front (Patient Portal)

**Purpose:** Patient-facing web application

**Location:** `vitas-patients-front/`

#### Step 6.1: Install Dependencies
```bash
cd vitas-patients-front
npm install
```

#### Step 6.2: Configure Environment Variables
Create `.env.local` file:
```bash
# API Gateway URL (from vitas-auth-otp)
NEXT_PUBLIC_API_BASE_URL=https://xxxxx.execute-api.sa-east-1.amazonaws.com/prod

# LangGraph URL
NEXT_PUBLIC_LANGGRAPH_URL=https://your-deployment.langchain.com

# Frontend URL
NEXT_PUBLIC_FRONTEND_URL=https://patients.your-domain.com

# JWT Secret (server-side)
JWT_SECRET=your-secure-jwt-secret-minimum-32-characters
```

#### Step 6.3: Deploy to Vercel
```bash
vercel --prod
```

---

## Post-Deployment Verification

### 1. Verify vitas-main-stack

#### Test Authentication Endpoint
```bash
curl -X POST https://q36l2q61i5.execute-api.sa-east-1.amazonaws.com/prod/auth/signin \
  -H "Content-Type: application/json" \
  -d '{
    "email": "doctor@example.com",
    "password": "password123"
  }'
```

Expected: 200 OK with JWT token

#### Test Health Endpoint
```bash
curl https://q36l2q61i5.execute-api.sa-east-1.amazonaws.com/prod/health
```

Expected: `{"status": "ok", "timestamp": "..."}`

#### Verify DynamoDB Tables
```bash
aws dynamodb list-tables --region sa-east-1 --profile vitas-sso | grep Vitas
```

Expected: 7 tables listed

### 2. Verify vitas-processing-stack

#### Test S3 Upload and Processing
1. Upload test audio file to S3:
```bash
aws s3 cp test-audio.mp3 \
  s3://vitas-media-processing-ACCOUNT_ID-v2/audios/doctor-123/patient-456/appointment-789.mp3 \
  --region sa-east-1 \
  --profile vitas-sso
```

2. Check Step Functions execution:
```bash
aws stepfunctions list-executions \
  --state-machine-arn arn:aws:states:sa-east-1:ACCOUNT_ID:stateMachine:VitasTranscriptionWorkflow \
  --region sa-east-1 \
  --profile vitas-sso
```

3. Monitor CloudWatch Logs:
```bash
aws logs tail /aws/lambda/VitasProcessingStack-TranscriptionFunction \
  --follow \
  --region sa-east-1 \
  --profile vitas-sso
```

#### Verify EventBridge Rule
```bash
aws events list-rules --name-prefix vitas-dlq-retry \
  --region sa-east-1 \
  --profile vitas-sso
```

Expected: Rule with 2-hour schedule

### 3. Verify vitas-auth-otp

#### Test OTP Send
```bash
curl -X POST https://xxxxx.execute-api.sa-east-1.amazonaws.com/prod/otp/send \
  -H "Content-Type: application/json" \
  -d '{
    "phoneNumber": "+5511999999999",
    "patientName": "Test Patient"
  }'
```

Expected: 200 OK with success message

#### Test WhatsApp Webhook
Send test message to WhatsApp Business number and check Lambda logs:
```bash
aws logs tail /aws/lambda/VitasChatbotStack-ChatIntegration \
  --follow \
  --region sa-east-1 \
  --profile vitas-sso
```

### 4. Verify vitas-chatbot

#### Test LangGraph API
```bash
curl -X POST https://your-deployment.langchain.com/threads/test-thread/runs \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "assistant_id": "your-assistant-id",
    "input": {"messages": [{"role": "user", "content": "Hola, quiero agendar una cita"}]}
  }'
```

Expected: 200 OK with thread response

#### Check LangSmith Monitoring
Visit LangSmith dashboard to verify:
- Agent traces appearing
- Tool calls executing
- No errors in logs

### 5. Verify Frontend Deployments

#### Test vitas-client
1. Visit: `https://your-domain.vercel.app`
2. Login with doctor credentials
3. Test chatbot sidebar
4. Record test audio and verify processing notifications

#### Test vitas-patients-front
1. Visit: `https://patients.your-domain.com`
2. Request OTP for phone number
3. Verify OTP via WhatsApp
4. Test chatbot for appointment booking

### 6. End-to-End Integration Test

Complete flow test:
1. **Upload Audio** (Doctor dashboard)
2. **Verify S3 Upload** (AWS Console)
3. **Monitor Step Functions** (AWS Console)
4. **Check Processing Notification** (Doctor dashboard)
5. **Verify SOAP Report** (DynamoDB / Doctor dashboard)
6. **Test Chatbot Booking** (WhatsApp)
7. **Complete Payment** (MercadoPago)
8. **Verify Appointment Confirmation** (WhatsApp + Dashboard)

---

## Environment Variables

### Complete Environment Variable Reference

#### vitas-main-stack
```bash
JWT_SECRET=                  # JWT signing key (32+ chars)
PROFILE_PICTURE_BUCKET=      # S3 bucket name
SNS_TOPIC_NAME=              # SNS topic name
AWS_REGION=sa-east-1         # AWS region
```

#### vitas-processing-stack
```bash
AWS_REGION=sa-east-1                    # AWS region
APPOINTMENTS_TABLE=Appointments_Table_V2 # DynamoDB table
DIAGNOSIS_TABLE=Diagnosis_Table_V2       # DynamoDB table
SNS_TOPIC_ARN=                          # SNS topic ARN
MEDIA_BUCKET=                           # S3 bucket for media

# SSM Parameters:
/vitas/openai-api-key
/vitas/anthropic-api-key
/vitas/webhook-url
/vitas/vercel-bypass-secret
/vitas/prompts/soap-report
/vitas/prompts/icd-diagnosis
```

#### vitas-auth-otp
```bash
AWS_REGION=sa-east-1                      # AWS region
PATIENTS_TABLE=Patients_Table_V2           # DynamoDB table
PATIENTDOCTORS_TABLE=PatientDoctors_Table_V2
DOCTORS_TABLE=Doctors_Table_V2
APPOINTMENTS_TABLE=Appointments_Table_V2
VITAS_APPOINTMENT_TABLE=Vitas_Appointment
SCHEDULE_CONFIG_TABLE=Vitas_Schedule_Config
MP_ACCESS_TOKEN=                          # MercadoPago token
LANGGRAPH_URL=                            # LangGraph API URL
LANGSMITH_ASSISTANT_ID=                   # Assistant ID
AGENT_API_KEY=                            # LangGraph API key
LANGGRAPH_CALLBACK_URL=                   # Webhook URL

# SSM Parameters:
/vitas-clinic/twilio-config
/vitas-clinic/jwt-secret
```

#### vitas-chatbot
```bash
OPENAI_API_KEY=                         # OpenAI API key
ANTHROPIC_API_KEY=                      # Anthropic API key
LANGSMITH_API_KEY=                      # LangSmith key
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=vitas-chatbot
AWS_DOCTORS_TABLE=Doctors_Table_V2
AWS_PATIENTSDOCTORS_TABLE=PatientDoctors_Table_V2
AWS_PATIENTS_DETAILS_TABLE=Patients_Table_V2
AWS_VIRTUAL_APPOINTMENTS_TABLE=Vitas_Appointment
AWS_SCHEDULE_APPOINTMENTS_TABLE=Vitas_Schedule_Config
AWS_REGION=sa-east-1
JWT_SECRET=                             # JWT secret
```

#### vitas-client
```bash
NEXT_PUBLIC_API_BASE_URL=               # Main API URL
NEXT_PUBLIC_AUTH_API_URL=               # Auth API URL
NEXT_PUBLIC_CLOUDFRONT_DOMAIN=          # CloudFront domain
LANGGRAPH_DEPLOY_URL=                   # LangGraph URL (server-side)
LANGSMITH_API_KEY=                      # LangSmith key (server-side)
NEXT_PUBLIC_FRONTEND_URL=               # Frontend URL
JWT_SECRET=                             # JWT secret (server-side)
DEEPGRAM_API_KEY=                       # Deepgram key (optional)
UPSTASH_REDIS_REST_URL=                 # Upstash Redis URL
UPSTASH_REDIS_REST_TOKEN=               # Upstash token
AWS_REGION=sa-east-1
AWS_ACCESS_KEY_ID=                      # AWS credentials (server-side)
AWS_SECRET_ACCESS_KEY=                  # AWS secret (server-side)
```

---

## Troubleshooting

### Common Deployment Issues

#### Issue: CDK Bootstrap Required
**Error:** `This stack uses assets, so the toolkit stack must be deployed`

**Solution:**
```bash
npx cdk bootstrap aws://ACCOUNT_ID/sa-east-1 --profile vitas-sso
```

#### Issue: Lambda Package Too Large
**Error:** `Unzipped size must be smaller than 262144000 bytes`

**Solution:**
- Remove `node_modules` and reinstall with `--production` flag
- Exclude unnecessary files in `.npmignore`
- Use Lambda layers for shared dependencies

#### Issue: DynamoDB Table Not Found
**Error:** `Cannot do operations on a non-existent table`

**Solution:**
- Verify vitas-main-stack deployed successfully
- Check table names in AWS Console match environment variables
- Ensure correct AWS region (sa-east-1)

#### Issue: SSM Parameter Not Found
**Error:** `ParameterNotFound: Parameter not found`

**Solution:**
```bash
# List all parameters
aws ssm describe-parameters --region sa-east-1 --profile vitas-sso

# Verify specific parameter exists
aws ssm get-parameter --name "/vitas/openai-api-key" --region sa-east-1 --profile vitas-sso
```

#### Issue: API Gateway 403 Forbidden
**Error:** `{"message":"Forbidden"}`

**Solution:**
- Check API key is included in `x-api-key` header
- Verify rate limiting hasn't been exceeded
- Check CORS configuration allows your origin

#### Issue: Step Functions Timeout
**Error:** `States.Timeout: Task timed out`

**Solution:**
- Increase Lambda timeout in CDK stack
- Increase Step Functions workflow timeout
- Check CloudWatch Logs for underlying Lambda errors

#### Issue: Twilio Webhook Validation Failed
**Error:** `Invalid Twilio signature`

**Solution:**
- Verify Twilio auth token in SSM Parameter Store
- Check webhook URL matches exactly (including HTTPS)
- Ensure Twilio account SID is correct

#### Issue: LangGraph Deployment Failed
**Error:** `Deployment failed: Invalid configuration`

**Solution:**
```bash
# Validate langgraph.json
cat langgraph.json

# Check environment variables
cat .env

# Re-authenticate
langgraph login

# Deploy with verbose logging
langgraph deploy --verbose
```

### Rollback Procedures

#### Rollback CDK Stack
```bash
# List stack history
aws cloudformation describe-stack-events \
  --stack-name VitasMainStack \
  --region sa-east-1 \
  --profile vitas-sso

# Rollback to previous version
aws cloudformation cancel-update-stack \
  --stack-name VitasMainStack \
  --region sa-east-1 \
  --profile vitas-sso
```

#### Rollback Vercel Deployment
```bash
# List deployments
vercel ls

# Rollback to specific deployment
vercel rollback <deployment-url>
```

#### Rollback LangGraph Deployment
```bash
# List deployments
langgraph deployments list

# Rollback to previous version
langgraph rollback <deployment-id>
```

### Monitoring Deployment Health

#### CloudWatch Dashboard
Visit CloudWatch Console → Dashboards → `VITAS-Medical-Processing-Pipeline`

#### CloudWatch Alarms
```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix "Vitas" \
  --region sa-east-1 \
  --profile vitas-sso
```

#### Lambda Logs
```bash
# Tail specific Lambda
aws logs tail /aws/lambda/FUNCTION_NAME --follow --region sa-east-1 --profile vitas-sso
```

#### Step Functions Executions
```bash
aws stepfunctions list-executions \
  --state-machine-arn <ARN> \
  --status-filter RUNNING \
  --region sa-east-1 \
  --profile vitas-sso
```

---

## Next Steps

After successful deployment:
1. **Review Monitoring:** Check CloudWatch dashboard and set up alerts
2. **Test All Flows:** Run end-to-end integration tests
3. **Security Audit:** Review IAM policies, enable CloudTrail
4. **Performance Tuning:** Optimize Lambda memory/timeout based on metrics
5. **Documentation:** Update internal docs with actual URLs and ARNs

---

**Related Documentation:**
- [System Architecture Overview](01-SYSTEM_ARCHITECTURE_OVERVIEW.md)
- [Security & Compliance](09-SECURITY_AND_COMPLIANCE.md)
- [Monitoring & Observability](08-MONITORING_AND_OBSERVABILITY.md)

**Last Updated:** 2025-11-06
