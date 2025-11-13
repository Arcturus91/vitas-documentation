# Prompt for Creating Spacious Draw.io Architecture Diagrams

## System Prompt

Your task is to review the architecture we provide you, understand it in detail, understand every flow otherwise ask the user. Then you will work as an AWS architect creating excellent AWS architecture diagrams.

### Reference Examples

Study these well-spaced AWS architecture diagrams as references for proper spacing and layout:

#### Example 1: Load Testing Architecture
![Load Testing Architecture](reference_images/example_1_load_testing.png)

**Key features to note:**
- Clear separation between frontend, backend, and regional resources
- Components grouped by logical layers (orange borders for each section)
- AWS icons are clearly spaced with readable labels
- Numbered flow markers (1-14) show data flow sequence
- Adequate white space between all components

#### Example 2: VPC Architecture with Processing Pipeline
![VPC Architecture](reference_images/example_2_vpc_architecture.png)

**Key features to note:**
- Multi-column layout with distinct sections
- Security groups clearly marked in red borders
- Public and private subnets visually separated
- Load balancers (ALB) positioned logically
- Sensor processing pipeline shows clear data flow
- External users and authorized clients positioned outside VPC

#### Example 3: Event-Driven Microservice Architecture
![Event-Driven Architecture](reference_images/example_3_eventbridge_microservice.png)

**Key features to note:**
- EventBridge scheduler as orchestration layer
- Lambda functions with DLQ (Dead Letter Queue) pattern
- Clear separation between business logic and AWS services
- Security and monitoring services grouped on the right
- Spanish labels demonstrating internationalization support
- Vertical flow from top to bottom

### Design Principles from Examples

From these references, follow these patterns:

1. **Layered Architecture**: Group components by function (Frontend, Backend, Infrastructure)
2. **Color-Coded Sections**: Use border colors to distinguish different layers
3. **Logical Flow**: Arrange components to show data flow naturally (top-to-bottom or left-to-right)
4. **Grouping**: Related services should be visually grouped together
5. **Clear Labels**: Every component should have a descriptive label
6. **Flow Markers**: Use numbered circles to indicate sequence of operations
7. **White Space**: Generous spacing prevents visual clutter

### Setting Up Reference Images

To use the reference images:

1. Create a `reference_images` folder in the same directory as this document
2. Save the three example images with these names:
   - `example_1_load_testing.png`
   - `example_2_vpc_architecture.png`
   - `example_3_eventbridge_microservice.png`
3. The markdown will automatically reference these images
4. When asking AI to create diagrams, reference these examples by name

**Alternative**: If using this prompt in a new session without images, describe the architecture patterns:
- "Multi-layer architecture like Example 1"
- "VPC security group pattern like Example 2"
- "Event-driven orchestration like Example 3"

## Context

When creating AWS architecture diagrams in Draw.io XML format, the default spacing often results in cramped, overlapping text and components that are difficult to read. This prompt provides specific guidelines to create properly spaced, professional-looking architecture diagrams.

## Core Problem Solved

Initial attempts at spacing improvements (50% increases) were insufficient. The solution requires **2-3x more space** than typical diagrams, with specific techniques to prevent text overlap.

## Step-by-Step Instructions

### 1. Canvas Size - Make it LARGE

```
- Minimum canvas size: 2600x1800 (for client diagrams)
- For processing stack diagrams: 2800x2200
- Rule of thumb: 85-100% larger than your initial estimate
- Use pageWidth and pageHeight attributes in mxGraphModel
```

### 2. CRITICAL: Separate Icons from Labels

This is the #1 fix for text overlap:

```xml
<!-- WRONG - Text embedded in icon -->
<mxCell id="lambda" value="Lambda\nFunction\n512MB\n5min"
  style="...shape=mxgraph.aws4.resourceIcon;resIcon=mxgraph.aws4.lambda;"/>

<!-- CORRECT - Empty icon + separate label -->
<mxCell id="lambda" value=""
  style="...shape=mxgraph.aws4.resourceIcon;resIcon=mxgraph.aws4.lambda;"/>
<mxCell id="lambda-label" value="Lambda Function&#xa;&#xa;512MB / 5min&#xa;OpenAI API"
  style="text;html=1;align=center;verticalAlign=top;whiteSpace=wrap;fontSize=12;">
  <mxGeometry x="110" y="280" width="160" height="100" as="geometry" />
</mxCell>
```

Key points:

- Icons should be **empty** (value="")
- Create separate text boxes for labels
- Position labels **below** icons with 10-20px gap
- Label boxes should be **120-160px wide** and **80-120px tall**

