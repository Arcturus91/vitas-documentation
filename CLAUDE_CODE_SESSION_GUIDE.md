# Claude Code Session Starter Guide

**How to Efficiently Use VITAS Documentation in Future Sessions**

This guide shows you how to get answers from Claude Code while **saving 90%+ tokens** by using the existing documentation instead of re-exploring the codebase.

---

## 🚀 Quick Start Template

Copy and paste this template when starting a new Claude Code session:

```
Hi Claude! I'm working on the VITAS Clinic project.

Comprehensive documentation exists at:
/Users/victorbarrantes/Desktop/coding/cloudforge-ai/vitasClinic/implementationDocs/

Please:
1. Read README.md first to understand the documentation structure
2. When I ask questions, reference the relevant docs instead of exploring code
3. Only read source files if the docs don't have the answer
4. Use Grep to search docs before reading entire files

My question: [YOUR SPECIFIC QUESTION HERE]
```

---

## 📚 Documentation Quick Reference

### Navigation Hub
**Start here:** `/implementationDocs/README.md`
- Contains links to all other docs
- Organized by topic area
- Quick reference tables

### Most Common Docs by Task

| Your Task | Read This Doc | Tokens |
|-----------|---------------|--------|
| Deploy a stack | `02-DEPLOYMENT_GUIDE.md` | ~3k |
| Understand chatbot | `04-AI_CHATBOT_INTEGRATION.md` | ~5k |
| Add API endpoint | `06-API_CONTRACTS.md` | ~3k |
| Query database | `07-DATABASE_SCHEMA.md` | ~4k |
| Debug processing | `05-PROCESSING_PIPELINE_INTEGRATION.md` | ~4k |
| Fix security issue | `09-SECURITY_AND_COMPLIANCE.md` | ~4k |
| Setup monitoring | `08-MONITORING_AND_OBSERVABILITY.md` | ~4k |
| Understand architecture | `01-SYSTEM_ARCHITECTURE_OVERVIEW.md` | ~6k |
| Frontend integration | `03-FRONTEND_BACKEND_INTEGRATION.md` | ~5k |
| Business overview | `00-EXECUTIVE_SUMMARY.md` | ~4k |

---

## ✅ Efficient Prompts (Token-Saving)

### Pattern: Specific Question + Doc Reference

**Template:**
```
Read [SPECIFIC_DOC].md and [DO_SPECIFIC_TASK]
```

**Examples:**

#### Example 1: Deployment Help
```
Read 02-DEPLOYMENT_GUIDE.md and help me deploy vitas-processing-stack.
What are the prerequisites and step-by-step commands?
```
**Token cost:** ~5k tokens (vs ~50k exploring code)

#### Example 2: Add Feature
```
Read 05-PROCESSING_PIPELINE_INTEGRATION.md and
01-SYSTEM_ARCHITECTURE_OVERVIEW.md.

I want to add a new Lambda function that processes medical images.
Where should it fit in the architecture and how do I integrate it
with Step Functions?
```
**Token cost:** ~12k tokens (vs ~80k exploring code)

#### Example 3: Fix Bug
```
Read 07-DATABASE_SCHEMA.md.

I'm getting an error querying Appointments_Table_V2 by appointment_id.
Which GSI should I use and show me the correct query syntax.
```
**Token cost:** ~6k tokens (vs ~40k searching code)

#### Example 4: Security Audit
```
Read 09-SECURITY_AND_COMPLIANCE.md.

Review our current JWT implementation and suggest improvements
for HIPAA compliance. What are the gaps?
```
**Token cost:** ~8k tokens (vs ~60k exploring code)

---

## ⚡ Ultra-Efficient: Search First Strategy

Use Grep before reading entire docs:

### Pattern: Grep + Targeted Read

```
Search for "JWT token" in implementationDocs/ using Grep.
Then read only the relevant sections.
```

**Example:**
```
Use Grep to find all references to "DLQ" (Dead Letter Queue)
in the documentation. Then explain the DLQ retry strategy.
```
**Token cost:** ~2k tokens (vs ~20k reading full docs)

---

## 📊 Token Cost Comparison

### ❌ Token-Wasting Approach
```
User: "Explain how the system works"

Claude would need to:
- Explore all 4 AWS stacks (80k tokens)
- Read frontend code (40k tokens)
- Trace integrations (30k tokens)
Total: 150k+ tokens ⚠️
```

### ✅ Token-Efficient Approach
```
User: "Read README.md, then read 01-SYSTEM_ARCHITECTURE_OVERVIEW.md
and give me a summary of the system"

Claude reads:
- README.md (1k tokens)
- Architecture overview (6k tokens)
- Generates summary (2k tokens)
Total: 9k tokens ✅ (94% savings)
```

---

## 🎯 Best Practices

### 1. **Always Start with README.md**
```
Read README.md and tell me which doc covers [TOPIC]
```

