---
title: "Microsoft's AI Stack Decoded, Part 2: The Agent Framework"
date: 2026-04-13
categories: [AI, Microsoft, Development]
tags: [agent-framework, semantic-kernel, autogen, python, dotnet, series]
description: "A deep dive into Microsoft Agent Framework 1.0 — the production-ready merger of Semantic Kernel and AutoGen. Real code, real patterns."
image:
  path: /images/agent-framework-architecture.png
  alt: "Microsoft Agent Framework Architecture"
series: microsoft-ai-stack
---

## The Framework War is Over

For two years, Microsoft ran two competing agent frameworks:

- **Semantic Kernel** — Enterprise-focused, .NET-first, built for production
- **AutoGen** — Research-focused, Python-first, built for multi-agent experiments

Developers had to choose. Enterprises had to bet. It was confusing.

That's over now. **Agent Framework 1.0** merges both into a single, production-ready platform for building AI agents in Python and .NET.

This is Part 2 of the Microsoft AI Stack series. Let's write some code.

---

## Getting Started

### Python

```bash
pip install agent-framework
```

```python
from agent_framework.foundry import FoundryChatClient
from azure.identity import DefaultAzureCredential

# Connect to Azure AI Foundry
client = FoundryChatClient(
    project_endpoint="https://your-project.services.ai.azure.com/api/projects/your-project",
    model="gpt-5.1",
    credential=DefaultAzureCredential()
)

# Create an agent
agent = client.as_agent(
    name="PolicyAnalyst",
    instructions="You help federal employees understand policy. Be concise and cite sources."
)

# Run it
result = await agent.run("What are the key changes in the 2026 FISMA update?")
print(result)
```

### .NET

```bash
dotnet add package Microsoft.Agents.AI.Foundry --prerelease
```

```csharp
using Microsoft.Agents.AI;
using Azure.AI.Projects;
using Azure.Identity;

var agent = new AIProjectClient(
        new Uri("https://your-project.services.ai.azure.com/api/projects/your-project"),
        new DefaultAzureCredential())
    .AsAIAgent(
        model: "gpt-5.1",
        instructions: "You help federal employees understand policy. Be concise and cite sources.");

var result = await agent.RunAsync("What are the key changes in the 2026 FISMA update?");
Console.WriteLine(result);
```

That's it. A working agent in under 10 lines.

---

## Adding Tools

Agents become powerful when they can take actions. Here's how to add tools:

### Python

```python
from agent_framework import Agent, tool

@tool
def search_regulations(query: str, agency: str = "all") -> str:
    """Search federal regulations by keyword and agency."""
    # Your search implementation
    results = regulations_api.search(query, agency)
    return format_results(results)

@tool
def get_document(document_id: str) -> str:
    """Retrieve a specific federal document by ID."""
    return documents_api.get(document_id)

agent = client.as_agent(
    name="RegulationExpert",
    instructions="""You help users navigate federal regulations.
    Use search_regulations to find relevant rules.
    Use get_document to retrieve full text when needed.
    Always cite CFR references.""",
    tools=[search_regulations, get_document]
)

result = await agent.run(
    "What are the cybersecurity requirements for federal contractors?"
)
```

### .NET

```csharp
[Tool("search_regulations")]
[Description("Search federal regulations by keyword and agency")]
public static string SearchRegulations(
    [Description("Search query")] string query,
    [Description("Agency filter")] string agency = "all")
{
    var results = RegulationsApi.Search(query, agency);
    return FormatResults(results);
}

var agent = client.AsAIAgent(
    model: "gpt-5.1",
    instructions: "You help users navigate federal regulations.",
    tools: [typeof(SearchRegulations), typeof(GetDocument)]);
```

The agent automatically decides when to use tools based on the conversation.

---

## MCP Integration

