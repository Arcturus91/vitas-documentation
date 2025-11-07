# Frontend-Backend Integration

## Table of Contents
- [Integration Architecture](#integration-architecture)
- [Authentication Flow](#authentication-flow)
- [Doctor Dashboard Integrations](#doctor-dashboard-integrations)
- [Patient Portal Integrations](#patient-portal-integrations)
- [Real-time Notifications](#real-time-notifications)
- [File Upload Flows](#file-upload-flows)
- [Error Handling](#error-handling)
- [Caching Strategies](#caching-strategies)

## Integration Architecture

### High-Level Frontend-Backend Communication

```
┌──────────────────────────────────────────────────────────────────┐
│                     FRONTEND APPLICATIONS                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌────────────────────┐          ┌────────────────────┐         │
│  │  Doctor Dashboard  │          │  Patient Portal    │         │
│  │   (vitas-client)   │          │ (vitas-patients)   │         │
│  │                    │          │                    │         │
│  │ • Next.js 14       │          │ • Next.js 14       │         │
│  │ • React 18         │          │ • React 18         │         │
│  │ • TypeScript       │          │ • TypeScript       │         │
│  │ • Server-side API  │          │ • Server-side API  │         │
│  └─────────┬──────────┘          └─────────┬──────────┘         │
│            │                               │                      │
└────────────┼───────────────────────────────┼──────────────────────┘
             │                               │
             └───────────┬───────────────────┘
                         │
         ┌───────────────┴────────────────────┐
         │                                     │
         ▼                                     ▼
┌──────────────────┐              ┌──────────────────────┐
│  Next.js API     │              │  Next.js API Routes  │
│  Routes (Proxy)  │              │     (Direct)         │
│                  │              │                      │
│ /api/chat/*      │              │ /api/audio-to-s3/*   │
│ (→ LangGraph)    │              │ /api/processing-*    │
└────────┬─────────┘              └──────────┬───────────┘
         │                                    │
         │                                    │
         └────────────┬───────────────────────┘
                      │
         ┌────────────┴─────────────────────┐
         │                                   │
         ▼                                   ▼
┌────────────────────┐          ┌──────────────────────┐
│  AWS API Gateway   │          │   AWS Lambda         │
│  (Main Stack)      │          │   (Direct Invoke)    │
└──────────┬─────────┘          └──────────┬───────────┘
           │                               │
           ▼                               ▼
    ┌──────────────┐               ┌──────────────┐
    │  Lambda      │               │  S3 Bucket   │
    │  Functions   │               │              │
    └──────┬───────┘               └──────────────┘
           │
           ▼
    ┌──────────────┐
    │  DynamoDB    │
    │  Tables      │
    └──────────────┘
```

### Communication Protocols

| Integration Type | Protocol | Authentication | Use Case |
|-----------------|----------|----------------|----------|
| **API Gateway REST** | HTTPS + JSON | JWT Bearer Token | Doctor/patient CRUD operations |
| **Next.js API Routes** | HTTPS + JSON | Cookie-based JWT | Proxying, SSR, webhooks |
| **LangGraph Streaming** | Server-Sent Events (SSE) | JWT Bearer Token | Real-time AI chat responses |
| **S3 Presigned URLs** | HTTPS PUT | Presigned signature | Direct audio/file uploads |
| **Webhook Callbacks** | HTTPS POST | Twilio signature / Custom headers | Async notifications |

---

## Authentication Flow

### 1. Doctor Login Flow

**Frontend:** `vitas-client/src/app/(auth)/signin/page.tsx`

```typescript
// User submits email + password
const response = await fetch('/api/signin', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ email, password })
});

const data = await response.json();
// Returns: { token: "eyJhbG...", doctor: {...} }
```

**Backend:** `vitas-client/src/app/api/signin/route.ts`

```typescript
import { NextResponse } from 'next/server';
import { cookies } from 'next/headers';

export async function POST(request: Request) {
  const { email, password } = await request.json();

  // Call AWS API Gateway (vitas-main-stack)
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_API_BASE_URL}/auth/signin`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password })
    }
  );

  const data = await response.json();

  if (data.token) {
    // Set HTTP-only cookie
    cookies().set('authToken', data.token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      maxAge: 60 * 60 * 24 // 24 hours
    });
  }

  return NextResponse.json(data);
}
```

**AWS Lambda:** `vitas-main-stack/lambda/vitas-auth/src/auth/auth.ts`

```typescript
export async function loginDoctor(email: string, password: string) {
  // Query Doctors_Table_V2 by email-index
  const doctor = await getDoctorByEmail(email);

  if (!doctor) {
    throw new Error('Doctor not found');
  }

  // Verify password with bcrypt
  const isValid = await bcrypt.compare(password, doctor.password);

  if (!isValid) {
    throw new Error('Invalid credentials');
  }

  // Generate JWT token
  const token = await new jose.SignJWT({
    doctorId: doctor.doctor_id,
    email: doctor.email,
    role: 'doctor'
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .sign(new TextEncoder().encode(process.env.JWT_SECRET));

  return { token, doctor };
}
```

**Data Flow:**
```
[Frontend Form]
    ↓ POST { email, password }
[Next.js API Route: /api/signin]
    ↓ Proxy to AWS
[API Gateway: POST /auth/signin]
    ↓ Invoke Lambda
[VitasAuthFunction Lambda]
    ↓ Query DynamoDB
[Doctors_Table_V2 via email-index]
    ↓ Verify password (bcrypt)
    ↓ Generate JWT
    ← Return { token, doctor }
[Next.js API Route]
    ↓ Set HTTP-only cookie
    ← Return to frontend
[Frontend]
    ↓ Store doctor in UserContext
    ↓ Redirect to dashboard
```

---

### 2. Patient OTP Authentication Flow

**Frontend:** `vitas-patients-front/`

#### Step 1: Request OTP
```typescript
// POST /otp/send
const response = await fetch('/api/otp/send', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    phoneNumber: '+5511999999999',
    patientName: 'João Silva'
  })
});

// Returns: { success: true, message: "OTP sent" }
```

**Backend:** `vitas-auth-otp/lambda-functions/send-otp/send-otp.ts`

```typescript
export const handler = async (event: any) => {
  const { phoneNumber, patientName } = JSON.parse(event.body);

  // Generate 6-digit OTP
  const otpCode = Math.floor(100000 + Math.random() * 900000).toString();

  // Store in DynamoDB (vitas-otp table)
  await dynamodb.put({
    TableName: 'vitas-otp',
    Item: {
      phoneNumber,
      otpCode,
      expiresAt: Date.now() + 300000, // 5 minutes TTL
      context: 'payment',
      attempts: 0,
      createdAt: new Date().toISOString()
    }
  }).promise();

  // Send via Twilio WhatsApp
  const twilioConfig = await getSSMParameter('/vitas-clinic/twilio-config');
  await twilio.messages.create({
    from: twilioConfig.fromNumber,
    to: `whatsapp:${phoneNumber}`,
    body: `Su código de verificación es: ${otpCode}`
  });

  return { statusCode: 200, body: JSON.stringify({ success: true }) };
};
```

#### Step 2: Verify OTP
```typescript
// POST /otp/verify
const response = await fetch('/api/otp/verify', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    phoneNumber: '+5511999999999',
    otpCode: '123456'
  })
});

