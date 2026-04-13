---
title: "Microsoft's AI Stack Decoded, Part 5: Multi-Agent Workflows"
date: 2026-04-13
categories: [AI, Microsoft, Architecture]
tags: [multi-agent, workflows, orchestration, human-in-loop, series]
description: "Orchestrating multiple AI agents for complex processes. Graph-based workflows, checkpointing, human approval gates, and production patterns."
image:
  path: /images/multi-agent-workflow.png
  alt: "Multi-Agent Workflow Architecture"
series: microsoft-ai-stack
---

## When One Agent Isn't Enough

A single agent can answer questions. But real enterprise processes need:

- Multiple specialized agents working together
- Explicit control over execution order
- Checkpoints to resume long-running processes
- Human approval at critical decision points
- Parallel execution for performance
- Error handling and rollback

This is where **workflows** come in.

This is Part 5 — the finale of the Microsoft AI Stack series.

---

## Graph-Based Workflows

Agent Framework uses a graph model for workflows. Each node is a function (AI-powered or not), and edges define the execution flow:

```python
from agent_framework import Workflow, Node

workflow = Workflow("ContractReview")

@workflow.node
async def extract_clauses(contract: Document) -> List[Clause]:
    """Extract all clauses from the contract."""
    return await extraction_agent.run(contract)

@workflow.node
async def assess_risk(clauses: List[Clause]) -> RiskReport:
    """Analyze clauses for risk factors."""
    return await risk_agent.run(clauses)

@workflow.node
async def generate_summary(risk_report: RiskReport) -> Summary:
    """Create executive summary."""
    return await summary_agent.run(risk_report)

# Define the graph
workflow.add_edge("extract_clauses", "assess_risk")
workflow.add_edge("assess_risk", "generate_summary")

# Run
result = await workflow.run(contract=uploaded_contract)
```

### Why Graphs?

Graphs give you:
- **Visibility** — See exactly what will execute
- **Debugging** — Know which node failed
- **Parallelism** — Independent nodes run concurrently
- **Checkpointing** — Resume from any node

---

## A Real Example: Procurement Review

Let's build a complete procurement document review workflow:

