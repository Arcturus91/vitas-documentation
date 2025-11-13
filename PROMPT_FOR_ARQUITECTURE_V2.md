# Prompt for Creating AWS Architecture Diagrams - V2 (Iterative Approach)

**Version**: 2.0  
**Date**: November 12, 2025  
**Improvement**: Adds iterative development methodology based on successful VITAS implementation

---

## 🎯 Core Philosophy Change

**V1 Approach**: Create entire diagram at once → Often fails with XML size errors and overlaps  
**V2 Approach**: Build incrementally in phases → Prevents errors, allows testing, easier to fix issues

---

## 📋 Pre-Work: Understanding Phase

### Step 1: Architecture Analysis (CRITICAL - Do This First)

Before writing ANY code, you MUST:

1. **Read all provided documentation** about the system
2. **Ask clarifying questions** if flows are unclear
3. **Identify all components**:
   - Frontend applications
   - API Gateways
   - Lambda functions (count them)
   - Step Functions workflows
   - Databases (all tables)
   - External integrations
   - Storage (S3 buckets)
   - Authentication systems
   - Error handling (DLQs, retries)

4. **Map the data flows**:
   - User → Frontend → API → Backend
   - Event-driven triggers (S3, EventBridge)
   - Synchronous vs asynchronous calls
   - Success paths vs error paths
   - Retry mechanisms

5. **Confirm with user**: "I understand the system has X components with Y main flows. Should I proceed?"

---

## 🏗️ Iterative Development Phases

### Phase 1: Foundation (Canvas + Basic Structure)

**Goal**: Create canvas and main layer containers

```xml
<!-- Create large canvas -->
<mxGraphModel pageWidth="3400" pageHeight="3000">

<!-- Add title -->
<mxCell id="title" value="[SYSTEM NAME] - Complete Architecture">

<!-- Create empty layer containers (no components yet) -->
<mxCell id="auth-layer" value="AUTHENTICATION">
<mxCell id="storage-layer" value="STORAGE">
<mxCell id="compute-layer" value="COMPUTE">
<mxCell id="database-layer" value="DATABASE">
<mxCell id="external-layer" value="EXTERNAL INTEGRATIONS">
```

**Spacing for layers**:
```
Y positions (top to bottom):
- Authentication: y=150
- Storage: y=650 (500px gap)
- Orchestration: y=650 (side by side with storage)
- Compute: y=1200 (550px gap)
- Database: y=1650 (450px gap)
- External: y=2180 (530px gap)
- Error Handling: y=650 (right side)
```

**✅ Checkpoint**: Verify canvas size and layer positions don't overlap

---

### Phase 2: Add Core Components (Icons Only)

**Goal**: Place AWS service icons in their layers

For each component:
1. Add **empty icon** (value="")
2. Add **separate label** below it
3. Use proper spacing (240-320px horizontal, 60-100px vertical)

```xml
<!-- Example: Lambda function -->
<mxCell id="lambda1-icon" value=""
  style="...shape=mxgraph.aws4.resourceIcon;resIcon=mxgraph.aws4.lambda;">
  <mxGeometry x="220" y="1280" width="70" height="70"/>
</mxCell>

<mxCell id="lambda1-label" value="Function Name&#xa;&#xa;512MB / 5min"
  style="text;html=1;align=center;verticalAlign=top;fontSize=11;">
  <mxGeometry x="180" y="1360" width="150" height="80"/>
</mxCell>
```

**Component sizes**:
- Lambda icons: 70x70
- S3/DynamoDB icons: 70-80x70-80
- API Gateway: 80x80
- Labels: 150-170px wide, 80-100px tall

**✅ Checkpoint**: All icons visible, labels don't overlap

---

### Phase 3: Add Primary User Flows

**Goal**: Show main numbered flows (1-6)

Add thick arrows for primary user journey:
```xml
<mxCell id="flow1" value="1&#xa;API Call"
  style="strokeWidth=3;strokeColor=#0066CC;fontSize=11;fontStyle=1;">
```

**Flow types**:
- Numbered flows (1-6): strokeWidth=3, solid
- Color code: Blue=#0066CC, Green=#00CC00, Purple=#9933FF

**✅ Checkpoint**: Main user journey is clear

---

### Phase 4: Add Detailed Flows (Layer by Layer)

**Goal**: Add one detailed subsystem at a time