### 3. Component Sizing Guidelines

#### AWS Icons:

```
- Small icons (Lambda, SQS, etc.): 70x80 (not 50x60)
- Medium icons (S3, DynamoDB): 80x80 (not 60x60)
- Large icons: 100x100
```

#### UI Component Boxes:

```
- API route boxes: 240x120 (not 130x70)
- Frontend components: 260x150 (not 140x80)
- State management boxes: 240x110 (not 120x60)
- Note/description boxes: 480x80 minimum (not 260x40)
```

#### External Services (circles):

```
- Service circles: 140x140 diameter (not 80x80)
```

### 4. Layer/Section Heights

Each major section container should have these MINIMUM heights:

```
Frontend Layer: 300px
State Management: 220px
API Routes Layer: 720px
Lambda Functions: 300px
Step Functions Workflows: 500px
Database Layer: 500px
Error Handling/DLQ: 450px
External Services: 300px
Data Flow Patterns: 360px
```

**Never use less than these values** - go higher if needed.

### 5. Vertical Spacing Between Layers

```
- Gap between major sections: 60-100px minimum
- Gap between title and first section: 80px
- Gap between components within a section: 40-60px
- Gap between icon and its label: 10px
```

Example positioning:

```
Title: y=30
Frontend Layer: y=180 (150px gap from title area)
State Management: y=540 (60px gap)
API Routes: y=820 (60px gap)
AWS Infrastructure: y=120 (on right side)
```

### 6. Horizontal Spacing

```
- Between major columns: 200-280px
- Between components in same row: 240-320px
- Between icon and side container edge: 60-80px
- Between text box and container edge: 40-60px
```

Example:

```
Component 1: x=160
Component 2: x=480 (320px gap)
Component 3: x=800 (320px gap)
```

### 7. Text Formatting Best Practices

```xml
<!-- Use line breaks for readability -->
<mxCell value="Component Name&#xa;&#xa;• Feature 1&#xa;• Feature 2&#xa;• Feature 3"
  style="...fontSize=12;spacingTop=10;spacingLeft=12;align=left;verticalAlign=top;"/>
```

Key rules:

- Use `&#xa;` for line breaks (not `\n`)
- Double line breaks (`&#xa;&#xa;`) between sections
- Font sizes: 11-13px for descriptions, 14-18px for headers, 32px for main title
- Always include: `spacingTop=10;spacingLeft=12;align=left;verticalAlign=top;`
- Use bullet points (•) for lists

### 8. Color Scheme (Keep Consistent)

```
Frontend/UI Layer: fillColor=#dae8fc;strokeColor=#6c8ebf (light blue)
State Management: fillColor=#fff2cc;strokeColor=#d6b656 (light yellow)
API Routes/Lambda: fillColor=#f8cecc;strokeColor=#b85450 (light pink)
Database: fillColor=#e1d5e7;strokeColor=#9673a6 (light purple)
Infrastructure: fillColor=#d5e8d4;strokeColor=#82b366 (light green)
External Services: fillColor=#ffe6cc;strokeColor=#d79b00 (light orange)
Data Flow: fillColor=#f5f5f5;strokeColor=#666666 (light gray)
```

### 9. Connection Arrows

```xml
<mxCell style="edgeStyle=orthogonalEdgeStyle;rounded=0;orthogonalLoop=1;
  jettySize=auto;html=1;strokeWidth=3;strokeColor=#0066CC;"/>
```

Rules:

- Use `strokeWidth=3` for main flows (not 2)
- Use `strokeWidth=2` for secondary/dashed flows
- Add `dashed=1` for async/indirect connections
- Color code by flow type:
  - Auth: #0066CC (blue)
  - Upload: #00CC00 (green)
  - Webhook: #CC0000 (red)
  - Results: #9933FF (purple)
  - Error: #FF0000 (red dashed)

### 10. Complete Example Section