// Returns: { token: "eyJhbG...", patient: {...} }
```

**Backend:** `vitas-auth-otp/lambda-functions/verify-otp/verify-otp.ts`

```typescript
export const handler = async (event: any) => {
  const { phoneNumber, otpCode } = JSON.parse(event.body);

  // Verify OTP from DynamoDB
  const otpRecord = await dynamodb.get({
    TableName: 'vitas-otp',
    Key: { phoneNumber }
  }).promise();

  if (!otpRecord.Item || otpRecord.Item.otpCode !== otpCode) {
    throw new Error('Invalid OTP');
  }

  if (otpRecord.Item.expiresAt < Date.now()) {
    throw new Error('OTP expired');
  }

  // Create or retrieve patient from Patients_Table_V2
  let patient = await getPatientByPhone(phoneNumber);

  if (!patient) {
    patient = await createPatient({
      patient_id: uuidv4(),
      phone_number: phoneNumber,
      patient_name: patientName,
      status: 'registered_via_whatsapp',
      created_at: new Date().toISOString()
    });
  }

  // Generate JWT (24-hour expiry for payment)
  const token = await new jose.SignJWT({
    patientId: patient.patient_id,
    phoneNumber: patient.phone_number,
    role: 'patient'
  })
    .setProtectedHeader({ alg: 'HS256' })
    .setExpirationTime('24h')
    .sign(new TextEncoder().encode(process.env.JWT_SECRET));

  // Delete OTP record (consumed)
  await dynamodb.delete({
    TableName: 'vitas-otp',
    Key: { phoneNumber }
  }).promise();

  return {
    statusCode: 200,
    body: JSON.stringify({ token, patient })
  };
};
```

**Data Flow:**
```
[Patient Portal Form]
    ↓ POST /otp/send { phoneNumber, patientName }