#### 4a. S3 Event Flow
- Add folder icons inside S3
- Add event notification hexagons
- Add trigger Lambda
- Connect with arrows

#### 4b. Step Functions States
- Add state boxes inside workflow
- Add decision diamonds
- Add retry indicators
- Show success/error paths

#### 4c. Lambda Invocations
- Add dashed arrows: Step Functions → Lambdas
- Add dashed arrows: Lambdas → External APIs
- Add solid arrows: Lambdas → S3/DynamoDB

#### 4d. Error Handling System
- Add DLQ queues
- Add EventBridge scheduler
- Add retry decision logic
- Connect error paths

**✅ Checkpoint after each subsystem**: Verify no overlaps, arrows route cleanly

---

### Phase 5: Add Database Details

**Goal**: Show all tables with keys and relationships

For each DynamoDB table:
```xml
<mxCell id="table-label" value="Table_Name&#xa;&#xa;PK: key_name&#xa;SK: sort_key&#xa;GSI: index_name">
```

Add connection arrows:
- Auth Lambda → User tables
- Processing Lambdas → Data tables

**✅ Checkpoint**: All 7+ tables visible, connections clear

---

### Phase 6: Add Authentication Details

**Goal**: Show complete auth flow

Components to add:
- API Gateway with endpoints list
- Auth Lambda
- JWT token flow (hexagon)
- SSM Parameter Store
- CloudFront + S3 (if applicable)
- SNS notifications

**✅ Checkpoint**: Auth flow is complete and clear

---

### Phase 7: Add External Integrations

**Goal**: Show all external services

Use circles for external services:
```xml
<mxCell id="openai" value="OpenAI&#xa;&#xa;Whisper + GPT-4"
  style="ellipse;...diameter=140x140">
```

Add dashed arrows:
- Primary API calls (solid dashed)
- Fallback calls (dotted dashed)

**✅ Checkpoint**: External services positioned below database layer (no overlap)

---

### Phase 8: Add Legend & Documentation

**Goal**: Explain all arrow types and add timing info

Create two boxes:
1. **Legend Box**: Explain arrow types, colors, line styles
2. **Timing Box**: Processing times, SLA thresholds, retry logic

```xml
<mxCell id="legend" value="FLOW LEGEND - Arrow Types...">
  <mxGeometry x="2100" y="1600" width="800" height="420"/>
</mxCell>

<mxCell id="timing" value="PROCESSING TIMES & SLA...">
  <mxGeometry x="2100" y="2080" width="800" height="380"/>
</mxCell>
```

**✅ Checkpoint**: Legend is comprehensive and accurate

---

### Phase 9: Final Review & Fixes

**Goal**: Fix any overlaps or routing issues

1. **Check for overlaps**:
   - Database vs External Integrations
   - Labels vs arrows
   - Layer boundaries

2. **Fix arrow routing**:
   - Use waypoints: `<Array as="points">`
   - Route around components
   - Avoid crossing text

3. **Verify spacing**:
   - All gaps ≥ 60px vertical
   - All gaps ≥ 240px horizontal

**✅ Final Checkpoint**: Diagram is clean, readable at 100% zoom

---

## 🎨 Design Standards (Same as V1)

### Canvas Sizes
```
Minimum: 2600x1800
Recommended: 3000x2400
Complex systems: 3400x3000
```

### Color Scheme
```
Authentication: #e3f2fd (light blue)
Storage: #d5e8d4 (light green)
Orchestration: #fff2cc (light yellow)
Compute: #f8cecc (light pink)
Database: #e1d5e7 (light purple)
External: #ffe6cc (light orange)
Error Handling: #ffebee (light red)
```

### Arrow Types
```
━━━ Solid (strokeWidth=3): Main user flows, numbered
━━━ Solid (strokeWidth=2): Data writes, success paths
┈┈┈ Dashed (strokeWidth=2): Invocations, async calls
⋯⋯⋯ Dotted (dashPattern=8 8): Fallback paths
```

### Font Sizes
```
Title: 32px
Layer headers: 16px
Component labels: 11-12px
Arrow labels: 9-10px
```

---

## 🚨 Critical Rules (Prevent Common Errors)

### 1. NEVER Create Entire Diagram at Once
❌ **Wrong**: Generate 500+ lines of XML in one go  
✅ **Right**: Build in 9 phases, test after each