```xml
<!-- ========== LAMBDA FUNCTIONS LAYER ========== -->
<mxCell id="lambda-layer" value="LAMBDA FUNCTIONS - Processing Layer (7 Functions - Node.js 20.x ARM64)"
  style="rounded=0;whiteSpace=wrap;html=1;fillColor=#f8cecc;strokeColor=#b85450;
  verticalAlign=top;fontSize=16;fontStyle=1;spacingTop=15;" vertex="1" parent="aws-cloud">
  <mxGeometry x="60" y="1120" width="2200" height="300" as="geometry" />
</mxCell>

<!-- Empty icon -->
<mxCell id="transcription-lambda" value=""
  style="sketch=0;...shape=mxgraph.aws4.resourceIcon;resIcon=mxgraph.aws4.lambda;"
  vertex="1" parent="aws-cloud">
  <mxGeometry x="140" y="1200" width="70" height="70" as="geometry" />
</mxCell>

<!-- Separate label with spacing -->
<mxCell id="transcription-label" value="Transcription&#xa;&#xa;1024MB / 15min&#xa;OpenAI Whisper&#xa;AWS Transcribe"
  style="text;html=1;align=center;verticalAlign=top;whiteSpace=wrap;fontSize=11;"
  vertex="1" parent="aws-cloud">
  <mxGeometry x="110" y="1280" width="130" height="100" as="geometry" />
</mxCell>
```

### 11. Validation Steps

After creating the diagram:

```bash
# 1. Validate XML syntax
xmllint --noout your-diagram.drawio

# 2. Check if XML is well-formed
echo $?  # Should output 0

# 3. Open in Draw.io and verify:
#    - No text overlap
#    - All components clearly separated
#    - Easy to read at 100% zoom
#    - No need to zoom in to read labels
```

### 12. Common Mistakes to Avoid

❌ **DON'T:**

- Put multi-line text directly on AWS icons
- Use default spacing (50% increase is not enough)
- Make canvas smaller than 2400x1600
- Use font sizes below 10px
- Position components closer than 200px apart
- Stack layers with less than 60px vertical gap

✅ **DO:**

- Separate all icon labels into text boxes
- Increase canvas by 85-100%
- Use 2-3x normal spacing
- Keep fonts 11-18px (except title at 32px)
- Test by viewing at 100% zoom - everything should be readable
- Add `spacingTop` and `spacingLeft` to all text elements

### 13. Quick Reference Checklist

Before submitting diagram:

- [ ] Canvas is at least 2600x1800
- [ ] All AWS icons are empty (value="")
- [ ] All icons have separate label text boxes below them
- [ ] Label boxes are 120-160px wide
- [ ] Vertical gaps between sections are 60-100px
- [ ] Horizontal spacing between components is 240-320px
- [ ] Component boxes are at least 240x120
- [ ] Layer containers are at least 300px tall
- [ ] Font sizes are 11-18px (32px for title)
- [ ] XML validates with xmllint
- [ ] All text is readable at 100% zoom in Draw.io
- [ ] No text overlap anywhere
- [ ] Connection arrows are visible and not cluttered

### 14. Troubleshooting Text Overlap

If you still see overlap after following above:

1. **Icon labels overlapping workflow descriptions:**

   - Move icons 40px further from edges
   - Increase label box width to 160px
   - Add extra `&#xa;` line break before text

2. **Section headers getting cut off:**

   - Increase section height by 50px
   - Add `spacingTop=15` to section header style

3. **Arrows obscuring text:**
   - Add more waypoints to connection paths
   - Use `<Array as="points">` to route around text
   - Increase `jettySize` parameter

### 15. Final Formula

```
Minimum Canvas Width = (Number of horizontal components × 300px) + 400px
Minimum Canvas Height = (Number of vertical sections × 350px) + 200px

Minimum Layer Height = (Number of components × 150px) + 100px
Component Spacing = Component Width + 80px minimum
```

## Example Command for AI Assistant

When asking an AI to create these diagrams, use this prompt template:

```
Create a Draw.io XML architecture diagram for [SYSTEM_NAME] with the following requirements:

REFERENCE STYLE:
Study the three reference examples in this document:
- Example 1: Load Testing Architecture (multi-layer with numbered flows)
- Example 2: VPC Architecture (security groups, public/private subnets)
- Example 3: Event-Driven Microservice (EventBridge orchestration pattern)

Match the spacing, grouping, and visual clarity of these examples.

CRITICAL SPACING REQUIREMENTS:
1. Canvas size: minimum 2600x1800 (make it 85-100% larger than normal)
2. ALL AWS icons must be empty (value="") with separate text labels below them
3. Icon-to-label gap: 10px
4. Label box size: 160px wide × 100px tall minimum
5. Component spacing: 240-320px horizontally, 60-100px vertically between sections
6. Component box sizes: 240x120 minimum for API routes, 260x150 for UI components
7. Layer heights: 300px minimum (never less)
8. Font sizes: 11-13px body, 16-18px headers, 32px title
9. Use &#xa;&#xa; for double line breaks between sections
10. Validate with xmllint before finalizing
11. Add numbered flow markers (1, 2, 3...) to show data flow sequence
12. Use color-coded borders to group related components

COMPONENTS TO INCLUDE:
[List your components here]

ARCHITECTURE FLOW:
[Describe the data flow]

DESIGN PATTERNS TO FOLLOW:
- Group components by logical layers (Frontend, Backend, Infrastructure)
- Use color-coded section borders (orange, blue, green, etc.)
- Add numbered circles to show operation sequence
- Position external services/users outside main architecture boundary
- Use dashed lines for async/event-driven flows
- Use solid lines for synchronous API calls

Use AWS4 icon library (mxgraph.aws4.*) and follow the spacing checklist exactly.
If text overlaps at any point, increase spacing by 50% more until resolved.
```