```python
from agent_framework import Workflow, Condition
from agents import (
    document_parser,
    compliance_checker,
    pricing_analyst,
    legal_reviewer,
    summary_writer
)

workflow = Workflow("ProcurementReview")

# ═══════════════════════════════════════════════════════════
# Stage 1: Parse and classify the document
# ═══════════════════════════════════════════════════════════

@workflow.node
async def parse_document(document: bytes, filename: str) -> ParsedDocument:
    """Extract text and structure from the procurement document."""
    return await document_parser.run(
        f"Parse this procurement document: {filename}",
        attachments=[document]
    )

@workflow.node
async def classify_document(parsed: ParsedDocument) -> DocumentType:
    """Determine document type: RFP, RFQ, Contract, Amendment, etc."""
    doc_type = await classifier_agent.run(
        f"Classify this document:\n{parsed.summary}"
    )
    return DocumentType(doc_type)

# ═══════════════════════════════════════════════════════════
# Stage 2: Parallel analysis (these run concurrently)
# ═══════════════════════════════════════════════════════════

@workflow.node
async def check_compliance(parsed: ParsedDocument) -> ComplianceReport:
    """Check FAR/DFAR compliance requirements."""
    return await compliance_checker.run(
        f"""Review for FAR/DFAR compliance:
        
        Key areas:
        - Required clauses (FAR 52.XXX)
        - Small business requirements
        - Cybersecurity requirements (CMMC if DoD)
        - Labor law compliance
        
        Document: {parsed.full_text}"""
    )

@workflow.node  
async def analyze_pricing(parsed: ParsedDocument) -> PricingAnalysis:
    """Analyze pricing structure and competitiveness."""
    return await pricing_analyst.run(
        f"""Analyze pricing:
        
        - Identify pricing model (FFP, T&M, Cost-Plus)
        - Compare to similar contracts (if available)
        - Flag unusual terms
        - Calculate total ceiling value
        
        Document: {parsed.full_text}"""
    )

@workflow.node
async def review_legal(parsed: ParsedDocument) -> LegalReview:
    """Review for legal risks and non-standard terms."""
    return await legal_reviewer.run(
        f"""Legal review:
        
        - Identify non-standard clauses
        - Flag liability concerns
        - Review IP provisions
        - Check termination clauses
        - Assess indemnification terms
        
        Document: {parsed.full_text}"""
    )

# ═══════════════════════════════════════════════════════════
# Stage 3: Human review gate
# ═══════════════════════════════════════════════════════════

@workflow.node(requires_approval=True)
async def human_review_gate(
    compliance: ComplianceReport,
    pricing: PricingAnalysis,
    legal: LegalReview
) -> ApprovalDecision:
    """Pause for human review of all findings."""
    
    # This creates a task in the review queue
    return {
        "compliance_issues": compliance.issues,
        "pricing_concerns": pricing.concerns,
        "legal_risks": legal.risks,
        "overall_risk": calculate_risk(compliance, pricing, legal),
        "recommendation": "Proceed" if all_clear else "Review Required"
    }

# ═══════════════════════════════════════════════════════════
# Stage 4: Generate final output
# ═══════════════════════════════════════════════════════════

@workflow.node
async def generate_summary(
    parsed: ParsedDocument,
    compliance: ComplianceReport,
    pricing: PricingAnalysis,
    legal: LegalReview,
    approval: ApprovalDecision
) -> FinalReport:
    """Generate executive summary with all findings."""
    return await summary_writer.run(
        f"""Create executive summary:
        
        Document: {parsed.summary}
        Compliance: {compliance.summary}
        Pricing: {pricing.summary}
        Legal: {legal.summary}
        Human Review: {approval.notes}
        
        Format as 1-page brief for procurement officer."""
    )

@workflow.node
async def notify_stakeholders(report: FinalReport) -> None:
    """Send notifications to relevant parties."""
    await email_service.send(
        to=report.stakeholders,
        subject=f"Procurement Review Complete: {report.document_name}",
        body=report.summary
    )

# ═══════════════════════════════════════════════════════════
# Define the graph
# ═══════════════════════════════════════════════════════════

# Sequential: parse → classify
workflow.add_edge("parse_document", "classify_document")

# Parallel: classify → [compliance, pricing, legal]
workflow.add_edge("classify_document", "check_compliance")
workflow.add_edge("classify_document", "analyze_pricing")
workflow.add_edge("classify_document", "review_legal")

# Converge: [compliance, pricing, legal] → human review
workflow.add_edge("check_compliance", "human_review_gate")
workflow.add_edge("analyze_pricing", "human_review_gate")
workflow.add_edge("review_legal", "human_review_gate")

# After approval: generate summary → notify
workflow.add_edge("human_review_gate", "generate_summary")
workflow.add_edge("generate_summary", "notify_stakeholders")
```

### Visual Representation

```
                    ┌─────────────────┐
                    │  parse_document │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ classify_document│
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
    ┌──────────────┐ ┌─────────────┐ ┌────────────┐
    │check_compliance│ │analyze_pricing│ │review_legal│
    └───────┬──────┘ └──────┬──────┘ └──────┬─────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │ human_review_gate 🧑 │ ← Pauses here
                  └──────────┬──────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ generate_summary │
                    └────────┬────────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │ notify_stakeholders │
                  └─────────────────────┘
```

---

## Human-in-the-Loop Patterns

### Approval Gates

```python
@workflow.node(requires_approval=True)
async def approve_purchase(amount: float, vendor: str) -> ApprovalResult:
    """Requires human approval for purchases over $10k."""
    return {
        "amount": amount,
        "vendor": vendor,
        "auto_approve": amount < 10000
    }

# Handle the approval
@workflow.on_approval("approve_purchase")
async def handle_approval(context, approved: bool, approver: str, notes: str):
    if approved:
        return context.continue_to("process_purchase")
    else:
        return context.continue_to("notify_rejection", reason=notes)
```

### Review Queues