[send-otp Lambda]
    ↓ Generate 6-digit OTP
    ↓ Store in vitas-otp (TTL: 5 min)
    ↓ Send via Twilio WhatsApp
    ← Success response
[Patient receives WhatsApp OTP]
    ↓ Enter OTP in portal
[Patient Portal]
    ↓ POST /otp/verify { phoneNumber, otpCode }
[verify-otp Lambda]
    ↓ Query vitas-otp table
    ↓ Validate OTP + expiry
    ↓ Create/retrieve patient (Patients_Table_V2)
    ↓ Generate JWT (24h)
    ↓ Delete OTP record
    ← Return { token, patient }
[Patient Portal]
    ↓ Store JWT in cookie
    ↓ Redirect to booking flow
```

---

## Doctor Dashboard Integrations

### 1. Patient List Retrieval

**Frontend:** `vitas-client/src/app/medinotas/page.tsx`

```typescript
import { getPatientsByDoctor } from '@/app/lib/api-connectors/getPatientsByDoctor';

const PatientsPage = () => {
  const [patients, setPatients] = useState([]);
  const { user } = useUser(); // Get doctor_id from context

  useEffect(() => {
    const fetchPatients = async () => {
      const data = await getPatientsByDoctor(user.doctor_id);
      setPatients(data);
    };
    fetchPatients();
  }, [user.doctor_id]);

  // Render patient list...
};
```

**API Connector:** `vitas-client/src/app/lib/api-connectors/getPatientsByDoctor.ts`

```typescript
export async function getPatientsByDoctor(doctorId: string) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_API_BASE_URL}/get-patients-by-doctor?doctorId=${doctorId}`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${getAuthToken()}`, // Get from cookie
        'Content-Type': 'application/json'
      },
      cache: 'no-store' // Disable Next.js caching
    }
  );

  if (!response.ok) {
    throw new Error('Failed to fetch patients');
  }

  return response.json();
}
```

**Next.js API Route:** `vitas-client/src/app/api/get-patients-by-doctor/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { DynamoDBClient, QueryCommand } from '@aws-sdk/client-dynamodb';
import { unmarshall } from '@aws-sdk/util-dynamodb';

const dynamodb = new DynamoDBClient({ region: process.env.AWS_REGION });

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const doctorId = searchParams.get('doctorId');

  if (!doctorId) {
    return NextResponse.json({ error: 'doctorId required' }, { status: 400 });
  }

  // Query PatientDoctors_Table_V2
  const command = new QueryCommand({
    TableName: 'PatientDoctors_Table_V2',
    KeyConditionExpression: 'doctor_id = :doctorId',
    ExpressionAttributeValues: {
      ':doctorId': { S: doctorId }
    }
  });

  const response = await dynamodb.send(command);
  const patientDoctorRecords = response.Items?.map(item => unmarshall(item)) || [];

  // Extract patient IDs
  const patientIds = patientDoctorRecords.map(record => record.patient_id);

  // Batch get patients from Patients_Table_V2
  const patients = await batchGetPatients(patientIds);

  return NextResponse.json(patients);
}

