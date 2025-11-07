# Security & Compliance

## Table of Contents
- [Authentication & Authorization](#authentication--authorization)
- [Data Encryption](#data-encryption)
- [IAM Policies](#iam-policies)
- [Secrets Management](#secrets-management)
- [Network Security](#network-security)
- [Compliance Considerations](#compliance-considerations)

## Authentication & Authorization

### JWT Token Management

**Token Types:**
| Token Type | Expiry | Use Case | Storage |
|------------|--------|----------|---------|
| Doctor Token | 24 hours | Dashboard access | HTTP-only cookie |
| Patient Payment Token | 24 hours | Appointment booking | HTTP-only cookie |
| Patient Chatbot Token | 2 hours | WhatsApp chat | wa_sessions table |

**JWT Signing:**
- Algorithm: HS256
- Secret stored in: AWS Systems Manager Parameter Store
- Secret rotation: Manual (recommend quarterly)

**Token Validation:**
```typescript
import * as jose from 'jose';

async function validateToken(token: string): Promise<JWTPayload> {
  const secret = new TextEncoder().encode(process.env.JWT_SECRET);

  try {
    const { payload } = await jose.jwtVerify(token, secret);
    return payload;
  } catch (error) {
    throw new Error('Invalid or expired token');
  }
}
```

### Role-Based Access Control (RBAC)

**Roles:**
- **Doctor:** Full access to own patients, appointments, consultations
- **Patient:** Access to own profile, appointments, payment
- **Admin:** (Future) System-wide access

**Authorization Check:**
```typescript
// Middleware example
function requireRole(allowedRoles: string[]) {
  return async (request: Request) => {
    const token = extractToken(request);
    const payload = await validateToken(token);

    if (!allowedRoles.includes(payload.role)) {
      throw new Error('Insufficient permissions');
    }

    return payload;
  };
}
```

---

## Data Encryption

### Encryption at Rest

**DynamoDB:**
- Encryption: AWS-managed keys (default)
- All tables encrypted automatically
- No performance impact

**S3:**
- Encryption: AES-256 (SSE-S3)
- All buckets encrypted by default
- Applied to: Audio files, transcripts, SOAP reports

**Systems Manager Parameter Store:**
- SecureString parameters use AWS KMS
- Automatic encryption/decryption
- Used for: JWT secrets, API keys, Twilio config

### Encryption in Transit

**All Communication Encrypted:**
- API Gateway: HTTPS only (TLS 1.2+)
- Frontend: HTTPS enforced (Vercel)
- S3 uploads: HTTPS presigned URLs
- Database: DynamoDB uses TLS
- External APIs: HTTPS (OpenAI, Anthropic, Twilio, MercadoPago)

**CORS Configuration:**
```typescript
// API Gateway CORS
{
  allowOrigins: ['https://your-domain.vercel.app'],
  allowMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
  allowHeaders: ['Content-Type', 'Authorization', 'X-Api-Key'],
  maxAge: 3600
}
```

---

## IAM Policies

### Principle of Least Privilege

Each Lambda function has minimal IAM permissions:

#### Transcription Function Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::vitas-media-processing-*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter"
      ],
      "Resource": "arn:aws:ssm:sa-east-1:*:parameter/vitas/openai-api-key"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:sa-east-1:*:log-group:/aws/lambda/*"
    }
  ]
}
```

#### SOAP Report Function Policy
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::vitas-media-processing-*/transcripts/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:sa-east-1:*:table/Appointments_Table_V2"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter"],
      "Resource": [
        "arn:aws:ssm:sa-east-1:*:parameter/vitas/openai-api-key",
        "arn:aws:ssm:sa-east-1:*:parameter/vitas/prompts/soap-report"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "arn:aws:sns:sa-east-1:*:VitasNotificationsV2"
    }
  ]
}
```

#### Step Functions Execution Role
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["lambda:InvokeFunction"],
      "Resource": [
        "arn:aws:lambda:sa-east-1:*:function:VitasProcessingStack-Transcription*",
        "arn:aws:lambda:sa-east-1:*:function:VitasProcessingStack-SOAP*",
        "arn:aws:lambda:sa-east-1:*:function:VitasProcessingStack-ICD*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["sqs:SendMessage"],
      "Resource": [
        "arn:aws:sqs:sa-east-1:*:vitas-transcription-dlq*",
        "arn:aws:sqs:sa-east-1:*:vitas-soap-report-dlq*",
        "arn:aws:sqs:sa-east-1:*:vitas-icd-diagnosis-dlq*"
      ]
    }
  ]
}
```

---

## Secrets Management

### AWS Systems Manager Parameter Store

**All Secrets Stored in SSM:**
| Parameter Name | Type | Description |
|----------------|------|-------------|
| `/vitas/auth/jwt-secret` | SecureString | JWT signing key |
| `/vitas/openai-api-key` | SecureString | OpenAI API key |
| `/vitas/anthropic-api-key` | SecureString | Anthropic API key |
| `/vitas/webhook-url` | String | Frontend webhook URL |
| `/vitas/vercel-bypass-secret` | SecureString | Vercel Protection bypass |
| `/vitas-clinic/twilio-config` | SecureString | Twilio credentials (JSON) |
| `/vitas-clinic/jwt-secret` | SecureString | JWT secret (auth-otp) |

**Creating Secrets:**
```bash
aws ssm put-parameter \
  --name "/vitas/openai-api-key" \
  --value "sk-proj-..." \
  --type "SecureString" \
  --region sa-east-1 \
  --profile vitas-sso
```

**Retrieving Secrets (Lambda):**
```typescript
import { SSMClient, GetParameterCommand } from '@aws-sdk/client-ssm';

const ssm = new SSMClient({ region: process.env.AWS_REGION });

async function getSecret(name: string): Promise<string> {
  const response = await ssm.send(new GetParameterCommand({
    Name: name,
    WithDecryption: true
  }));

  if (!response.Parameter?.Value) {
    throw new Error(`Secret ${name} not found`);
  }

  return response.Parameter.Value;
}
```

### Secret Rotation

**Recommended Schedule:**
- JWT secrets: Every 90 days
- API keys: When compromised or annually
- Twilio credentials: Annually

**Rotation Process:**
1. Create new secret in SSM
2. Update Lambda environment variables to use new parameter
3. Deploy updated Lambda functions
4. Verify functionality
5. Delete old secret after 7-day grace period

---

## Network Security

### API Gateway Security

**API Keys:**
- All main-stack APIs require `x-api-key` header
- Rate limiting: 100 req/sec, 10k/day quota
- Keys rotated quarterly

**Request Validation:**
- JSON schema validation enabled
- Request size limits: 1MB (Gateway), 6MB (Lambda payload)
- Query string parameter validation

**Throttling:**
```typescript
// API Gateway usage plan
{
  throttle: {
    rateLimit: 100,  // requests per second
    burstLimit: 200   // burst capacity
  },
  quota: {
    limit: 10000,     // requests per day
    period: 'DAY'
  }
}
```

### S3 Bucket Security

**Block Public Access:** Enabled on all buckets

**Bucket Policy (vitas-media-processing):**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::vitas-media-processing-*",
        "arn:aws:s3:::vitas-media-processing-*/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

**Presigned URL Security:**
- Expiration: 15 minutes (audio uploads)
- HTTPS only
- Single-use recommended (generate per upload)

### CloudFront Security

**Origin Access Control (OAC):**
- S3 bucket not publicly accessible
- CloudFront has exclusive access via OAC
- Prevents direct S3 URL access

**HTTPS Enforcement:**
- Redirect HTTP to HTTPS
- TLS 1.2 minimum
- Supports HTTP/2 and HTTP/3

---

## Compliance Considerations

### HIPAA Compliance (Roadmap)

**Current Status:** Not HIPAA compliant (in development)

**Requirements for HIPAA:**
1. ✅ Encryption at rest (DynamoDB, S3)
2. ✅ Encryption in transit (TLS)
3. ⏳ Business Associate Agreements (BAA) with AWS
4. ⏳ Audit logging (CloudTrail)
5. ⏳ Access controls and authentication
6. ⏳ Data retention and deletion policies
7. ⏳ Incident response procedures

**Steps to HIPAA Compliance:**
1. Sign AWS BAA
2. Enable CloudTrail for all API calls
3. Implement data retention policies
4. Document security procedures
5. Conduct security audit
6. Staff training on PHI handling

### HL7/FHIR Compliance (Roadmap)

**Current Status:** Not implemented

**Planned Features:**
- FHIR R4 API endpoints
- HL7 message parsing for EHR integration
- Interoperability with external systems
- Standardized data formats

### LGPD Compliance (Brazil)

**Data Subject Rights:**
- ✅ Data access (patients can view their data)
- ✅ Data deletion (delete endpoints implemented)
- ⏳ Data portability (export feature needed)
- ⏳ Consent management

**Data Processing:**
- ✅ Explicit consent for OTP (implicit via WhatsApp opt-in)
- ✅ Purpose limitation (medical consultations only)
- ✅ Data minimization (collect only necessary fields)
- ✅ Secure storage and transmission

---

## Security Best Practices

### 1. Regular Security Audits
- Review IAM policies quarterly
- Audit CloudTrail logs for suspicious activity
- Scan dependencies for vulnerabilities (npm audit, pip-audit)

### 2. Incident Response Plan
```
1. Detection: CloudWatch alarms trigger
2. Assessment: Review logs, determine impact
3. Containment: Rotate credentials if compromised
4. Eradication: Patch vulnerabilities
5. Recovery: Restore services
6. Lessons Learned: Document and improve
```

### 3. Code Security
- Input validation on all user inputs (Zod schemas)
- SQL injection prevention (using DynamoDB, no raw SQL)
- XSS prevention (React escapes by default)
- CSRF protection (SameSite cookies)

### 4. Third-Party Security
- OpenAI, Anthropic: HTTPS APIs, no PHI in prompts
- Twilio: Validate webhook signatures
- MercadoPago: Verify payment webhooks
- LangGraph Cloud: JWT authentication

### 5. Monitoring & Alerting
- Failed login attempts
- Unusual API access patterns
- DynamoDB capacity exceeded
- Lambda errors spike

---

## Security Checklist

**Before Production:**
- [ ] All secrets in SSM Parameter Store
- [ ] HTTPS enforced everywhere
- [ ] API keys rotated
- [ ] CloudTrail enabled
- [ ] Security alarms configured
- [ ] Incident response plan documented
- [ ] Dependency vulnerabilities resolved
- [ ] Penetration testing completed
- [ ] Legal review of data handling
- [ ] Privacy policy updated

---

**Last Updated:** 2025-11-06