---

## Notes from Implementation

During our iterations, we learned:

- 50% spacing increase was NOT enough (diagram still cramped)
- Separating icon labels was THE KEY breakthrough
- Testing with actual screenshot revealed issues invisible in code
- Canvas needed to be 85-100% larger, not just 20-30%
- Every dimension needed 2-3x increase, not just selective ones

This prompt codifies all those learnings for future replication.

---

## Real-World Examples

### Example 1: VITAS Processing Stack Architecture

- **Canvas**: 2800x2200
- **Layers**: 7 major sections (S3, Workflows, Lambda, DLQ, Database, External, Summary)
- **Components**: 25+ AWS services
- **Result**: Clear, readable diagram with no overlap

**Key measurements used:**

```
S3 Layer: 450px height
Workflow Layer: 500px height
Lambda Layer: 300px height
Icon spacing: 240px horizontally
Label boxes: 130-160px wide
```

### Example 2: VITAS Client Architecture

- **Canvas**: 2600x1800
- **Layers**: 8 major sections (Frontend, State, API Routes, AWS, External Services, Data Flow)
- **Components**: 30+ API routes, 6 AWS services, 4 external integrations
- **Result**: Professional, spacious layout

**Key measurements used:**

```
Frontend Layer: 300px height
API Routes Layer: 720px height
AWS Infrastructure: 1000px height
Component spacing: 320px horizontally
API boxes: 240x120 each
```

---

## Additional Tips

### For Multi-Column Layouts

When placing components in multiple columns (e.g., left side for frontend, right side for AWS):

```
Left Column:
- X position: 100-1200
- Components spaced 320px apart

Right Column:
- X position: 1340-2440
- Start with 60px gap from left column boundary
- Components spaced 280px apart
```

### For Dense API Route Sections

When you have many API routes (20-30 endpoints):

```
- Group by category (Auth, Upload, Webhook, etc.)
- 4 columns maximum
- Each route box: 240x120
- Vertical stacking: 170px between rows (120px box + 50px gap)
- Horizontal spacing: 280px between columns
```

### For Connection Arrows

To avoid spaghetti diagrams:

```
- Limit connections to essential flows only
- Use waypoints to route around components
- Color code by flow type
- Use dashed lines for secondary/async flows
- Add numbered markers (1, 2, 3) to show sequence
```

---

## Version History

- **v1.0** (2025-11-12): Initial documentation based on VITAS Clinic architecture diagrams
- Captures learnings from multiple iterations
- Tested with both client and processing stack architectures
- Validated with Draw.io desktop application

---

## Quick Start Template

Copy this as a starting point:

```xml
<mxfile host="app.diagrams.net" agent="Claude Code" version="24.0.0">
  <diagram id="Architecture" name="System Architecture">
    <mxGraphModel dx="3500" dy="2500" grid="1" gridSize="10" guides="1" tooltips="1"
      connect="1" arrows="1" fold="1" page="1" pageScale="1"
      pageWidth="2600" pageHeight="1800" math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />

        <!-- Title -->
        <mxCell id="title-line" value=""
          style="line;strokeWidth=3;html=1;fontSize=14;shadow=0;sketch=0;"
          vertex="1" parent="1">
          <mxGeometry x="60" y="80" width="2400" height="10" as="geometry" />
        </mxCell>

        <mxCell id="title-text" value="YOUR SYSTEM ARCHITECTURE"
          style="text;html=1;strokeColor=none;fillColor=none;align=left;
          verticalAlign=middle;whiteSpace=wrap;rounded=0;fontSize=32;
          fontColor=#000000;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="60" y="30" width="1200" height="50" as="geometry" />
        </mxCell>

        <!-- Add your layers and components here -->

      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

Remember: **When in doubt, add more space!**