async function batchGetPatients(patientIds: string[]) {
  // Implementation of BatchGetItem for Patients_Table_V2
  // Returns array of patient objects
}
```

**Data Flow:**
```
[Doctor Dashboard]
    ↓ Call getPatientsByDoctor(doctorId)
[Next.js API Route: /api/get-patients-by-doctor]
    ↓ Query DynamoDB directly (server-side AWS SDK)
[PatientDoctors_Table_V2]
    ↓ Get patient IDs for doctor
[Patients_Table_V2]
    ↓ Batch get patient details
    ← Return patient array
[Frontend]
    ↓ Update state
    ↓ Render patient list
```

---

### 2. Audio Upload and Processing

**Frontend:** `vitas-client/src/app/medinotas/voice-record/`

```typescript
const uploadAudioToS3 = async (audioBlob: Blob, metadata: any) => {
  // Step 1: Get presigned URL from backend
  const response = await fetch('/api/audio-to-s3-presigned-url', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      fileName: `${metadata.appointmentId}.mp3`,
      doctorId: metadata.doctorId,
      patientId: metadata.patientId,
      appointmentId: metadata.appointmentId
    })
  });

  const { presignedUrl, s3Key } = await response.json();

  // Step 2: Upload directly to S3 using presigned URL
  await fetch(presignedUrl, {
    method: 'PUT',
    body: audioBlob,
    headers: {
      'Content-Type': 'audio/mpeg'
    }
  });

  // Step 3: Start polling for processing notifications
  startPolling(); // From useProcessingNotifications hook

  return s3Key;
};
```

**Backend:** `vitas-client/src/app/api/audio-to-s3-presigned-url/route.ts`

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3Client = new S3Client({ region: process.env.AWS_REGION });

export async function POST(request: Request) {
  const { fileName, doctorId, patientId, appointmentId } = await request.json();

  const s3Key = `audios/doctor-${doctorId}/patient-${patientId}/${fileName}`;

  // Create presigned URL for upload (expires in 15 minutes)
  const command = new PutObjectCommand({
    Bucket: process.env.MEDIA_BUCKET,
    Key: s3Key,
    ContentType: 'audio/mpeg',
    Metadata: {
      'doctor-id': doctorId,
      'patient-id': patientId,
      'appointment-id': appointmentId,
      'upload-timestamp': new Date().toISOString()
    }
  });

  const presignedUrl = await getSignedUrl(s3Client, command, { expiresIn: 900 });

  return NextResponse.json({ presignedUrl, s3Key });
}
```

**Data Flow:**
```
[Doctor Dashboard: Voice Recorder]
    ↓ Record audio (MediaRecorder API)
    ↓ Convert to MP3 (lamejs)
    ↓ POST /api/audio-to-s3-presigned-url
[Next.js API Route]
    ↓ Generate S3 presigned URL (AWS SDK)
    ← Return { presignedUrl, s3Key }
[Frontend]
    ↓ PUT audio blob directly to S3 presigned URL
[S3: vitas-media-processing bucket]
    ↓ ObjectCreated event
[Workflow Trigger Lambda]
    ↓ Start Transcription Step Functions workflow
[Step Functions: Transcription Workflow]
    ↓ Invoke Transcription Lambda
    ↓ → Analysis Workflow (SOAP + ICD)
    ↓ Success → Failure Notification Lambda
[Failure Notification Lambda]
    ↓ POST /api/processing-webhook
[Frontend: Polling hook]
    ↓ GET /api/processing-webhook?doctorId={id}
    ← Receive notification
[Frontend]
    ↓ Display snackbar notification
    ↓ Invalidate appointments cache
```

---

### 3. SOAP Report Retrieval

**Frontend:** `vitas-client/src/app/medinotas/appointment/[id]/page.tsx`