Agent Framework has first-class support for [Model Context Protocol (MCP)](https://modelcontextprotocol.io) — the emerging standard for AI tool integration:

```python
from agent_framework.tools import MCPTool

# Connect to an MCP server
policy_tools = MCPTool.from_server("https://mcp-analyst.turek.in/mcp")

agent = client.as_agent(
    name="PolicyAdvisor",
    instructions="You advise on federal IT policy.",
    tools=[policy_tools]  # All tools from the MCP server
)
```

This means you can:
- Connect to any MCP-compatible tool server
- Use Microsoft's hosted MCP tools (mail, calendar, SharePoint)
- Build your own MCP servers and share them

---

## Multi-Turn Conversations with Sessions

Real applications need conversation history. Sessions handle this:

```python
async with agent.session() as session:
    # First turn
    response1 = await session.run("What's FedRAMP?")
    print(f"Agent: {response1}")
    
    # Second turn - agent remembers context
    response2 = await session.run("How long does the authorization process take?")
    print(f"Agent: {response2}")
    
    # Third turn - still in context
    response3 = await session.run("What about for a SaaS product specifically?")
    print(f"Agent: {response3}")
```

Sessions persist conversation state. You can also save and restore sessions:

```python
# Save session state
state = await session.export_state()
save_to_database(user_id, state)

# Later: restore the session
saved_state = load_from_database(user_id)
async with agent.session(state=saved_state) as session:
    # Continues where they left off
    response = await session.run("What was that timeline again?")
```

---

## Middleware: Intercept Everything

Middleware lets you intercept agent actions — for logging, rate limiting, content filtering, or custom logic:

```python
from agent_framework import Middleware

class AuditMiddleware(Middleware):
    async def on_request(self, request, context):
        # Log every request
        logger.info(f"User {context.user_id}: {request.content}")
        return request
    
    async def on_response(self, response, context):
        # Log every response
        logger.info(f"Agent response: {response.content[:100]}...")
        
        # Save to audit trail
        audit_db.insert({
            "user": context.user_id,
            "request": context.request.content,
            "response": response.content,
            "timestamp": datetime.now(),
            "model": context.model,
            "tokens": response.usage.total_tokens
        })
        return response

class ContentFilterMiddleware(Middleware):
    async def on_response(self, response, context):
        # Check content safety
        safety_result = await content_safety.analyze(response.content)
        if safety_result.is_harmful:
            return Response("I can't provide that information.")
        return response

# Apply middleware
agent = client.as_agent(
    name="SecureAgent",
    instructions="...",
    middleware=[AuditMiddleware(), ContentFilterMiddleware()]
)
```

This is how you build enterprise-grade agents — logging, compliance, and safety baked into every interaction.

---

## Multi-Agent Workflows

When a single agent isn't enough, you need workflows. Agent Framework uses a graph-based approach:

```python
from agent_framework import Workflow, Node

# Define specialized agents
researcher = client.as_agent(
    name="Researcher",
    instructions="You research topics thoroughly and gather facts.",
    tools=[web_search, document_search]
)

analyst = client.as_agent(
    name="Analyst",
    instructions="You analyze information and identify key insights.",
)

writer = client.as_agent(
    name="Writer",
    instructions="You write clear, concise executive summaries.",
)

# Define the workflow
workflow = Workflow("ResearchReport")

@workflow.node
async def research(topic: str) -> dict:
    """Gather information on the topic."""
    facts = await researcher.run(f"Research this topic thoroughly: {topic}")
    return {"facts": facts, "topic": topic}

@workflow.node
async def analyze(facts: str, topic: str) -> dict:
    """Analyze the research findings."""
    analysis = await analyst.run(f"Analyze these findings about {topic}: {facts}")
    return {"analysis": analysis, "topic": topic}

@workflow.node
async def write(analysis: str, topic: str) -> str:
    """Write the final report."""
    report = await writer.run(
        f"Write a 1-page executive summary about {topic} based on: {analysis}"
    )
    return report

# Connect the nodes
workflow.add_edge("research", "analyze")
workflow.add_edge("analyze", "write")

# Run it
report = await workflow.run(topic="Impact of AI on federal workforce")
print(report)
```

### Human-in-the-Loop

For sensitive workflows, you need human approval:

```python
@workflow.node(requires_approval=True)
async def submit_for_review(report: str) -> dict:
    """Submit report for human review before publishing."""
    # Workflow pauses here until human approves
    return {"report": report, "status": "pending_review"}

@workflow.on_approval("submit_for_review")
async def handle_approval(result, approved: bool, feedback: str):
    if approved:
        await publish_report(result["report"])
    else:
        # Route back to writer with feedback
        return workflow.goto("write", feedback=feedback)
```

---

## Streaming Responses

For real-time UX, stream responses as they generate:

```python
async for chunk in agent.run_stream("Explain zero trust architecture"):
    print(chunk, end="", flush=True)
```

Or with full event details:

```python
async for event in agent.run_stream("Explain zero trust architecture"):
    if event.type == "text":
        print(event.content, end="")
    elif event.type == "tool_call":
        print(f"\n[Calling {event.tool_name}...]")
    elif event.type == "tool_result":
        print(f"[Got result from {event.tool_name}]")
```

---

## Telemetry and Observability

Agent Framework integrates with OpenTelemetry out of the box:

```python
from opentelemetry import trace
from agent_framework.telemetry import configure_telemetry

# Configure once at startup
configure_telemetry(
    service_name="policy-agent",
    export_endpoint="https://your-collector.com"
)

# Now all agent calls are automatically traced
agent = client.as_agent(...)

# Each run creates spans with:
# - Request/response content
# - Token usage
# - Tool calls
# - Latency breakdown
# - Error details
```

This is critical for production — you need to know why the agent did what it did.

---

## Putting It All Together

Here's a production-ready agent service:

```python
from fastapi import FastAPI, Depends
from agent_framework import Agent
from agent_framework.telemetry import configure_telemetry

app = FastAPI()

# Configure telemetry
configure_telemetry(service_name="federal-policy-agent")

# Create agent at startup
@app.on_event("startup")
async def startup():
    global agent
    agent = FoundryChatClient(...).as_agent(
        name="FederalPolicyAdvisor",
        instructions=POLICY_INSTRUCTIONS,
        tools=[regulation_search, document_lookup, acronym_lookup],
        middleware=[AuditMiddleware(), ContentFilterMiddleware(), RateLimitMiddleware()]
    )

# Session management
def get_session_store():
    return SessionStore(redis_client)

@app.post("/chat")
async def chat(
    message: str,
    session_id: str,
    store: SessionStore = Depends(get_session_store)
):
    # Load or create session
    state = await store.get(session_id)
    
    async with agent.session(state=state) as session:
        response = await session.run(message)
        
        # Save updated state
        await store.set(session_id, await session.export_state())
        
        return {"response": response}

@app.get("/health")
async def health():
    return {"status": "healthy"}
```

Deploy this to Azure Container Apps, add an API gateway, and you have a production agent service.

---

## Migration Guides

### From Semantic Kernel

If you're coming from Semantic Kernel, the concepts map directly:

| Semantic Kernel | Agent Framework |
|-----------------|-----------------|
| `Kernel` | `Agent` |
| `KernelFunction` | `@tool` decorator |
| `ChatCompletionService` | `FoundryChatClient` |
| `KernelArguments` | Function parameters |
| Plugins | Tool collections |

```python
# Semantic Kernel (old)
kernel = Kernel()
kernel.add_plugin(MyPlugin())
result = await kernel.invoke_prompt("...")

# Agent Framework (new)
agent = client.as_agent(tools=[my_tools])
result = await agent.run("...")
```

### From AutoGen

If you're coming from AutoGen, multi-agent patterns are now workflows:

```python
# AutoGen (old)
agent1 = AssistantAgent(name="researcher", ...)
agent2 = AssistantAgent(name="writer", ...)
groupchat = GroupChat(agents=[agent1, agent2], ...)

# Agent Framework (new)
workflow = Workflow("research_workflow")
# Define nodes and edges explicitly
```

The explicit workflow graph gives you more control and observability than the implicit AutoGen approach.

---

## What's Next

That's Agent Framework in depth. In Part 3, we'll look at **Copilot Studio** — building production agents without writing any code. For many enterprise scenarios, that's actually the right choice.

---

*This is Part 2 of the "Microsoft AI Stack Decoded" series. [Part 1](/posts/microsoft-ai-stack-part-1-the-full-picture/) covers the full platform overview.*
