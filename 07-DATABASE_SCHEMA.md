# Database Schema

## Table of Contents
- [DynamoDB Tables Overview](#dynamodb-tables-overview)
- [Table Schemas](#table-schemas)
- [Global Secondary Indexes](#global-secondary-indexes)
- [Relationships](#relationships)
- [Access Patterns](#access-patterns)

## DynamoDB Tables Overview

**Total Tables:** 12
**Billing Mode:** PAY_PER_REQUEST (all tables)
**Region:** sa-east-1

| Table Name | Stack | PK | SK | GSI Count | TTL |
|------------|-------|----|----|-----------|-----|
| Patients_Table_V2 | main | patient_id | phone_number | 1 | No |
| PatientDoctors_Table_V2 | main | doctor_id | patient_id | 1 | No |
| Doctors_Table_V2 | main | doctor_id | - | 2 | No |
| Appointments_Table_V2 | main | patient_id | appointment_id | 3 | No |
| Diagnosis_Table_V2 | main | appointment_id | patient_id | 0 | No |
| Vitas_Appointment | main | doctor_id | appt | 1 | Yes |
| Vitas_Schedule_Config | main | doctor_id | config | 0 | No |
| vitas-otp | auth-otp | phoneNumber | - | 0 | Yes (5 min) |
| wa_sessions | auth-otp | pk (phone) | - | 0 | Yes (15 min) |
| vitas-paylinks | auth-otp | pk | - | 0 | Yes |

---

## Table Schemas

### 1. Patients_Table_V2

**Purpose:** Store patient profiles

**Keys:**
- **PK:** `patient_id` (String)
- **SK:** `phone_number` (String)

**GSI:**
- **phone_number-index:** PK=phone_number (for OTP lookup)

**Attributes:**
```typescript
{
  patient_id: string;           // UUID
  phone_number: string;         // +5511999999999
  patient_name: string;
  patient_email?: string;
  patient_age?: number;
  patient_dni?: string;         // ID document number
  status: string;               // "registered_via_whatsapp" | "registered_via_chatbot" | "active"
  chatbot_access: boolean;      // true if registered via chatbot
  last_login?: string;          // ISO timestamp
  created_at: string;           // ISO timestamp
  updated_at?: string;
}
```

### 2. PatientDoctors_Table_V2

**Purpose:** Junction table for doctor-patient relationships

**Keys:**
- **PK:** `doctor_id` (String)
- **SK:** `patient_id` (String)

**GSI:**
- **patient_id-index:** PK=patient_id (reverse lookup)

**Attributes:**
```typescript
{
  doctor_id: string;
  patient_id: string;
  relationship_type: string;   // "primary" | "referred"
  created_at: string;
  notes?: string;
}
```

### 3. Doctors_Table_V2

**Purpose:** Store doctor profiles

**Keys:**
- **PK:** `doctor_id` (String)

**GSI:**
- **email-index:** PK=email (for login)
- **active_for_ai_booking-index:** PK=active_for_ai_booking (filter active doctors)

**Attributes:**
```typescript
{
  doctor_id: string;
  email: string;
  password: string;             // bcrypt hash
  full_name: string;
  speciality: string;           // "Cardiología" | "General" | etc.
  id_number: string;            // Professional ID
  phone_number?: string;
  profile_picture_url?: string;
  active_for_ai_booking: boolean;
  created_at: string;
}
```

### 4. Appointments_Table_V2

**Purpose:** Store consultations with SOAP reports

**Keys:**
- **PK:** `patient_id` (String)
- **SK:** `appointment_id` (String)

**GSI:**
- **appointment_id-index:** PK=appointment_id
- **doctor_id-created_at-index:** PK=doctor_id, SK=created_at
- **doctor_id-index:** PK=doctor_id

**Attributes:**
```typescript
{
  patient_id: string;
  appointment_id: string;       // UUID
  doctor_id: string;
  appointment_date: string;     // YYYY-MM-DD
  appointment_time: string;     // HH:MM
  status: string;               // "completed" | "pending" | "cancelled"
  paid: boolean;
  payment_id?: string;

  // Audio processing
  audio_s3_key?: string;
  transcription_text?: string;
  transcription_service?: string; // "openai" | "aws-transcribe"

  // SOAP Report
  soap_subjective?: string;
  soap_objective?: string;
  soap_assessment?: string;
  soap_plan?: string;
  soap_service?: string;        // "openai" | "anthropic"

  created_at: string;
  processing_started_at?: string;
  processing_completed_at?: string;
}
```

### 5. Diagnosis_Table_V2

**Purpose:** Store ICD-10 diagnoses

**Keys:**
- **PK:** `appointment_id` (String)
- **SK:** `patient_id` (String)

**Attributes:**
```typescript
{
  appointment_id: string;
  patient_id: string;
  doctor_id: string;
  diagnoses: Array<{
    icd_code: string;           // "J06.9"
    description: string;        // "Acute upper respiratory infection"
    confidence?: string;        // "high" | "medium" | "low"
  }>;
  diagnosis_service?: string;   // "openai" | "anthropic"
  created_at: string;
}
```

### 6. Vitas_Appointment

**Purpose:** Available appointment slots (with TTL for unpaid)

**Keys:**
- **PK:** `doctor_id` (String)
- **SK:** `appt` (String) - Format: `APPT#{uuid}` or `LOCK#{date}#{slot}`

**GSI:**
- **appointment_id-index:** PK=appointment_id

**Attributes:**
```typescript
{
  doctor_id: string;
  appt: string;                 // "APPT#uuid" or "LOCK#2025-11-06#10:00"
  appointment_id: string;
  patient_id: string;
  appointment_date: string;
  appointment_time: string;
  status: string;               // "pending" | "confirmed" | "cancelled"
  paid: boolean;
  payment_id?: string;
  payment_amount?: number;
  payment_currency?: string;
  reason?: string;              // Consultation reason
  created_via?: string;         // "chatbot" | "web"
  created_at: string;
  ttl?: number;                 // Epoch seconds (15 min for unpaid)
}
```

### 7. Vitas_Schedule_Config

**Purpose:** Doctor schedule configuration

**Keys:**
- **PK:** `doctor_id` (String)
- **SK:** `config` (String) - Fixed value "CONFIG"

**Attributes:**
```typescript
{
  doctor_id: string;
  config: string;               // "CONFIG"
  monday: Array<{ time: string; duration: number }>;
  tuesday: Array<{ time: string; duration: number }>;
  wednesday: Array<{ time: string; duration: number }>;
  thursday: Array<{ time: string; duration: number }>;
  friday: Array<{ time: string; duration: number }>;
  saturday: Array<{ time: string; duration: number }>;
  sunday: Array<{ time: string; duration: number }>;
  timezone: string;             // "America/Sao_Paulo"
  updated_at: string;
}
```

### 8. vitas-otp

**Purpose:** Temporary OTP storage

**Keys:**
- **PK:** `phoneNumber` (String)

**TTL:** `expiresAt` (5 minutes)

**Attributes:**
```typescript
{
  phoneNumber: string;          // "+5511999999999"
  otpCode: string;              // "123456"
  expiresAt: number;            // Epoch seconds
  context: string;              // "chatbot" | "payment"
  attempts: number;             // Rate limiting counter
  createdAt: string;
}
```

### 9. wa_sessions

**Purpose:** WhatsApp chatbot sessions

**Keys:**
- **PK:** `pk` (String) - Format: "whatsapp:+51..."

**TTL:** `expiresAt` (15 min idle)

**Attributes:**
```typescript
{
  pk: string;                   // "whatsapp:+5511999999999"
  threadId: string;             // LangGraph thread ID
  jwt: string;                  // 2-hour JWT token
  patientId: string;
  expiresAt: number;            // Epoch seconds (15 min idle)
  absoluteExpiry: number;       // Epoch seconds (1 hour max)
  lastActiveAt: number;         // Epoch milliseconds
}
```

### 10. vitas-paylinks

**Purpose:** Short URL tokens for payment links

**Keys:**
- **PK:** `pk` (String) - Short code

**TTL:** `expiresAt`

**Attributes:**
```typescript
{
  pk: string;                   // Short code
  appointmentId: string;
  paymentUrl: string;
  expiresAt: number;
  created_at: string;
}
```

---

## Global Secondary Indexes

### Summary Table

| Table | GSI Name | PK | SK | Purpose |
|-------|----------|----|----|---------|
| Patients_Table_V2 | phone_number-index | phone_number | - | OTP verification lookup |
| PatientDoctors_Table_V2 | patient_id-index | patient_id | - | Get patient's doctors |
| Doctors_Table_V2 | email-index | email | - | Doctor login |
| Doctors_Table_V2 | active_for_ai_booking-index | active_for_ai_booking | - | Filter active doctors |
| Appointments_Table_V2 | appointment_id-index | appointment_id | - | Direct appointment lookup |
| Appointments_Table_V2 | doctor_id-created_at-index | doctor_id | created_at | Doctor's appointments sorted by date |
| Appointments_Table_V2 | doctor_id-index | doctor_id | - | All doctor's appointments |
| Vitas_Appointment | appointment_id-index | appointment_id | - | Payment confirmation lookup |

---

## Relationships

```
Doctors_Table_V2 (1) ──┬──< PatientDoctors_Table_V2 (M) >──┬─── Patients_Table_V2 (M)
                       │                                    │
                       │                                    │
                       ├──< Appointments_Table_V2 (M)      │
                       │            ↓ (1:1)                 │
                       │    Diagnosis_Table_V2 (1)          │
                       │                                    │
                       └──< Vitas_Appointment (M)           │
                       │            ↑                       │
                       │            └───────────────────────┘
                       │
                       └──< Vitas_Schedule_Config (1)
```

---

## Access Patterns

### Pattern 1: Doctor Login
```
Query: Doctors_Table_V2 via email-index
WHERE email = "doctor@example.com"
```

### Pattern 2: OTP Verification
```
Query: Patients_Table_V2 via phone_number-index
WHERE phone_number = "+5511999999999"
```

### Pattern 3: Get Doctor's Patients
```
1. Query: PatientDoctors_Table_V2
   WHERE doctor_id = "doctor-123"

2. BatchGetItem: Patients_Table_V2
   WHERE patient_id IN [list from step 1]
```

### Pattern 4: Get Consultation Details
```
1. Query: Appointments_Table_V2 via appointment_id-index
   WHERE appointment_id = "appointment-789"

2. Query: Diagnosis_Table_V2
   WHERE appointment_id = "appointment-789"
```

### Pattern 5: Get Available Slots
```
1. Query: Vitas_Schedule_Config
   WHERE doctor_id = "doctor-123" AND config = "CONFIG"

2. Query: Vitas_Appointment
   WHERE doctor_id = "doctor-123"
   AND appt BEGINS_WITH "APPT#" OR "LOCK#2025-11-06"

3. Calculate: configured_slots - booked_slots
```

### Pattern 6: Payment Confirmation
```
1. Query: Vitas_Appointment via appointment_id-index
   WHERE appointment_id = "appointment-789"

2. Update: Set status="confirmed", paid=true, remove TTL

3. Put: PatientDoctors_Table_V2
   (doctor_id, patient_id) relationship
```

---

**Last Updated:** 2025-11-06