```typescript
import { getConsultationDetails } from '@/app/lib/api-connectors/getConsultationDetails';

const AppointmentPage = ({ params }: { params: { id: string } }) => {
  const [consultation, setConsultation] = useState(null);

  useEffect(() => {
    const fetchData = async () => {
      const data = await getConsultationDetails(params.id);
      setConsultation(data);
    };
    fetchData();
  }, [params.id]);

  return (
    <div>
      <h1>Consulta: {consultation?.appointmentId}</h1>
      <section>
        <h2>SOAP Report</h2>
        <div>
          <h3>Subjetivo</h3>
          <p>{consultation?.soapReport?.subjective}</p>

          <h3>Objetivo</h3>
          <p>{consultation?.soapReport?.objective}</p>

          <h3>Evaluación</h3>
          <p>{consultation?.soapReport?.assessment}</p>

          <h3>Plan</h3>
          <p>{consultation?.soapReport?.plan}</p>
        </div>
      </section>

      <section>
        <h2>Diagnósticos ICD-10</h2>
        <ul>
          {consultation?.icdDiagnoses?.map(dx => (
            <li key={dx.code}>{dx.code}: {dx.description}</li>
          ))}
        </ul>
      </section>
    </div>
  );
};
```

**API Connector:** `vitas-client/src/app/lib/api-connectors/getConsultationDetails.ts`

```typescript
export async function getConsultationDetails(appointmentId: string) {
  const response = await fetch(
    `/api/get-consultation-details?appointmentId=${appointmentId}`,
    { cache: 'no-store' }
  );

  if (!response.ok) {
    throw new Error('Failed to fetch consultation');
  }

  return response.json();
}
```

**Next.js API Route:** `vitas-client/src/app/api/get-consultation-details/route.ts`

```typescript
export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url);
  const appointmentId = searchParams.get('appointmentId');

  // Query Appointments_Table_V2 via appointment_id-index GSI
  const appointment = await getAppointmentById(appointmentId);

  if (!appointment) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 });
  }

  // Query Diagnosis_Table_V2
  const diagnoses = await getDiagnosesByAppointmentId(appointmentId);

  return NextResponse.json({
    ...appointment,
    icdDiagnoses: diagnoses
  });
}
```

**Data Flow:**
```
[Doctor Dashboard: Appointment Detail Page]
    ↓ GET /api/get-consultation-details?appointmentId=...
[Next.js API Route]
    ↓ Query DynamoDB (server-side)
[Appointments_Table_V2 via appointment_id-index]
    ← Get appointment with soapReport field
[Diagnosis_Table_V2]
    ← Get ICD diagnoses
[Next.js API Route]
    ← Merge data and return
[Frontend]
    ↓ Update state
    ↓ Render SOAP report + diagnoses
```

---

## Patient Portal Integrations

### 1. Appointment Booking Flow

**Frontend:** `vitas-patients-front/src/app/book/page.tsx`