### 2. **Be Specific About What You Need**
```
❌ Bad:  "Tell me about authentication"
✅ Good: "Read 03-FRONTEND_BACKEND_INTEGRATION.md and explain
         the doctor login flow with code examples"
```

### 3. **Use Diagrams for Visual Understanding**
```
Show me diagrams/audio-processing-flow.mmd and explain the
retry logic when transcription fails
```

### 4. **Incremental Deep Dive**
```
Session progression (token-efficient):

Step 1: "Read README.md - where is chatbot documented?"
        → Points to 04-AI_CHATBOT_INTEGRATION.md (1k tokens)

Step 2: "Read that doc and explain the WhatsApp flow"
        → Reads doc, explains (5k tokens)

Step 3: "Now show me the actual chat-integration Lambda code"
        → Reads source file (5k tokens)

Total: 11k tokens (vs 60k if exploring from scratch)
```

### 5. **Search Before Reading**
```
Grep for "MercadoPago" in implementationDocs/ and tell me
which docs mention it. Then explain the payment integration.
```

### 6. **Reference Multiple Docs When Needed**
```
Read 02-DEPLOYMENT_GUIDE.md and 09-SECURITY_AND_COMPLIANCE.md.
What security checks should I do before deploying to production?
```

---

## 🔍 Common Tasks & Optimal Prompts

### Task: Understand a Feature

**Optimal prompt:**
```
I need to understand [FEATURE].

First, grep for "[FEATURE]" in implementationDocs/ to find
relevant docs. Then read those docs and explain how it works.
```

### Task: Add New Functionality

**Optimal prompt:**
```
I want to add [NEW_FEATURE].

Read:
- 01-SYSTEM_ARCHITECTURE_OVERVIEW.md (understand where it fits)
- [RELEVANT_INTEGRATION_DOC].md (see integration pattern)

Then suggest implementation approach with code examples.
```

### Task: Debug an Issue

**Optimal prompt:**
```
I'm seeing error: [ERROR_MESSAGE]

Search implementationDocs/ for related terms, then read relevant
sections. Explain likely causes and fixes.
```

### Task: Review Code Quality

**Optimal prompt:**
```
Read 09-SECURITY_AND_COMPLIANCE.md and review this code:
[PASTE CODE]

Does it follow our security best practices?
```

---

## 📈 Token Savings Calculator

| Approach | Token Cost | Use When |
|----------|------------|----------|
| **Grep search** | 500-1k | Quick lookup, finding where something is documented |
| **Read 1 doc** | 3-6k | Focused question on one topic |
| **Read 2-3 docs** | 8-15k | Cross-cutting concern (e.g., security + deployment) |
| **Read README + 1 doc** | 4-8k | Starting a session, need orientation |
| **Read diagram** | 1-2k | Visual understanding of flow |
| **Explore code directly** | 40-100k+ | Docs don't have answer (rare) |

---

## 🛠️ Template Library

Copy these templates for common tasks:

### Template: Deployment
```
Read 02-DEPLOYMENT_GUIDE.md.

I want to deploy [STACK_NAME]. Show me:
1. Prerequisites
2. Step-by-step commands
3. Verification steps
4. Common errors and fixes
```

### Template: Add Feature
```
Read 01-SYSTEM_ARCHITECTURE_OVERVIEW.md and [RELEVANT_DOC].md.

I want to add [FEATURE_DESCRIPTION].

Please:
1. Identify which stack/component it belongs to
2. Show architecture integration points
3. Provide code skeleton
4. List required environment variables
```

### Template: Debug
```
Grep for "[ERROR_KEYWORD]" in implementationDocs/.

Then read relevant sections and explain:
1. What causes this error
2. How to fix it
3. How to prevent it in the future
```

### Template: Security Review
```
Read 09-SECURITY_AND_COMPLIANCE.md.

Review [CODE/FEATURE] against our security standards.
Identify:
1. Vulnerabilities
2. Best practice violations
3. Recommended fixes
```

### Template: Database Query
```
Read 07-DATABASE_SCHEMA.md.

I need to [QUERY_DESCRIPTION]. Show me:
1. Which table(s) to query
2. Which GSI to use (if any)
3. Example DynamoDB query code
4. Access pattern documentation
```

---

## 🎓 Advanced Techniques

### Technique 1: Lazy Loading Docs
```
Read only what you need, when you need it:

User: "I want to add a new API endpoint"
Claude: [Reads README.md → sees 06-API_CONTRACTS.md]
        [Reads only API contracts doc]

User: "Now help me deploy it"
Claude: [Now reads 02-DEPLOYMENT_GUIDE.md]
        [Doesn't re-read architecture docs]
```

