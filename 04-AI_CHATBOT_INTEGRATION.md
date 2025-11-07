# AI Chatbot Integration

## Table of Contents
- [Overview](#overview)
- [LangGraph Architecture](#langgraph-architecture)
- [WhatsApp Integration Flow](#whatsapp-integration-flow)
- [Frontend Chatbot Integration](#frontend-chatbot-integration)
- [Agent Tools & Capabilities](#agent-tools--capabilities)
- [Session Management](#session-management)
- [Streaming Implementation](#streaming-implementation)
- [Error Handling & Fallbacks](#error-handling--fallbacks)

## Overview

The VITAS Clinic chatbot is an AI-powered conversational assistant built with **LangGraph** that provides two distinct experiences:

1. **WhatsApp Chatbot** - Patient-facing, appointment booking via messaging
2. **Web Chatbot** - Doctor-facing, embedded in dashboard sidebar

Both use the same LangGraph backend but with different **role-based agents**:
- **Patient Agent** - Appointment booking, doctor search, profile updates
- **Doctor Agent** - Patient records access, schedule viewing, medical information search

### Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **AI Framework** | LangGraph 0.6.0 | Stateful agent orchestration |
| **LLM** | OpenAI GPT-4o-mini | Language model (temperature=0) |
| **Pattern** | ReAct (Reasoning + Acting) | Agent decision-making framework |
| **Messaging** | Twilio WhatsApp Business API | Message delivery |
| **Frontend** | @langchain/langgraph-sdk/react | Streaming chat UI |
| **Backend** | AWS Lambda (Node.js 20.x) | WhatsApp webhook gateway |
| **Deployment** | LangGraph Cloud | Managed hosting |

---

## LangGraph Architecture

### StateGraph Definition

**File:** `vitas-chatbot/src/react_agent/graph.py`

```python
from langgraph.graph import StateGraph, START, END
from react_agent.state import State
from react_agent.doctor_agent import doctor_agent_node
from react_agent.patient_agent import patient_agent_node, load_patient, check_information, upsert_patient_profile

# Create state graph
graph_builder = StateGraph(State)

# Add nodes
graph_builder.add_node("doctor_agent", doctor_agent_node)
graph_builder.add_node("patient_agent", patient_agent_node)
graph_builder.add_node("load_patient", load_patient)
graph_builder.add_node("upsert_patient_profile", upsert_patient_profile)

# Define conditional routing
def role_condition(state: State):
    """Route based on user role (doctor vs patient)"""
    role = state.get("role", "patient")
    return "doctor_agent" if role == "doctor" else "load_patient"

def check_information_condition(state: State):
    """Check if patient profile is complete"""
    patient = state.get("patient")

    if not patient:
        return "upsert_patient_profile"

    # Check required fields
    required_fields = ["patient_name", "patient_email", "patient_age", "patient_dni"]
    missing_fields = [f for f in required_fields if not patient.get(f)]

    if missing_fields:
        return "upsert_patient_profile"

    return "patient_agent"

# Add edges
graph_builder.add_conditional_edges(START, role_condition)
graph_builder.add_edge("doctor_agent", END)
graph_builder.add_edge("load_patient", "check_information")
graph_builder.add_conditional_edges("check_information", check_information_condition)
graph_builder.add_edge("upsert_patient_profile", "patient_agent")
graph_builder.add_edge("patient_agent", END)

# Compile graph
graph = graph_builder.compile()
```

### Graph Visualization

```
                    START
                      │
                      ▼
            ┌─────────────────┐
            │ role_condition  │
            └────────┬─────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
   ┌──────────────┐      ┌──────────────┐
   │doctor_agent  │      │load_patient  │
   │              │      │              │
   │• Get patients│      │• Load from   │
   │• View records│      │  DynamoDB    │
   │• Search info │      │• Extract JWT │
   └──────┬───────┘      └──────┬───────┘
          │                     │
          ▼                     ▼
        END           ┌──────────────────┐
                      │check_information│
                      └────────┬─────────┘
                               │
               ┌───────────────┴────────────────┐
               │                                 │
               ▼                                 ▼
      ┌────────────────────┐         ┌──────────────────┐
      │upsert_patient_     │         │patient_agent     │
      │    profile         │         │                  │
      │                    │         │• Search doctors  │
      │• Collect missing   │         │• Find slots      │
      │  patient data      │         │• Book appointment│
      │• Update DB         │         │• Update profile  │
      └────────┬───────────┘         └──────────┬───────┘
               │                                 │
               ▼                                 │
        ┌──────────────┐                       │
        │patient_agent │◄──────────────────────┘
        └──────┬───────┘
               │
               ▼
              END
```

### Agent Nodes

#### 1. Doctor Agent

**File:** `vitas-chatbot/src/react_agent/doctor_agent/node.py`

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent

# Define tools for doctor
tools = [
    get_patients_by_doctor,
    get_scheduled_consultations_by_doctor,
    get_consultation_details,
    search_medical_information
]

# Create ReAct agent
doctor_agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini", temperature=0),
    tools=tools,
    state_modifier="""You are a medical assistant helping doctors access patient information.
    You have access to:
    - Patient lists
    - Consultation records with SOAP reports and ICD diagnoses
    - Schedule information
    - Medical knowledge database

    Be professional, concise, and accurate in your responses."""
)

def doctor_agent_node(state: State):
    """Execute doctor agent"""
    result = doctor_agent.invoke(state)
    return {"messages": result["messages"]}
```

#### 2. Patient Agent

**File:** `vitas-chatbot/src/react_agent/patient_agent/node.py`

```python
# Define tools for patient
tools = [
    get_available_doctors,
    find_free_slots_for_doctor,
    schedule_patient_appointment,
    upsert_patient_profile
]

# Create ReAct agent
patient_agent = create_react_agent(
    model=ChatOpenAI(model="gpt-4o-mini", temperature=0),
    tools=tools,
    state_modifier="""You are a helpful assistant for booking medical appointments.
    You can:
    - Search for available doctors by specialty
    - Find available appointment slots
    - Book appointments
    - Update patient profile information

    Always be friendly, clear, and guide the user through the booking process step by step.
    Speak in Spanish (Latin American)."""
)

def patient_agent_node(state: State):
    """Execute patient agent"""
    result = patient_agent.invoke(state)
    return {"messages": result["messages"]}
```

---

## WhatsApp Integration Flow

### Complete Message Flow

```
Patient sends WhatsApp message
    ↓
[Twilio] Receives message
    ↓
[Twilio] POST webhook to /chat-webhook
    ├─ Headers: X-Twilio-Signature
    ├─ Body: form-urlencoded (From, Body, MessageSid)
    ↓
[Lambda: chat-integration]
    ├─ Validate Twilio signature
    ├─ Parse phone number: whatsapp:+51999999999
    ├─ Load/create session (wa_sessions table)
    │  ├─ Check session expiry (15 min idle, 1h absolute)
    │  ├─ Rotate JWT if <5 minutes remaining
    │  └─ Renew session TTL
    ├─ Auto-create patient in Patients_Table_V2
    │  └─ Set status: "registered_via_chatbot"
    ├─ Call LangGraph API
    │  ├─ POST /threads/{threadId}/runs
    │  ├─ Headers:
    │  │  ├─ Authorization: Bearer {jwt}
    │  │  ├─ assistant_id: {LANGSMITH_ASSISTANT_ID}
    │  │  └─ webhook: {LANGGRAPH_CALLBACK_URL}
    │  └─ Body:
    │     └─ { messages: [{ role: "user", content: "..." }], source: "patient" }
    └─ Return empty TwiML (200 OK)
    ↓
[LangGraph Cloud] Receives request
    ├─ Authenticate JWT (src/security/auth.py)
    ├─ Create/load thread
    ├─ Add message to thread
    ├─ Execute StateGraph
    │  ├─ START → role_condition
    │  ├─ Route to patient_agent (based on JWT role)
    │  ├─ load_patient (query DynamoDB)
    │  ├─ check_information
    │  ├─ patient_agent executes
    │  │  ├─ Reasoning: "User wants to book appointment"
    │  │  ├─ Tool call: get_available_doctors(specialty="general")
    │  │  │  └─ Query Doctors_Table_V2 via Python SDK
    │  │  ├─ Tool call: find_free_slots_for_doctor(doctor_id="123", date="2025-11-10")
    │  │  │  └─ Query Vitas_Appointment, Vitas_Schedule_Config
    │  │  └─ Generate response
    │  └─ END
    └─ Async webhook callback
    ↓
[LangGraph] POST /agent-webhook
    ├─ Query params: ?from=whatsapp:+51... &thread=thread-123
    ├─ Body: { messages: [...], thread_id: "...", status: "done" }
    ↓
[Lambda: agent-webhook]
    ├─ Extract latest AI message
    ├─ Convert markdown to WhatsApp format
    │  ├─ **bold** → *bold*
    │  ├─ __italic__ → _italic_
    │  ├─ [link](url) → text url
    │  └─ Remove headers (###)
    ├─ Send via Twilio WhatsApp API
    │  └─ POST to Twilio with message body
    └─ Return 200 OK (prevent retries)
    ↓
[Twilio] Sends WhatsApp message
    ↓
Patient receives response
```

### Code Implementation

#### Step 1: Twilio Webhook Handler

**File:** `vitas-auth-otp/lambda-functions/chat-integration/chat-integration.ts`

```typescript
import twilio from 'twilio';
import { DynamoDBClient, GetItemCommand, PutItemCommand } from '@aws-sdk/client-dynamodb';
import { SSMClient, GetParameterCommand } from '@aws-sdk/client-ssm';
import * as jose from 'jose';

const dynamodb = new DynamoDBClient({ region: process.env.AWS_REGION });
const ssm = new SSMClient({ region: process.env.AWS_REGION });

export const handler = async (event: any) => {
  // Step 1: Validate Twilio signature
  const twilioSignature = event.headers['X-Twilio-Signature'] || event.headers['x-twilio-signature'];
  const twilioConfig = await getTwilioConfig();

  const isValid = twilio.validateRequest(
    twilioConfig.authToken,
    twilioSignature,
    `https://${event.requestContext.domainName}${event.rawPath}`,
    event.body // form-urlencoded params
  );

  if (!isValid) {
    return { statusCode: 403, body: 'Invalid signature' };
  }

  // Step 2: Parse incoming message
  const params = new URLSearchParams(event.body);
  const from = params.get('From'); // whatsapp:+51999999999
  const messageBody = params.get('Body');
  const messageSid = params.get('MessageSid');

  // Step 3: Load or create session
  const session = await getOrCreateSession(from);

  // Check if JWT needs rotation (< 5 minutes remaining)
  const decoded = await jose.jwtVerify(session.jwt, jwtSecret);
  const exp = decoded.payload.exp! * 1000;
  const timeRemaining = exp - Date.now();

  let jwt = session.jwt;

  if (timeRemaining < 5 * 60 * 1000) {
    // Rotate JWT
    jwt = await generateNewJWT(session.patientId);
    session.jwt = jwt;
  }

  // Step 4: Renew session expiry
  const now = Date.now();
  const idleTTL = now + 15 * 60 * 1000; // 15 minutes idle
  const absoluteTTL = session.absoluteExpiry || (now + 60 * 60 * 1000); // 1 hour absolute

  await dynamodb.send(new PutItemCommand({
    TableName: 'wa_sessions',
    Item: {
      pk: { S: from },
      threadId: { S: session.threadId },
      jwt: { S: jwt },
      expiresAt: { N: Math.floor(Math.min(idleTTL, absoluteTTL) / 1000).toString() },
      absoluteExpiry: { N: Math.floor(absoluteTTL / 1000).toString() },
      lastActiveAt: { N: now.toString() },
      patientId: { S: session.patientId }
    }
  }));

  // Step 5: Auto-create patient if doesn't exist
  await createPatientIfNotExists(from, session.patientId);

  // Step 6: Call LangGraph API
  const langGraphUrl = process.env.LANGGRAPH_URL;
  const assistantId = process.env.LANGSMITH_ASSISTANT_ID;
  const webhookUrl = `${process.env.LANGGRAPH_CALLBACK_URL}?from=${encodeURIComponent(from)}&thread=${session.threadId}`;

  try {
    await fetch(`${langGraphUrl}/threads/${session.threadId}/runs`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${jwt}`,
        'assistant_id': assistantId,
        'webhook': webhookUrl
      },
      body: JSON.stringify({
        messages: [
          { role: 'user', content: messageBody }
        ],
        source: 'patient'
      })
    });
  } catch (error) {
    console.error('LangGraph API error:', error);
    // Return empty TwiML to avoid Twilio retries
  }

  // Return empty TwiML (no immediate response)
  return {
    statusCode: 200,
    headers: { 'Content-Type': 'application/xml' },
    body: '<?xml version="1.0" encoding="UTF-8"?><Response></Response>'
  };
};

async function getOrCreateSession(from: string) {
  const result = await dynamodb.send(new GetItemCommand({
    TableName: 'wa_sessions',
    Key: { pk: { S: from } }
  }));

  if (result.Item) {
    return {
      threadId: result.Item.threadId.S!,
      jwt: result.Item.jwt.S!,
      patientId: result.Item.patientId.S!,
      absoluteExpiry: parseInt(result.Item.absoluteExpiry?.N || '0') * 1000
    };
  }

  // Create new session
  const patientId = `patient-${Date.now()}`;
  const threadId = `thread-${Date.now()}`;
  const jwt = await generateNewJWT(patientId);

  return { threadId, jwt, patientId, absoluteExpiry: null };
}
```

#### Step 2: LangGraph Callback Handler

**File:** `vitas-auth-otp/lambda-functions/agent-webhook/agent-webhook.ts`

```typescript
import { SSMClient, GetParameterCommand } from '@aws-sdk/client-ssm';

const ssm = new SSMClient({ region: process.env.AWS_REGION });

export const handler = async (event: any) => {
  const { from, thread } = event.queryStringParameters;
  const payload = JSON.parse(event.body);

  // Extract latest AI message
  const messages = payload.messages || [];
  const latestMessage = messages[messages.length - 1];

  if (!latestMessage || latestMessage.role !== 'assistant') {
    return { statusCode: 200, body: 'No assistant message' };
  }

  let messageBody = latestMessage.content;

  // Convert markdown to WhatsApp format
  messageBody = convertMarkdownToWhatsApp(messageBody);

  // Send via Twilio
  const twilioConfig = await getTwilioConfig();
  const twilioClient = require('twilio')(twilioConfig.accountSid, twilioConfig.authToken);

  try {
    await twilioClient.messages.create({
      from: twilioConfig.fromNumber,
      to: from, // whatsapp:+51999999999
      body: messageBody
    });
  } catch (error) {
    console.error('Twilio send error:', error);
  }

  // Always return 200 to prevent LangGraph retries
  return { statusCode: 200, body: 'OK' };
};

function convertMarkdownToWhatsApp(text: string): string {
  // Bold: **text** → *text*
  text = text.replace(/\*\*(.*?)\*\*/g, '*$1*');

  // Italic: __text__ → _text_
  text = text.replace(/__(.*?)__/g, '_$1_');

  // Remove headers: ### Title → Title
  text = text.replace(/^#+\s+(.+)$/gm, '$1');

  // Links: [text](url) → text url
  text = text.replace(/\[([^\]]+)\]\(([^)]+)\)/g, '$1 $2');

  return text.trim();
}
```

---

## Frontend Chatbot Integration

### Doctor Dashboard Chatbot Component

**File:** `vitas-client/src/app/medinotas/components/Chatbot.tsx`

```typescript
import { useStream } from '@langchain/langgraph-sdk/react';
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import rehypeRaw from 'rehype-raw';

const API_URL = '/api/chat'; // Proxy to LangGraph
const ASSISTANT_ID = process.env.NEXT_PUBLIC_LANGSMITH_ASSISTANT_ID;

export default function Chatbot() {
  const [currentThreadId, setCurrentThreadId] = useState<string | null>(null);
  const [messages, setMessages] = useState<Message[]>([]);

  // Initialize streaming hook
  const thread = useStream<ChatState>({
    apiUrl: API_URL,
    assistantId: ASSISTANT_ID,
    threadId: currentThreadId,
    messagesKey: "messages",
    onThreadId: (threadId) => {
      setCurrentThreadId(threadId);
      localStorage.setItem('chatThreadId', threadId);
    },
    onError: (error) => {
      console.error('Chat error:', error);
      enqueueSnackbar('Error en el chat', { variant: 'error' });
    },
  });

  useEffect(() => {
    // Load persisted thread ID
    const savedThreadId = localStorage.getItem('chatThreadId');
    if (savedThreadId) {
      setCurrentThreadId(savedThreadId);
    }
  }, []);

  useEffect(() => {
    // Update messages from stream
    if (thread.messages) {
      setMessages(thread.messages);
    }
  }, [thread.messages]);

  const sendMessage = async (content: string) => {
    // Optimistic update
    setMessages(prev => [...prev, {
      id: Date.now().toString(),
      role: 'user',
      content,
      timestamp: new Date().toISOString()
    }]);

    // Stream response
    await thread.stream({ messages: [{ role: 'user', content }] });
  };

  return (
    <div className="chatbot-container">
      {/* Message list */}
      <div className="messages">
        {messages.map(message => (
          <div key={message.id} className={`message ${message.role}`}>
            {message.role === 'assistant' ? (
              <ReactMarkdown
                remarkPlugins={[remarkGfm]}
                rehypePlugins={[rehypeRaw]}
              >
                {message.content}
              </ReactMarkdown>
            ) : (
              <p>{message.content}</p>
            )}
          </div>
        ))}

        {/* Streaming indicator */}
        {thread.isStreaming && (
          <div className="message assistant">
            <div className="typing-indicator">
              <span></span><span></span><span></span>
            </div>
          </div>
        )}
      </div>

      {/* Input form */}
      <form onSubmit={(e) => {
        e.preventDefault();
        const input = e.currentTarget.elements.namedItem('message') as HTMLInputElement;
        sendMessage(input.value);
        input.value = '';
      }}>
        <input
          type="text"
          name="message"
          placeholder="Escribe tu mensaje..."
          disabled={thread.isStreaming}
        />
        <button type="submit" disabled={thread.isStreaming}>
          {thread.isStreaming ? 'Enviando...' : 'Enviar'}
        </button>

        {/* Stop button (for streaming) */}
        {thread.isStreaming && (
          <button type="button" onClick={() => thread.stop()}>
            Detener
          </button>
        )}
      </form>

      {/* Branch switching (for edit/regenerate) */}
      {thread.branches && thread.branches.length > 1 && (
        <div className="branches">
          {thread.branches.map((branch, index) => (
            <button
              key={branch.id}
              onClick={() => thread.switchToBranch(branch.id)}
              className={branch.id === thread.currentBranchId ? 'active' : ''}
            >
              Variante {index + 1}
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

### Next.js API Proxy

**File:** `vitas-client/src/app/api/chat/[...path]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { cookies } from 'next/headers';

const LANGGRAPH_URL = process.env.LANGGRAPH_DEPLOY_URL;

export async function GET(
  request: NextRequest,
  { params }: { params: { path: string[] } }
) {
  return proxyToLangGraph(request, params.path, 'GET');
}

export async function POST(
  request: NextRequest,
  { params }: { params: { path: string[] } }
) {
  return proxyToLangGraph(request, params.path, 'POST');
}

async function proxyToLangGraph(
  request: NextRequest,
  pathSegments: string[],
  method: string
) {
  // Get JWT from cookie
  const authToken = cookies().get('authToken')?.value;

  if (!authToken) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Build LangGraph URL
  const path = pathSegments.join('/');
  const searchParams = request.nextUrl.searchParams.toString();
  const url = `${LANGGRAPH_URL}/${path}${searchParams ? `?${searchParams}` : ''}`;

  // Forward request to LangGraph
  const response = await fetch(url, {
    method,
    headers: {
      'Authorization': `Bearer ${authToken}`,
      'Content-Type': 'application/json',
      'assistant_id': process.env.LANGSMITH_ASSISTANT_ID || ''
    },
    body: method === 'POST' ? await request.text() : undefined
  });

  // Handle streaming responses (Server-Sent Events)
  const contentType = response.headers.get('content-type');

  if (contentType?.includes('text/event-stream')) {
    // Stream response directly
    return new NextResponse(response.body, {
      headers: {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
        'Connection': 'keep-alive'
      }
    });
  }

  // Regular JSON response
  const data = await response.json();
  return NextResponse.json(data, { status: response.status });
}
```

---

## Agent Tools & Capabilities

### Patient Agent Tools

#### 1. Get Available Doctors

**File:** `vitas-chatbot/src/react_agent/patient_agent/tools/get_doctors.py`

```python
from langchain.tools import tool
from react_agent.aws.dynamo_client import DynamoClient

@tool
def get_available_doctors(specialty: str = None) -> list[dict]:
    """
    Search for available doctors, optionally filtered by specialty.

    Args:
        specialty: Medical specialty (e.g., "cardiology", "general", "pediatrics")

    Returns:
        List of dictionaries with doctor information
    """
    client = DynamoClient()

    # Query Doctors_Table_V2
    if specialty:
        # Use active_for_ai_booking-index GSI
        doctors = client.query_doctors_by_specialty(specialty)
    else:
        # Scan all active doctors
        doctors = client.scan_active_doctors()

    return [
        {
            "doctor_id": doc["doctor_id"],
            "full_name": doc["full_name"],
            "specialty": doc.get("speciality", "General"),
            "email": doc.get("email")
        }
        for doc in doctors
        if doc.get("active_for_ai_booking") == True
    ]
```

#### 2. Find Free Slots

**File:** `vitas-chatbot/src/react_agent/patient_agent/tools/find_slots.py`

```python
@tool
def find_free_slots_for_doctor(doctor_id: str, date: str) -> list[dict]:
    """
    Find available appointment slots for a specific doctor on a given date.

    Args:
        doctor_id: The doctor's unique ID
        date: Date in YYYY-MM-DD format

    Returns:
        List of available time slots
    """
    client = DynamoClient()

    # Get doctor's schedule configuration
    schedule_config = client.get_schedule_config(doctor_id)

    if not schedule_config:
        return []

    # Get configured slots for the given day of week
    day_of_week = datetime.strptime(date, '%Y-%m-%d').strftime('%A').lower()
    configured_slots = schedule_config.get(day_of_week, [])

    # Get existing appointments for this date
    existing_appointments = client.get_appointments_by_doctor_and_date(doctor_id, date)
    booked_times = {appt["appointment_time"] for appt in existing_appointments}

    # Filter out booked slots
    available_slots = [
        {
            "time": slot["time"],
            "duration_minutes": slot.get("duration", 30)
        }
        for slot in configured_slots
        if slot["time"] not in booked_times
    ]

    return available_slots
```

#### 3. Schedule Appointment

**File:** `vitas-chatbot/src/react_agent/patient_agent/tools/schedule_appointment.py`

```python
import uuid
from datetime import datetime, timedelta

@tool
def schedule_patient_appointment(
    patient_id: str,
    doctor_id: str,
    date: str,
    time: str,
    reason: str = None
) -> dict:
    """
    Schedule a new appointment for the patient.

    Args:
        patient_id: Patient's unique ID
        doctor_id: Doctor's unique ID
        date: Appointment date (YYYY-MM-DD)
        time: Appointment time (HH:MM format)
        reason: Optional reason for visit

    Returns:
        Appointment details with confirmation
    """
    client = DynamoClient()

    appointment_id = str(uuid.uuid4())

    # Create appointment in Vitas_Appointment table
    appointment = {
        "doctor_id": doctor_id,
        "appt": f"APPT#{appointment_id}",
        "appointment_id": appointment_id,
        "patient_id": patient_id,
        "appointment_date": date,
        "appointment_time": time,
        "reason": reason,
        "status": "pending",  # Changed to "confirmed" after payment
        "paid": False,
        "created_at": datetime.now().isoformat(),
        "created_via": "chatbot",
        # TTL: 15 minutes (will be removed after payment confirmation)
        "ttl": int((datetime.now() + timedelta(minutes=15)).timestamp())
    }

    client.put_appointment(appointment)

    # Generate payment link (simplified)
    payment_url = f"https://payment.vitasclinic.com/pay/{appointment_id}"

    return {
        "appointment_id": appointment_id,
        "status": "pending_payment",
        "message": f"Cita reservada para {date} a las {time}. Completa el pago en: {payment_url}",
        "payment_url": payment_url,
        "expires_in_minutes": 15
    }
```

### Doctor Agent Tools

#### 1. Get Patients

**File:** `vitas-chatbot/src/react_agent/doctor_agent/tools/get_patients.py`

```python
@tool
def get_patients_by_doctor(doctor_id: str) -> list[dict]:
    """
    Get list of patients associated with this doctor.

    Args:
        doctor_id: Doctor's unique ID (extracted from JWT)

    Returns:
        List of patient profiles
    """
    client = DynamoClient()

    # Query PatientDoctors_Table_V2 by doctor_id (PK)
    relationships = client.query_patient_doctor_relationships(doctor_id)
    patient_ids = [rel["patient_id"] for rel in relationships]

    # Batch get patients from Patients_Table_V2
    patients = client.batch_get_patients(patient_ids)

    return [
        {
            "patient_id": p["patient_id"],
            "patient_name": p.get("patient_name", "Unknown"),
            "phone_number": p.get("phone_number"),
            "email": p.get("patient_email"),
            "age": p.get("patient_age")
        }
        for p in patients
    ]
```

#### 2. Get Consultation Details

**File:** `vitas-chatbot/src/react_agent/doctor_agent/tools/get_consultation.py`

```python
@tool
def get_consultation_details(appointment_id: str) -> dict:
    """
    Get complete consultation details including SOAP report and diagnoses.

    Args:
        appointment_id: The appointment ID

    Returns:
        Complete consultation data
    """
    client = DynamoClient()

    # Query Appointments_Table_V2 via appointment_id-index
    appointment = client.get_appointment_by_id(appointment_id)

    if not appointment:
        return {"error": "Appointment not found"}

    # Query Diagnosis_Table_V2
    diagnoses = client.get_diagnoses_by_appointment(appointment_id)

    return {
        "appointment_id": appointment["appointment_id"],
        "patient_id": appointment["patient_id"],
        "date": appointment.get("appointment_date"),
        "soap_report": {
            "subjective": appointment.get("soap_subjective"),
            "objective": appointment.get("soap_objective"),
            "assessment": appointment.get("soap_assessment"),
            "plan": appointment.get("soap_plan")
        },
        "icd_diagnoses": [
            {
                "code": dx.get("icd_code"),
                "description": dx.get("description")
            }
            for dx in diagnoses
        ],
        "transcription": appointment.get("transcription_text")
    }
```

---

## Session Management

### WhatsApp Session Lifecycle

**Session Table:** `wa_sessions`

**Schema:**
```typescript
{
  pk: string;              // "whatsapp:+51999999999" (partition key)
  threadId: string;        // LangGraph thread ID
  jwt: string;             // JWT token for API auth (2h expiry)
  patientId: string;       // Patient ID from Patients_Table_V2
  expiresAt: number;       // TTL (15 min idle timeout)
  absoluteExpiry: number;  // Hard 1-hour cap
  lastActiveAt: number;    // Last message timestamp
}
```

**Session Rules:**
1. **Idle Timeout:** 15 minutes of inactivity → session expires
2. **Absolute Timeout:** 1 hour maximum session duration
3. **JWT Rotation:** Auto-rotate JWT when <5 minutes remaining
4. **Renewal:** Each message extends `expiresAt` by 15 minutes (up to absolute limit)

**Implementation:** See WhatsApp Integration Flow above (`chat-integration.ts`)

---

## Streaming Implementation

### Server-Sent Events (SSE)

LangGraph supports streaming responses via Server-Sent Events (SSE), enabling real-time token-by-token rendering in the frontend.

**Backend (LangGraph):**
- Automatically streams responses when `stream=true` in API call
- Uses `text/event-stream` content type
- Sends events: `message_start`, `message_delta`, `message_end`

**Frontend (React):**
- `useStream()` hook handles SSE connection
- Automatically parses events and updates state
- Provides `isStreaming` flag for UI feedback

**Benefits:**
- Lower perceived latency (first tokens appear immediately)
- Better user experience (see response as it's generated)
- Ability to cancel generation mid-stream

---

## Error Handling & Fallbacks

### LangGraph Error Handling

**1. Tool Execution Errors**
```python
@tool
def get_available_doctors(specialty: str = None) -> list[dict]:
    try:
        # Query DynamoDB
        doctors = client.query_doctors(specialty)
        return doctors
    except Exception as e:
        # Return error message (agent will handle gracefully)
        return {
            "error": str(e),
            "message": "No se pudieron cargar los doctores disponibles"
        }
```

**2. Agent Retries**
LangGraph automatically retries failed tool calls up to 3 times with exponential backoff.

**3. Fallback Responses**
```python
# In agent state modifier
"""If you encounter errors or cannot complete a task:
- Apologize to the user
- Explain what went wrong (in simple terms)
- Suggest alternative actions
- Never expose technical error details"""
```

### WhatsApp Integration Error Handling

**1. Twilio Signature Validation Failure**
```typescript
if (!isValid) {
  console.warn('Invalid Twilio signature');
  return { statusCode: 403, body: 'Forbidden' };
}
```

**2. LangGraph API Failure**
```typescript
try {
  await fetch(langGraphUrl, {...});
} catch (error) {
  console.error('LangGraph error:', error);
  // Return empty TwiML to prevent Twilio retries
  // User will not receive a response, can retry by sending another message
}
```

**3. Session Expiry**
```typescript
if (session && session.expiresAt < Date.now()) {
  // Create new session
  session = await createNewSession(from);
  // User's conversation history is lost, starts fresh
}
```

### Frontend Error Handling

**1. Streaming Errors**
```typescript
const thread = useStream({
  //...
  onError: (error) => {
    console.error('Chat error:', error);
    enqueueSnackbar('Error en el chat. Por favor, intenta nuevamente.', {
      variant: 'error'
    });
  }
});
```

**2. Network Errors**
```typescript
// Auto-retry on network failure
const response = await fetch(url, { retry: 3, retryDelay: 1000 });
```

---

## Performance Optimization

### Latency Reduction Strategies

**Documented in:** `vitas-chatbot/LATENCY_OPTIMIZATION.md`

1. **Message Trimming** - Keep only last 20 messages in thread context
2. **Prompt Optimization** - Shortened system prompts (-30% tokens)
3. **Tool Result Filtering** - Return only necessary fields from tools
4. **Temperature 0** - Faster inference with deterministic outputs
5. **Streaming** - Token-by-token delivery reduces perceived latency
6. **Session Caching** - Reuse thread context across messages

**Results:**
- Average response time: <2 seconds (p95)
- Token usage reduction: 40%
- Cost savings: $0.02 → $0.01 per conversation

---

## Testing

### Local Development

```bash
cd vitas-chatbot

# Start LangGraph dev server
langgraph dev

# Test with curl
curl -X POST http://localhost:8000/threads/test-thread/runs \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer test-jwt" \
  -d '{
    "messages": [{"role": "user", "content": "Quiero agendar una cita"}],
    "source": "patient"
  }'
```

### WhatsApp Testing

Use Twilio WhatsApp Sandbox:
1. Send "join <sandbox-code>" to Twilio number
2. Send test messages
3. Monitor Lambda logs:
```bash
aws logs tail /aws/lambda/VitasChatbotStack-ChatIntegration --follow
```

---

## Summary

The VITAS Clinic AI chatbot provides:
- **Dual interfaces:** WhatsApp (patients) + Web (doctors)
- **Role-based agents:** Patient vs Doctor with specialized tools
- **Stateful conversations:** LangGraph thread persistence
- **Real-time streaming:** Token-by-token response rendering
- **Secure authentication:** JWT with auto-rotation
- **Session management:** 15-min idle, 1-hour absolute timeouts
- **Database integration:** DynamoDB queries via agent tools

**Related Documentation:**
- [Frontend-Backend Integration](03-FRONTEND_BACKEND_INTEGRATION.md) - General API integration
- [Database Schema](07-DATABASE_SCHEMA.md) - Complete table schemas

**Last Updated:** 2025-11-06