```typescript
const BookAppointmentPage = () => {
  const [doctors, setDoctors] = useState([]);
  const [selectedDoctor, setSelectedDoctor] = useState(null);
  const [availableSlots, setAvailableSlots] = useState([]);

  // Step 1: Get available doctors
  useEffect(() => {
    const fetchDoctors = async () => {
      const response = await fetch('/api/doctors?active=true');
      setDoctors(await response.json());
    };
    fetchDoctors();
  }, []);

  // Step 2: Get available slots for selected doctor
  const handleDoctorSelect = async (doctorId: string) => {
    setSelectedDoctor(doctorId);

    const response = await fetch(
      `/api/appointments/slots?doctorId=${doctorId}&date=${selectedDate}`
    );
    setAvailableSlots(await response.json());
  };

  // Step 3: Create appointment (pending payment)
  const handleBooking = async (slot: any) => {
    const response = await fetch('/api/appointments/book', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${authToken}` // From OTP verification
      },
      body: JSON.stringify({
        doctorId: selectedDoctor,
        patientId: patient.patient_id,
        date: selectedDate,
        timeSlot: slot.time,
        appointmentId: uuidv4()
      })
    });

    const { appointmentId, paymentUrl } = await response.json();

    // Step 4: Redirect to MercadoPago
    window.location.href = paymentUrl;
  };

  // Render UI...
};
```

**Backend:** `vitas-auth-otp/` (hypothetical endpoint, may be in main-stack)

```typescript
// POST /appointments/book
export const handler = async (event: any) => {
  const { doctorId, patientId, date, timeSlot, appointmentId } = JSON.parse(event.body);

  // Verify JWT from Authorization header
  const token = event.headers.Authorization.replace('Bearer ', '');
  const decoded = await jose.jwtVerify(token, secret);

  if (decoded.payload.patientId !== patientId) {
    throw new Error('Unauthorized');
  }

  // Create appointment in Vitas_Appointment table
  await dynamodb.put({
    TableName: 'Vitas_Appointment',
    Item: {
      doctor_id: doctorId,
      appt: `APPT#${appointmentId}`,
      appointment_id: appointmentId,
      patient_id: patientId,
      appointment_date: date,
      appointment_time: timeSlot,
      status: 'pending',
      paid: false,
      created_at: new Date().toISOString(),
      ttl: Math.floor(Date.now() / 1000) + 900 // 15 minutes TTL
    }
  }).promise();

  // Generate MercadoPago payment link
  const mercadopago = new MercadoPagoAPI(process.env.MP_ACCESS_TOKEN);
  const payment = await mercadopago.preferences.create({
    items: [{
      title: `Consulta con Dr. ${doctorName}`,
      quantity: 1,
      unit_price: 50.00
    }],
    back_urls: {
      success: `${process.env.FRONTEND_URL}/booking/success`,
      failure: `${process.env.FRONTEND_URL}/booking/failure`
    },
    notification_url: `${process.env.API_URL}/confirm-pay`,
    external_reference: appointmentId,
    metadata: { appointmentId, patientId, doctorId }
  });

  return {
    statusCode: 200,
    body: JSON.stringify({
      appointmentId,
      paymentUrl: payment.init_point
    })
  };
};
```

**Data Flow:**
```
[Patient Portal: Booking Page]
    ↓ Select doctor, date, time
    ↓ POST /appointments/book (JWT in header)
[Lambda: Book Appointment]
    ↓ Verify JWT
    ↓ Create appointment (Vitas_Appointment, status=pending, TTL=15min)
    ↓ Generate MercadoPago payment link
    ← Return { appointmentId, paymentUrl }
[Patient Portal]
    ↓ Redirect to MercadoPago
[Patient completes payment]
[MercadoPago]
    ↓ Webhook POST /confirm-pay
[Lambda: confirm-pay]
    ↓ Verify payment status via MercadoPago API
    ↓ Update appointment (status=confirmed, paid=true, remove TTL)
    ↓ Create PatientDoctors_Table_V2 entry
    ↓ Send WhatsApp confirmation
[Patient receives WhatsApp message]
```

---

## Real-time Notifications

### Processing Notifications (Doctor Dashboard)

**Frontend Hook:** `vitas-client/src/app/hooks/useProcessingNotifications.ts`

```typescript
export function useProcessingNotifications(doctorId: string | null) {
  const [isPolling, setIsPolling] = useState(false);
  const intervalRef = useRef<NodeJS.Timeout | null>(null);

  const pollForNotifications = useCallback(async () => {
    if (!doctorId) return;

    try {
      const response = await fetch(
        `${process.env.NEXT_PUBLIC_API_BASE_URL}/processing-webhook?doctorId=${doctorId}`,
        { cache: 'no-store' }
      );

      const data = await response.json();

      if (data.notifications.length > 0) {
        // Display each notification
        data.notifications.forEach((notification: any) => {
          const message = notification.type === 'success'
            ? `Procesamiento completado para la cita de ${patientName}`
            : `Error en procesamiento: ${notification.error}`;

          enqueueSnackbar(message, {
            variant: notification.type,
            autoHideDuration: notification.type === 'success' ? 5000 : 10000
          });
        });

        // Invalidate cache
        invalidateCache.appointments();

        // Stop polling after receiving notifications
        stopPolling();
      }
    } catch (error) {
      console.error('Polling error:', error);
      stopPolling();
    }
  }, [doctorId]);

  const startPolling = useCallback(() => {
    if (isPolling || !doctorId) return;

    setIsPolling(true);

    // Poll immediately
    pollForNotifications();

    // Then poll every 10 seconds
    intervalRef.current = setInterval(pollForNotifications, 10000);

    // Auto-stop after 10 minutes
    setTimeout(() => stopPolling(), 10 * 60 * 1000);
  }, [doctorId, pollForNotifications]);

  return { startPolling, stopPolling, isPolling };
}
```

**Backend Webhook:** `vitas-client/src/app/api/processing-webhook/route.ts`

```typescript
// In-memory store for recent notifications (TTL: 1 minute)
const recentNotifications = new Map<string, StoredNotification[]>();