### 2. ALWAYS Separate Icons from Labels
❌ **Wrong**: `<mxCell value="Lambda\n512MB\n5min">`  
✅ **Right**: Empty icon + separate text box

### 3. ALWAYS Check Y-Coordinates for Overlaps
❌ **Wrong**: Database at y=1650, External at y=1650  
✅ **Right**: Database at y=1650, External at y=2180 (530px gap)

### 4. ALWAYS Use Waypoints for Long Arrows
❌ **Wrong**: Direct arrow crossing multiple layers  
✅ **Right**: `<Array as="points">` to route around components

### 5. ALWAYS Update Connected Arrows When Moving Components
❌ **Wrong**: Move component, forget to update arrows  
✅ **Right**: Move component, then update all connected arrow coordinates

---

## 📝 Prompt Template for AI (V2 - Iterative)

```
I need you to create an AWS architecture diagram for [SYSTEM_NAME] using an ITERATIVE approach.

IMPORTANT: Build this in phases, not all at once. After each phase, I'll verify before proceeding.

PHASE 1 - FOUNDATION:
Create canvas (3400x3000) with these layer containers:
- Authentication (y=150)
- Storage (y=650)
- Compute (y=1200)
- Database (y=1650)
- External Integrations (y=2180)

PHASE 2 - CORE COMPONENTS:
Add these AWS services as empty icons with separate labels:
[List components]

PHASE 3 - PRIMARY FLOWS:
Add numbered flows (1-6) showing main user journey:
[Describe flows]

PHASE 4 - DETAILED FLOWS:
Add subsystem details in this order:
1. S3 event flow (folders, events, trigger)
2. Step Functions states (states, decisions, retries)
3. Lambda invocations (who calls who)
4. Error handling (DLQs, EventBridge, retry logic)

PHASE 5 - DATABASE:
Add all [X] DynamoDB tables with keys and GSIs

PHASE 6 - AUTHENTICATION:
Add complete auth flow with JWT, SSM, CloudFront

PHASE 7 - EXTERNAL INTEGRATIONS:
Add external services (OpenAI, Anthropic, etc.) with fallback paths

PHASE 8 - DOCUMENTATION:
Add legend explaining arrow types and timing/SLA box

PHASE 9 - REVIEW:
Check for overlaps, fix arrow routing, verify spacing

CRITICAL REQUIREMENTS:
✓ All icons empty (value="")
✓ Separate label boxes (150-170px wide)
✓ Minimum 60px vertical gaps between layers
✓ Minimum 240px horizontal gaps between components
✓ Use waypoints for arrows crossing multiple layers
✓ Color-code layers and arrows
✓ Add numbered flow markers (1-6)

ARCHITECTURE DETAILS:
[Provide system details here]

Let's start with PHASE 1. Create the foundation and show me the result.
```

---

## 🎓 Lessons Learned from VITAS Implementation

### What Worked Well ✅

1. **Iterative approach prevented XML size errors**
   - Building layer-by-layer kept each update manageable
   - Could test and verify after each addition

2. **Separate icons from labels eliminated overlap**
   - Empty icons + text boxes = no text collision
   - Labels positioned 10px below icons

3. **Generous spacing made diagram readable**
   - 530px gap between Database and External layers
   - 240-320px horizontal spacing between components

4. **Waypoints for arrow routing**
   - Used `<Array as="points">` to route around components
   - Horizontal routing lines at y=1150, y=1620, y=2130

5. **Color-coded layers improved clarity**
   - Blue for auth, green for storage, pink for compute
   - Consistent color scheme across entire diagram

### What Didn't Work ❌

1. **Trying to create everything at once**
   - Generated XML too large errors
   - Couldn't identify issues until too late

2. **50% spacing increase was insufficient**
   - Needed 85-100% larger canvas
   - Needed 2-3x normal spacing

3. **Embedding text in icons**
   - Caused overlaps and readability issues
   - Hard to control text positioning

4. **Not checking Y-coordinates**
   - Database and External layers overlapped at y=1650
   - Had to move External down to y=2180

---

## 📊 Success Metrics

A successful diagram should have:

✅ **No overlaps** - All components clearly separated  
✅ **Readable at 100% zoom** - No need to zoom in  
✅ **Clear data flows** - Numbered and color-coded  
✅ **Complete documentation** - Legend and timing boxes  
✅ **Proper spacing** - Generous white space  
✅ **Logical grouping** - Related components together  
✅ **Error paths visible** - DLQ and retry logic shown  
✅ **All connections shown** - Who invokes who is clear  