Approvals appear in a review queue:

```python
# Get pending approvals
pending = await workflow.get_pending_approvals(
    assignee="procurement-officers@agency.gov"
)

for approval in pending:
    print(f"Workflow: {approval.workflow_id}")
    print(f"Node: {approval.node_name}")
    print(f"Data: {approval.context}")
    print(f"Waiting since: {approval.created_at}")

# Approve or reject
await workflow.resolve_approval(
    workflow_id="abc123",
    approved=True,
    approver="jsmith@agency.gov",
    notes="Approved per policy exception memo dated 4/10/26"
)
```

### Timeout and Escalation

```python
@workflow.node(
    requires_approval=True,
    timeout_hours=24,
    escalation_path=["supervisor@agency.gov", "director@agency.gov"]
)
async def urgent_approval(request: UrgentRequest) -> ApprovalResult:
    """Urgent requests escalate if not handled in 24 hours."""
    return request
```

---

## Checkpointing and Recovery

Long-running workflows need to survive failures:

```python
# Enable checkpointing
workflow = Workflow(
    "ProcurementReview",
    checkpoint_store=AzureBlobCheckpointStore(
        connection_string=os.environ["STORAGE_CONNECTION"]
    )
)

# Run with automatic checkpointing
run_id = await workflow.run(document=contract)

# Later: resume from checkpoint (after crash, restart, etc.)
result = await workflow.resume(run_id=run_id)
```

### Manual Checkpoints

```python
@workflow.node
async def long_analysis(data: LargeDataset) -> AnalysisResult:
    results = []
    
    for i, chunk in enumerate(data.chunks):
        result = await analyze_chunk(chunk)
        results.append(result)
        
        # Checkpoint every 100 chunks
        if i % 100 == 0:
            await workflow.checkpoint(
                progress=i,
                partial_results=results
            )
    
    return combine_results(results)
```

---

## Parallel Execution

Nodes without dependencies run concurrently:

```python
# These three run in parallel (no edges between them)
workflow.add_edge("classify", "check_compliance")
workflow.add_edge("classify", "analyze_pricing")
workflow.add_edge("classify", "review_legal")

# With explicit parallelism control
workflow = Workflow(
    "HighVolumeProcessing",
    max_parallel_nodes=10,  # Limit concurrent execution
    rate_limit_per_minute=100  # Throttle API calls
)
```

### Fan-Out / Fan-In

```python
@workflow.node
async def split_document(document: Document) -> List[Section]:
    """Split into sections for parallel processing."""
    return document.sections

@workflow.node(map_over="sections")
async def analyze_section(section: Section) -> SectionAnalysis:
    """Runs once per section, in parallel."""
    return await analyzer_agent.run(section)

@workflow.node
async def combine_analyses(analyses: List[SectionAnalysis]) -> FullAnalysis:
    """Combine all section analyses."""
    return FullAnalysis(sections=analyses)

# Fan-out: split → [analyze each]
# Fan-in: [all analyses] → combine
workflow.add_edge("split_document", "analyze_section")
workflow.add_edge("analyze_section", "combine_analyses")
```

---

## Conditional Routing

Route based on results:

```python
@workflow.node
async def assess_risk(document: Document) -> RiskLevel:
    """Determine risk level."""
    return await risk_agent.run(document)

@workflow.node
async def standard_review(document: Document) -> Review:
    """Standard review process."""
    return await standard_agent.run(document)

@workflow.node
async def enhanced_review(document: Document) -> Review:
    """Enhanced review for high-risk items."""
    return await enhanced_agent.run(document)

# Conditional routing
workflow.add_edge(
    "assess_risk",
    Condition(
        when=lambda result: result.level == "HIGH",
        then="enhanced_review",
        otherwise="standard_review"
    )
)
```

---

## Error Handling

```python
@workflow.node(
    retries=3,
    retry_delay_seconds=60,
    fallback="manual_processing"
)
async def call_external_api(data: InputData) -> APIResult:
    """Calls external API with retry logic."""
    return await external_api.call(data)

@workflow.node
async def manual_processing(data: InputData, error: Exception) -> APIResult:
    """Fallback when API fails."""
    # Create ticket for manual processing
    await create_support_ticket(data, error)
    return APIResult(status="pending_manual")
```