// POST endpoint (called by Lambda)
export async function POST(request: NextRequest): Promise<NextResponse> {
  const body = await request.json();

  // Validate webhook source
  const source = request.headers.get('x-source');
  if (source !== 'aws-lambda') {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { type, service, metadata } = body;
  const doctorId = metadata.doctorId;

  if (doctorId) {
    const notification = {
      id: `${metadata.appointmentId}-${service}-${Date.now()}`,
      type: type === 'success' ? 'success' : 'error',
      message: type === 'success'
        ? `Procesamiento completado`
        : `Error: ${body.error}`,
      appointmentId: metadata.appointmentId,
      service,
      timestamp: body.timestamp,
      metadata,
      doctorId,
      expiresAt: Date.now() + 60000 // 1 minute TTL
    };

    // Store notification
    const existing = recentNotifications.get(doctorId) || [];
    existing.push(notification);
    recentNotifications.set(doctorId, existing);
  }

  return NextResponse.json({ success: true });
}

// GET endpoint (called by frontend polling)
export async function GET(request: NextRequest): Promise<NextResponse> {
  const { searchParams } = new URL(request.url);
  const doctorId = searchParams.get('doctorId');

  if (!doctorId) {
    return NextResponse.json({ status: 'ok' });
  }

  // Get notifications for this doctor
  const notifications = recentNotifications.get(doctorId) || [];

  // Remove doctorId before sending to frontend
  const frontendNotifications = notifications.map(
    ({ doctorId: _, expiresAt: __, ...notification }) => notification
  );

  // Clear notifications after retrieval
  if (notifications.length > 0) {
    recentNotifications.delete(doctorId);
  }

  return NextResponse.json({
    notifications: frontendNotifications,
    count: frontendNotifications.length,
    timestamp: new Date().toISOString()
  });
}
```

**Data Flow:**
```
[Audio Processing Completes]
    ↓ Failure Notification Lambda
[Lambda: failure-notification]
    ↓ POST /api/processing-webhook
    ↓ Headers: x-source=aws-lambda
    ↓ Body: { type, service, metadata: { doctorId, appointmentId } }
[Next.js Webhook Route]
    ↓ Validate source
    ↓ Store in recentNotifications Map (1 min TTL)
    ← Return 200 OK
[Frontend: useProcessingNotifications Hook]
    ↓ Polling every 10 seconds
    ↓ GET /api/processing-webhook?doctorId={id}
[Next.js Webhook Route]
    ↓ Retrieve notifications from Map
    ↓ Clear notifications for doctor
    ← Return { notifications: [...] }
[Frontend Hook]
    ↓ Display snackbar (notistack)
    ↓ Invalidate appointments cache
    ↓ Stop polling
```

---

## File Upload Flows

### Audio File Upload (S3 Presigned URL)

**Flow Summary:**
1. Frontend requests presigned URL from backend
2. Backend generates presigned URL with S3 metadata
3. Frontend uploads file directly to S3 using presigned URL
4. S3 ObjectCreated event triggers processing workflow

**Benefits:**
- No file passes through backend (reduces Lambda payload limits)
- Faster uploads (direct to S3)
- Secure (presigned URLs expire after 15 minutes)
- Metadata embedded in S3 object

**Implementation:** See [Audio Upload and Processing](#2-audio-upload-and-processing) above.

---

## Error Handling

### Frontend Error Handling Patterns

**1. API Error Responses**
```typescript
// Generic API call wrapper
async function apiCall(url: string, options: RequestInit) {
  try {
    const response = await fetch(url, options);

    if (!response.ok) {
      const errorData = await response.json();
      throw new Error(errorData.message || 'API call failed');
    }

    return response.json();
  } catch (error) {
    console.error('API Error:', error);

    // Show user-friendly error
    enqueueSnackbar(
      error instanceof Error ? error.message : 'Error de red',
      { variant: 'error' }
    );

    throw error;
  }
}
```

**2. Authentication Errors**
```typescript
// Middleware to check auth status
async function authenticatedFetch(url: string, options: RequestInit) {
  const response = await fetch(url, options);

  if (response.status === 401) {
    // Token expired or invalid
    enqueueSnackbar('Sesión expirada. Por favor, inicie sesión nuevamente.', {
      variant: 'warning'
    });

    // Redirect to login
    router.push('/signin');

    throw new Error('Unauthorized');
  }

  return response;
}
```

**3. Processing Errors**
```typescript
// In useProcessingNotifications hook
if (notification.type === 'error') {
  const errorMessage = notification.metadata?.retryScheduled
    ? `Error en ${serviceName}, reintentaremos en ${notification.metadata.nextRetryIn}`
    : `Error en procesamiento: ${notification.error}`;

  enqueueSnackbar(errorMessage, {
    variant: 'error',
    autoHideDuration: 10000 // Longer duration for errors
  });
}
```

---

## Caching Strategies

### 1. Client-side Caching (Browser)

**Implementation:** `vitas-client/src/app/utils/cache.ts`

```typescript
const CACHE_EXPIRY = {
  PATIENTS_LIST: 5 * 60 * 1000, // 5 minutes
  APPOINTMENTS: 2 * 60 * 1000,   // 2 minutes
  CONSULTATION: 10 * 60 * 1000   // 10 minutes
};

export function getFromCache<T>(key: string): T | null {
  const cached = localStorage.getItem(key);

  if (!cached) return null;

  const { data, timestamp, expiry } = JSON.parse(cached);

  if (Date.now() - timestamp > expiry) {
    localStorage.removeItem(key);
    return null;
  }

  return data as T;
}

export function setInCache<T>(key: string, data: T, expiry: number): void {
  localStorage.setItem(key, JSON.stringify({
    data,
    timestamp: Date.now(),
    expiry
  }));
}

export function invalidateCache(key: string): void {
  localStorage.removeItem(key);
}
```

**Usage:**
```typescript
// Get patients with caching
const getPatients = async (doctorId: string) => {
  const cached = getFromCache<Patient[]>(CACHE_KEYS.PATIENTS_LIST);

  if (cached) {
    return cached;
  }

  const patients = await fetchPatientsFromAPI(doctorId);
  setInCache(CACHE_KEYS.PATIENTS_LIST, patients, CACHE_EXPIRY.PATIENTS_LIST);

  return patients;
};
```

### 2. Server-side Caching (Next.js)

**Fetch with revalidation:**
```typescript
// Revalidate every 60 seconds
const response = await fetch(url, { next: { revalidate: 60 } });
```

**Opt out of caching:**
```typescript
// Always fetch fresh data
const response = await fetch(url, { cache: 'no-store' });
```

### 3. Cache Invalidation

**After data mutation:**
```typescript
// After uploading audio
await uploadAudio(audioBlob);
invalidateCache.appointments(); // Force refresh on next load

// After updating patient
await updatePatient(patientData);
invalidateCache.patientslist(); // Force refresh
```

---

## Summary

This document covered the complete integration between frontend applications (Next.js) and backend services (AWS Lambda, DynamoDB, S3). Key patterns include:

1. **JWT-based authentication** with HTTP-only cookies
2. **API Gateway proxying** through Next.js API routes
3. **Direct S3 uploads** using presigned URLs
4. **Real-time notifications** via polling webhook endpoint
5. **Error handling** with user-friendly messages
6. **Client-side caching** with automatic invalidation

**Related Documentation:**
- [AI Chatbot Integration](04-AI_CHATBOT_INTEGRATION.md) - LangGraph streaming integration
- [Processing Pipeline Integration](05-PROCESSING_PIPELINE_INTEGRATION.md) - Audio processing workflow
- [API Contracts](06-API_CONTRACTS.md) - Complete API reference

**Last Updated:** 2025-11-06
