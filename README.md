# VITAS Clinic - Complete System Integration Documentation

Welcome to the comprehensive integration documentation for the VITAS Clinic healthcare platform. This documentation covers the complete system architecture, including all AWS infrastructure stacks, frontend applications, and AI-powered chatbot integration.

## 🚀 New to Claude Code? Start Here!

**→ [Claude Code Session Starter Guide](CLAUDE_CODE_SESSION_GUIDE.md)** - Learn how to use this documentation efficiently in future Claude Code sessions and **save 90%+ tokens**

This guide shows you:
- How to get answers for ~5-15k tokens instead of 100k+
- Templates for common tasks
- Token-saving strategies and best practices

## 🎯 Quick Navigation

### Executive Documentation
- **[Executive Summary](00-EXECUTIVE_SUMMARY.md)** - High-level system overview for stakeholders and decision-makers

### Technical Architecture
- **[System Architecture Overview](01-SYSTEM_ARCHITECTURE_OVERVIEW.md)** - Complete system architecture across all 4 AWS stacks + LangGraph
- **[Deployment Guide](02-DEPLOYMENT_GUIDE.md)** - Stack deployment order, dependencies, and verification procedures

### Integration Documentation
- **[Frontend-Backend Integration](03-FRONTEND_BACKEND_INTEGRATION.md)** - Next.js ↔ API Gateway ↔ Lambda integration flows
- **[AI Chatbot Integration](04-AI_CHATBOT_INTEGRATION.md)** - WhatsApp → Twilio → Lambda → LangGraph complete flow
- **[Processing Pipeline Integration](05-PROCESSING_PIPELINE_INTEGRATION.md)** - Audio processing with Step Functions and AI analysis

### API & Data Contracts
- **[API Contracts](06-API_CONTRACTS.md)** - All REST endpoints, schemas, authentication requirements
- **[Database Schema](07-DATABASE_SCHEMA.md)** - Complete DynamoDB schema (12 tables) with relationships

### Operations & Security
- **[Monitoring & Observability](08-MONITORING_AND_OBSERVABILITY.md)** - CloudWatch dashboards, alarms, and metrics
- **[Security & Compliance](09-SECURITY_AND_COMPLIANCE.md)** - IAM, JWT, secrets management, CORS, rate limiting

## 📊 Architecture Diagrams

All diagrams are created using Mermaid (text-based, renders natively in GitHub):

- **[Complete System Architecture](diagrams/complete-system-architecture.mmd)** - High-level view of all components
- **[Audio Processing Flow](diagrams/audio-processing-flow.mmd)** - End-to-end audio → transcription → AI analysis
- **[Chatbot Conversation Flow](diagrams/chatbot-conversation-flow.mmd)** - WhatsApp message → LangGraph → response
- **[Appointment Booking Flow](diagrams/appointment-booking-flow.mmd)** - Frontend → API → payment → confirmation
- **[Authentication Flow](diagrams/authentication-flow.mmd)** - OTP → JWT → API access
- **[Database Relationships](diagrams/database-relationships.mmd)** - Entity relationship diagram (ERD)

## 🏗️ System Components

### AWS Infrastructure Stacks

| Stack | Purpose | Key Resources |
|-------|---------|---------------|
| **vitas-main-stack** | Core authentication & user management | API Gateway, Auth Lambda, 7 DynamoDB tables, S3, CloudFront |
| **vitas-processing-stack** | Medical audio processing pipeline | Step Functions, 7 Lambdas, 3 DLQs, EventBridge, S3 |
| **vitas-auth-otp** | OTP authentication & chatbot gateway | 9 Lambdas, Twilio integration, WhatsApp webhook |
| **vitas-chatbot** | AI conversational assistant (LangGraph) | Python LangGraph, ReAct agents, OpenAI GPT-4o-mini |

### Frontend Applications

| Application | Technology | Purpose |
|-------------|------------|---------|
| **vitas-client** | Next.js 14, React 18, TypeScript | Doctor dashboard, SOAP reports, patient management |
| **vitas-patients-front** | Next.js, React | Patient portal, appointment booking |

## 🔑 Key Features

### Authentication & Security
- **OTP-based authentication** via WhatsApp (no passwords)
- **JWT tokens** with context-aware expiration (2h chatbot, 24h payment)
- **Rate limiting** via Upstash Redis
- **IAM-based access control** across all AWS resources

### AI-Powered Features
- **Medical audio transcription** (OpenAI Whisper → AWS Transcribe fallback)
- **SOAP report generation** (OpenAI GPT-4o → Anthropic Claude fallback)
- **ICD-10 diagnosis classification** (AI-powered medical coding)
- **Conversational chatbot** (LangGraph with doctor/patient agents)

### Processing Reliability
- **Step Functions orchestration** (Standard + Express workflows)
- **Automatic retries** (3 attempts with exponential backoff)
- **Dead Letter Queues** (DLQ) for failed processing
- **24-hour SLA guarantee** via scheduled retries (every 2 hours)
- **Real-time notifications** via webhooks

### Integrations
- **Twilio WhatsApp Business API** - Message delivery
- **MercadoPago** - Payment processing
- **Deepgram** - Real-time audio transcription
- **OpenAI & Anthropic** - AI/LLM providers
- **LangGraph Cloud** - Stateful conversation management

## 📍 AWS Region

All infrastructure is deployed in:
- **Primary Region:** `sa-east-1` (São Paulo, Brazil)

## 🚀 Getting Started

1. **For Stakeholders:** Start with the [Executive Summary](00-EXECUTIVE_SUMMARY.md)
2. **For Developers:** Review [System Architecture](01-SYSTEM_ARCHITECTURE_OVERVIEW.md) then [Deployment Guide](02-DEPLOYMENT_GUIDE.md)
3. **For DevOps:** Focus on [Deployment Guide](02-DEPLOYMENT_GUIDE.md) and [Monitoring](08-MONITORING_AND_OBSERVABILITY.md)
4. **For Security Audits:** See [Security & Compliance](09-SECURITY_AND_COMPLIANCE.md)

## 📈 System Metrics

- **Total DynamoDB Tables:** 12
- **Total Lambda Functions:** 17
- **Total S3 Buckets:** 3
- **API Gateway Endpoints:** 35+
- **Step Functions Workflows:** 2
- **EventBridge Rules:** 1 (scheduled)
- **SQS Queues:** 3 (DLQs)

## 🔗 Related Repositories

- [vitas-client](../vitas-client/) - Doctor frontend application
- [vitas-main-stack](../vitas-main-stack/) - Core AWS infrastructure
- [vitas-processing-stack](../vitas-processing-stack/) - Audio processing pipeline
- [vitas-auth-otp](../vitas-auth-otp/) - Authentication & chatbot gateway
- [vitas-chatbot](../vitas-chatbot/) - LangGraph AI assistant

## 📝 Documentation Updates

This documentation was last updated: **2025-11-06**

For questions or contributions, please contact the development team.

---

**© 2025 VITAS Clinic - Comprehensive Healthcare Platform**
# vitas-documentation
