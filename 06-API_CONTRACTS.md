# API Contracts

## Table of Contents
- [API Gateway Endpoints](#api-gateway-endpoints)
- [Request/Response Schemas](#requestresponse-schemas)
- [Authentication](#authentication)
- [Error Responses](#error-responses)
- [Rate Limiting](#rate-limiting)

## API Gateway Endpoints

### vitas-main-stack API

**Base URL:** `https://q36l2q61i5.execute-api.sa-east-1.amazonaws.com/prod/`

#### Authentication Endpoints
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/signin` | None | Doctor login |
| POST | `/auth/signup` | None | Doctor registration |
| GET | `/auth/profile` | Bearer | Get doctor profile |
| PUT | `/auth/profile` | Bearer | Update doctor profile |
| DELETE | `/auth/profile` | Bearer | Delete doctor account |
| PUT | `/auth/password` | Bearer | Change password |
| POST | `/auth/upload-url` | Bearer | Generate presigned URL for profile picture |
| GET | `/health` | None | Health check |

### vitas-auth-otp API

**Base URL:** `https://xxxxx.execute-api.sa-east-1.amazonaws.com/prod/`

#### OTP Endpoints
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/otp/send` | None | Send OTP for payment |
| POST | `/otp/verify` | None | Verify OTP, get 24h JWT |
| POST | `/chatbot/send-otp` | None | Send OTP for chatbot |
| POST | `/chatbot/verify-otp` | None | Verify OTP, get 2h JWT |

#### Appointment Endpoints
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/appointments/get` | Bearer | Get appointment details |

#### Webhook Endpoints
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/chat-webhook` | Twilio Signature | WhatsApp message webhook |
| POST | `/agent-webhook` | None | LangGraph callback |
| POST | `/confirm-pay` | None | MercadoPago payment webhook |

### vitas-client API (Next.js Routes)

**Base URL:** `https://your-domain.vercel.app/api/`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/signin` | None | Proxies to AWS auth |
| GET | `/get-patients-by-doctor` | Cookie | Get doctor's patients |
| GET | `/get-consultation-details` | Cookie | Get appointment details |
| POST | `/audio-to-s3-presigned-url` | Cookie | Get presigned S3 URL |
| POST | `/processing-webhook` | AWS Lambda | Receive processing notifications |
| GET | `/processing-webhook` | Cookie | Poll for notifications |
| ALL | `/chat/[...path]` | Cookie | Proxy to LangGraph |

---

## Request/Response Schemas

### Authentication

#### POST /auth/signin
**Request:**
```json
{
  "email": "doctor@example.com",
  "password": "SecurePassword123!"
}
```

**Response (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "doctor": {
    "doctor_id": "doctor-123",
    "email": "doctor@example.com",
    "full_name": "Dr. Juan Pérez",
    "specialty": "Cardiología",
    "created_at": "2025-01-01T00:00:00Z"
  }
}
```

### OTP Authentication

#### POST /otp/send
**Request:**
```json
{
  "phoneNumber": "+5511999999999",
  "patientName": "João Silva"
}
```

**Response (200):**
```json
{
  "success": true,
  "message": "OTP sent via WhatsApp"
}
```

#### POST /otp/verify
**Request:**
```json
{
  "phoneNumber": "+5511999999999",
  "otpCode": "123456"
}
```

**Response (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "patient": {
    "patient_id": "patient-456",
    "phone_number": "+5511999999999",
    "patient_name": "João Silva",
    "status": "registered_via_whatsapp",
    "created_at": "2025-11-06T10:00:00Z"
  }
}
```

### Processing Notifications

#### POST /api/processing-webhook (Receive from Lambda)
**Request:**
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

**Response (200):**
```json
{
  "success": true,
  "message": "Notification received"
}
```

#### GET /api/processing-webhook (Frontend polling)
**Request:**
```
GET /api/processing-webhook?doctorId=doctor-123
```

**Response (200):**
```json
{
  "notifications": [
    {
      "id": "appointment-789-transcription-1730897400000",
      "type": "success",
      "message": "Procesamiento completado para la cita de João Silva",
      "appointmentId": "appointment-789",
      "service": "transcription-complete",
      "timestamp": "2025-11-06T10:30:00Z",
      "metadata": {
        "appointmentId": "appointment-789",
        "patientId": "patient-456"
      }
    }
  ],
  "count": 1,
  "timestamp": "2025-11-06T10:30:10Z"
}
```

---

## Authentication

### JWT Token Structure

**Doctor Token (24h):**
```json
{
  "doctorId": "doctor-123",
  "email": "doctor@example.com",
  "role": "doctor",
  "exp": 1730980800
}
```

**Patient Token (Payment - 24h):**
```json
{
  "patientId": "patient-456",
  "phoneNumber": "+5511999999999",
  "role": "patient",
  "exp": 1730980800
}
```

**Patient Token (Chatbot - 2h):**
```json
{
  "patientId": "patient-456",
  "phoneNumber": "+5511999999999",
  "role": "patient",
  "chatbot_access": true,
  "exp": 1730890800
}
```

### Authorization Headers

**Bearer Token:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Cookie (Frontend):**
```
Cookie: authToken=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...; HttpOnly; Secure; SameSite=Lax
```

---

## Error Responses

### Standard Error Format
```json
{
  "error": "Error message",
  "message": "Detailed description",
  "statusCode": 400,
  "timestamp": "2025-11-06T10:00:00Z"
}
```

### Common Error Codes
| Code | Message | Description |
|------|---------|-------------|
| 400 | Bad Request | Invalid request parameters |
| 401 | Unauthorized | Missing or invalid authentication |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |

---

## Rate Limiting

### API Gateway Rate Limits

**vitas-main-stack:**
- 100 requests/second per API key
- 200 burst capacity
- 10,000 requests/day quota

**vitas-auth-otp:**
- OTP endpoints: 3 attempts per phone number per 5 minutes
- General endpoints: 1,000 requests/hour per IP

### Frontend Rate Limiting (Upstash Redis)

**Configuration:**
```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),
  analytics: true
});

// Usage
const { success, limit, remaining } = await ratelimit.limit(identifier);
```

---

**Last Updated:** 2025-11-06