### Technique 2: Context Preservation
```
In a single session, build context incrementally:

Query 1: "Read 01-SYSTEM_ARCHITECTURE_OVERVIEW.md"
         [Claude now has system context for rest of session]

Query 2: "Based on what you just read, where would a
         PDF generation feature fit?"
         [Uses cached context, no re-reading]

Query 3: "Show me the relevant CDK code"
         [Only now reads source files]
```

### Technique 3: Diagram-First Approach
```
For complex flows, start with diagrams:

1. "Show me diagrams/audio-processing-flow.mmd"
   [Visual understanding, 1k tokens]

2. "Now read the 'Error Handling' section of
    05-PROCESSING_PIPELINE_INTEGRATION.md"
   [Focused reading, 2k tokens]

3. "Show me the DLQ processor Lambda code"
   [Targeted source code, 4k tokens]

Total: 7k tokens (vs 60k exploring code blindly)
```

---

## ⚠️ Avoid These Token Traps

### Trap 1: Vague Questions
```
❌ "Tell me about the system"
   → Forces Claude to read everything (80k+ tokens)

✅ "Read README.md and tell me the main components"
   → Targeted, efficient (2k tokens)
```

### Trap 2: Exploring Code First
```
❌ "Look at vitas-processing-stack and explain what it does"
   → Must read all CDK code, Lambda functions (40k tokens)

✅ "Read 01-SYSTEM_ARCHITECTURE_OVERVIEW.md section on
    vitas-processing-stack"
   → Reads summary (3k tokens)
```

### Trap 3: Not Using Search
```
❌ "Where is JWT validation implemented?"
   → Would need to explore all repos (60k tokens)

✅ "Grep for 'JWT validation' in implementationDocs/"
   → Finds it instantly (500 tokens)
```

---

## 📊 Real Session Example

**Scenario:** You want to add a new notification type

### ❌ Inefficient Session (102k tokens)
```
User: "I want to add a new notification type"

Claude explores:
- vitas-processing-stack code (30k)
- vitas-client notification code (25k)
- Lambda functions (20k)
- DynamoDB tables (15k)
- Generates recommendations (12k)
Total: 102k tokens
```

### ✅ Efficient Session (8k tokens)
```
User: "Grep for 'notification' in implementationDocs/ and tell me
      which docs cover it"

Claude: [Greps, finds 05-PROCESSING_PIPELINE_INTEGRATION.md] (500 tokens)

User: "Read that doc's notification section"

Claude: [Reads only notifications section] (2k tokens)

User: "Based on this, show me how to add a new notification type"

Claude: [Generates code based on doc pattern] (3k tokens)

User: "Now show me the actual Lambda code to verify"

Claude: [Reads failure-notification Lambda] (2.5k tokens)

Total: 8k tokens (92% savings!)
```

---

## 🎯 Quick Decision Tree

```
START: I have a question about VITAS

├─ Is it about "what/where/which"?
│  └─ Use Grep first (500 tokens)
│     └─ Then read found doc (3-5k tokens)
│
├─ Is it about "how does X work"?
│  └─ Read README → relevant doc (5-8k tokens)
│
├─ Is it about "how do I implement Y"?
│  └─ Read architecture + integration doc (10-15k tokens)
│     └─ Then read source code if needed (+5-10k tokens)
│
└─ Is it about "show me the code"?
   └─ Read doc first to understand context (3-5k tokens)
      └─ Then read source with targeted search (5-10k tokens)
```

---

## 💡 Pro Tips

1. **Bookmark README.md** - Always your starting point
2. **Use Grep liberally** - Fastest way to find information
3. **Read diagrams first** - Visual understanding is faster
4. **Ask targeted questions** - Specific = efficient
5. **Build context incrementally** - Don't re-read in same session
6. **Reference docs by name** - "Read X.md" is faster than "tell me about X"
7. **Check existing examples** - Docs have code snippets
8. **Trust the docs** - They were created from comprehensive exploration

---

## 📝 Session Checklist

Before asking Claude to explore code, ask yourself:

```
☐ Did I read README.md to know what's documented?
☐ Did I grep for relevant keywords?
☐ Did I specify which doc to read?
☐ Is my question specific enough?
☐ Can a diagram answer this faster?
☐ Do I really need source code, or will doc examples work?
```

If you checked 3+ boxes, you're being token-efficient! ✅

---

## 🏆 Success Metrics

You're using this guide effectively when:

- ✅ Most sessions use <20k tokens
- ✅ You reference specific docs by name
- ✅ You use Grep for quick lookups
- ✅ You read diagrams before code
- ✅ You get answers without exploring source code
- ✅ Sessions complete faster than before

---

## 📞 Need Help?

If docs don't have the answer:
1. Ask Claude to update the relevant doc
2. Or explore code, then add findings to docs
3. Keep documentation up-to-date for future sessions

**Remember:** Every time you explore code without checking docs first,
you're spending 10x more tokens than necessary! 💸

---

**Last Updated:** 2025-11-06

**Questions?** Read README.md or grep for your topic in implementationDocs/