### Compensation (Rollback)

```python
@workflow.node(compensation="undo_reservation")
async def reserve_resource(resource_id: str) -> Reservation:
    """Reserve a resource."""
    return await resource_api.reserve(resource_id)

@workflow.compensation("reserve_resource")
async def undo_reservation(reservation: Reservation):
    """Release the reservation if workflow fails."""
    await resource_api.release(reservation.id)
```

---

## Observability

### Traces

Every workflow run generates traces:

```python
from opentelemetry import trace

# Traces automatically include:
# - Workflow start/end
# - Each node execution
# - Agent calls within nodes
# - Tool invocations
# - Human approval wait times
# - Errors and retries
```

### Metrics

```python
# Built-in metrics
workflow_duration_seconds
workflow_success_rate
node_execution_time_seconds
human_approval_wait_time_seconds
parallel_utilization_percent
checkpoint_size_bytes
```

### Dashboard

```
Workflow: ProcurementReview
Run ID: abc123
Status: ⏳ Waiting for human approval

Nodes:
  ✅ parse_document (2.3s)
  ✅ classify_document (1.1s)
  ✅ check_compliance (8.4s)
  ✅ analyze_pricing (6.2s)  
  ✅ review_legal (7.8s)
  ⏳ human_review_gate (waiting 4h 23m)
  ⬜ generate_summary
  ⬜ notify_stakeholders

Assigned to: procurement-officers@agency.gov
Escalation in: 19h 37m
```

---

## Production Patterns

### Pattern 1: Document Processing Pipeline

```python
# High-volume document processing
workflow = Workflow(
    "DocumentIngestion",
    max_parallel_nodes=20,
    checkpoint_interval_seconds=300
)

# Stages: Upload → Extract → Classify → Index → Notify
```

### Pattern 2: Approval Chain

```python
# Multi-level approval workflow
# Junior → Senior → Director (for high-value items)
workflow = Workflow("PurchaseApproval")

@workflow.node
async def route_by_amount(request: PurchaseRequest):
    if request.amount < 10000:
        return "junior_approval"
    elif request.amount < 100000:
        return "senior_approval"
    else:
        return "director_approval"
```

### Pattern 3: Scheduled Workflows

```python
# Daily report generation
from agent_framework.triggers import Schedule

@workflow.trigger(Schedule.daily(hour=6, tz="America/New_York"))
async def generate_daily_report():
    data = await fetch_daily_data()
    return await workflow.run(data=data)
```

---

## The Complete Picture

Over this series, we've covered:

1. **Part 1**: The full Microsoft AI stack — models to agents to governance
2. **Part 2**: Agent Framework — building individual agents with code
3. **Part 3**: Copilot Studio — building agents without code
4. **Part 4**: Governance — Content Safety, Purview, Defender, Entra
5. **Part 5**: Workflows — orchestrating agents for complex processes

Together, this is the platform for enterprise AI:

```
┌─────────────────────────────────────────────────────────┐
│                    Enterprise AI                         │
├─────────────────────────────────────────────────────────┤
│  Workflows (orchestration, human-in-loop)               │
├─────────────────────────────────────────────────────────┤
│  Agents (Copilot Studio + Agent Framework)              │
├─────────────────────────────────────────────────────────┤
│  Governance (Content Safety, Purview, Defender, Entra)  │
├─────────────────────────────────────────────────────────┤
│  Models (GPT, Phi, MAI, Florence, KOSMOS)               │
├─────────────────────────────────────────────────────────┤
│  Infrastructure (Azure AI Foundry)                       │
└─────────────────────────────────────────────────────────┘
```

Microsoft is the only vendor with the complete stack. That's the thesis.

---

*This concludes the "Microsoft AI Stack Decoded" series. If you found this valuable, share it with your team. The enterprises that understand this stack will move faster than those still assembling point solutions.*