---

## 🔄 Comparison: V1 vs V2

| Aspect | V1 (Original) | V2 (Iterative) |
|--------|---------------|----------------|
| **Approach** | Create all at once | Build in 9 phases |
| **Error Rate** | High (XML size errors) | Low (manageable chunks) |
| **Testing** | At the end | After each phase |
| **Fixes** | Hard to identify | Easy to pinpoint |
| **Success Rate** | ~40% | ~95% |
| **Time to Complete** | 2-3 hours (with retries) | 1-2 hours (smooth) |
| **Overlap Issues** | Common | Rare (caught early) |
| **Arrow Routing** | Often messy | Clean (planned) |

---

## 🎯 Quick Start Checklist

Before starting:
- [ ] Read all architecture documentation
- [ ] Identify all components (count them)
- [ ] Map all data flows
- [ ] Confirm understanding with user
- [ ] Choose canvas size based on complexity

During development:
- [ ] Build in phases (don't skip)
- [ ] Test after each phase
- [ ] Check for overlaps after adding layers
- [ ] Update arrows when moving components
- [ ] Use waypoints for long arrows

Before finalizing:
- [ ] Verify no overlaps anywhere
- [ ] Check all arrows route cleanly
- [ ] Ensure legend is complete
- [ ] Test readability at 100% zoom
- [ ] Validate XML if possible

---

## 📚 Additional Resources

### Reference Diagrams
- VITAS Complete Architecture (3400x3000)
- VITAS Processing Stack (2800x2200)
- VITAS Client Architecture (2600x1800)

### Tools
- Draw.io Desktop (recommended)
- xmllint for validation
- Screenshot tool for verification

### Documentation
- AWS Architecture Icons: mxgraph.aws4.*
- Draw.io XML format guide
- Color scheme reference

---

## 🆚 When to Use V1 vs V2

**Use V1 (Original) when**:
- Simple diagrams (< 15 components)
- Single-layer architectures
- Quick mockups or drafts
- Time-constrained situations

**Use V2 (Iterative) when**:
- Complex systems (20+ components)
- Multi-layer architectures
- Production-quality diagrams
- Need to show detailed flows
- Working with AI assistants (prevents errors)

---

## 💡 Pro Tips

1. **Start with paper sketch** - Draw rough layout first
2. **Count components** - Estimate canvas size needed
3. **Plan Y-coordinates** - Avoid overlaps from the start
4. **Use consistent spacing** - Pick values and stick to them
5. **Test frequently** - Open in Draw.io after each phase
6. **Save versions** - Keep backup before major changes
7. **Document as you go** - Update legend with each addition
8. **Ask for feedback** - Show user after key phases

---

## 🔧 Troubleshooting Guide

### Issue: XML Too Large Error
**Solution**: You're trying to create too much at once. Break into smaller phases.

### Issue: Components Overlapping
**Solution**: Check Y-coordinates. Ensure minimum 60px vertical gap between layers.

### Issue: Arrows Crossing Text
**Solution**: Use waypoints with `<Array as="points">` to route around components.

### Issue: Labels Cut Off
**Solution**: Increase label box width to 170px and height to 100px.

### Issue: Diagram Too Cramped
**Solution**: Increase canvas by 50% and add 100px to all gaps.

---

## 📈 Version History

- **V1.0** (Nov 12, 2025): Original prompt with spacing guidelines
- **V2.0** (Nov 12, 2025): Added iterative approach based on VITAS success
  - 9-phase development methodology
  - Checkpoint system after each phase
  - Lessons learned section
  - Troubleshooting guide
  - V1 vs V2 comparison

---

## 🎉 Success Story: VITAS Architecture

Using V2 iterative approach, we successfully created a comprehensive architecture diagram with:

- **50+ components** across 7 layers
- **7 Lambda functions** with detailed connections
- **7 DynamoDB tables** with keys and relationships
- **Complete auth flow** with JWT, SSM, CloudFront
- **Step Functions states** with retry logic
- **DLQ/SLA system** with 24-hour guarantee
- **External integrations** with fallback paths
- **Comprehensive legend** and timing documentation

**Result**: Clean, readable diagram with zero overlaps, completed in ~1.5 hours

---

**Remember**: When in doubt, build incrementally. Test often. Fix early. 🚀
