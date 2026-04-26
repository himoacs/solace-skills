---
name: solace-agent-mesh
description: 'Solace Agent Mesh (SAM) development guidance. Use for building AI agent orchestration frameworks, creating custom agents/tools/plugins, configuring gateways (REST, SSE, Slack, Teams), understanding A2A protocol, SAM CLI commands, and event-driven agentic AI patterns.'
argument-hint: 'agents, gateways, orchestrator, tools, or plugins'
---

# Solace Agent Mesh (SAM)

Expert guidance for developing with Solace Agent Mesh - an event-driven agentic AI framework for orchestrating autonomous AI agents.

## When to Use

- **Building AI Agents**: Creating custom agents with tools and capabilities
- **Orchestration**: Configuring workflows and task distribution
- **Gateway Development**: Setting up REST, SSE, Slack, Teams, or custom gateways
- **Plugin Development**: Creating reusable SAM plugins
- **A2A Protocol**: Understanding agent-to-agent communication patterns
- **SAM CLI**: Project setup, configuration, and deployment commands

## Core Concepts

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     External Systems                         │
│  (REST API, Slack, Teams, Web UI, Event Mesh)               │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                      Gateways                                │
│  Entry points that translate requests to A2A messages        │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                   Orchestrator Agent                         │
│  Analyzes requests, plans actions, distributes tasks         │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    Agent Mesh                                │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Agent A │  │ Agent B │  │ Agent C │  │ Agent D │        │
│  │ [Tools] │  │ [Tools] │  │ [Tools] │  │ [Tools] │        │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                 Solace Event Broker                          │
│  Event-driven backbone for agent communication               │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose |
|-----------|---------|
| **Orchestrator** | Breaks down requests into tasks, routes to appropriate agents |
| **Agents** | Autonomous units that expose tools/actions to the mesh |
| **Gateways** | Entry points from external systems (REST, Slack, etc.) |
| **Tools** | Functions that agents can execute |
| **Plugins** | Reusable packages of agents, gateways, and tools |
| **Proxies** | Intermediaries for external service calls |

## SAM CLI Commands

### Project Setup

```bash
# Install SAM CLI
pip install solace-agent-mesh

# Create new project
sam init my-agent-project
cd my-agent-project

# Start the agent mesh
sam run

# Run in development mode (auto-reload)
sam run --dev
```

### Adding Components

```bash
# Add a new agent
sam add agent my-agent

# Add a gateway with specific interface
sam add gateway rest-api --interface rest
sam add gateway slack-bot --interface slack

# Add a tool to an agent
sam add tool my-agent/search-tool

# Add a plugin
sam add plugin solace-ai-connector
```

### Configuration

```bash
# Validate configuration
sam validate

# Show current configuration
sam config show

# Set environment variables
sam config set OPENAI_API_KEY=sk-xxx
```

## Creating a Custom Agent

### Agent Structure

```
my-agent/
├── __init__.py
├── agent.py          # Agent class definition
├── config.yaml       # Agent configuration
└── tools/
    ├── __init__.py
    └── my_tool.py    # Tool implementations
```

### Agent Definition

```python
# my-agent/agent.py
from solace_agent_mesh import Agent, tool

class MyAgent(Agent):
    """
    Agent description - this appears in the mesh registry.
    """
    
    name = "my-agent"
    description = "Processes customer data and generates insights"
    
    # Agent configuration
    config = {
        "model": "gpt-4",
        "temperature": 0.7,
    }
    
    @tool
    def analyze_data(self, data: dict) -> dict:
        """
        Analyze input data and return insights.
        
        Args:
            data: Input data dictionary with customer information
            
        Returns:
            Analysis results with key metrics
        """
        # Tool implementation
        return {"insights": "processed"}
    
    @tool
    def generate_report(self, format: str = "pdf") -> str:
        """
        Generate a report in the specified format.
        
        Args:
            format: Output format (pdf, html, json)
            
        Returns:
            Path to generated report
        """
        return f"/reports/output.{format}"
```

### Agent Configuration

```yaml
# my-agent/config.yaml
name: my-agent
description: Processes customer data and generates insights

# LLM configuration
model:
  provider: openai
  name: gpt-4
  temperature: 0.7

# Tool access scopes
scopes:
  - data:read
  - reports:write

# Dependencies on other agents
depends_on:
  - data-fetcher
  - report-generator
```

## Gateway Configuration

### REST Gateway

```yaml
# gateways/rest-gateway/config.yaml
name: rest-gateway
type: rest

interface:
  host: 0.0.0.0
  port: 8080
  cors:
    enabled: true
    origins: ["*"]

# System prompt for requests through this gateway
system_purpose: |
  You are an API assistant. Process incoming requests
  and route them to appropriate agents.

# Output formatting
output_description: |
  Return JSON responses with status and data fields.

# Authentication
auth:
  type: bearer
  provider: oauth2
```

### Slack Gateway

```yaml
# gateways/slack-gateway/config.yaml
name: slack-gateway
type: slack

interface:
  bot_token: ${SLACK_BOT_TOKEN}
  signing_secret: ${SLACK_SIGNING_SECRET}
  
# Channels to listen on
channels:
  - ai-assistant
  - support

system_purpose: |
  You are a helpful Slack bot. Assist team members
  with their questions using available agents.
```

## A2A Protocol Messages

### Stimulus (Input Request)

```json
{
  "type": "stimulus",
  "session_id": "sess-12345",
  "gateway": "rest-gateway",
  "content": {
    "text": "Analyze sales data for Q4",
    "attachments": []
  },
  "context": {
    "user_id": "user-789",
    "scopes": ["data:read"]
  }
}
```

### Task (Orchestrator → Agent)

```json
{
  "type": "task",
  "task_id": "task-67890",
  "session_id": "sess-12345",
  "target_agent": "data-analyst",
  "action": "analyze_data",
  "parameters": {
    "dataset": "sales",
    "period": "Q4-2025"
  }
}
```

### Response (Agent → Orchestrator)

```json
{
  "type": "response",
  "task_id": "task-67890",
  "status": "completed",
  "result": {
    "insights": [...],
    "metrics": {...}
  }
}
```

## Best Practices

### Agent Design

1. **Single Responsibility**: Each agent should have a focused purpose
2. **Clear Tool Descriptions**: Detailed docstrings help the orchestrator route correctly
3. **Idempotent Tools**: Tools should be safe to retry
4. **Error Handling**: Return structured errors, don't throw exceptions
5. **Scopes**: Define minimal required permissions

### Gateway Design

1. **Authentication First**: Always configure auth for production
2. **Rate Limiting**: Protect against abuse
3. **Output Formatting**: Tailor output to the interface (Slack markdown vs JSON)
4. **System Purpose**: Be specific about the gateway's role

### Performance

1. **Async Tools**: Use async for I/O-bound operations
2. **Streaming**: Enable streaming for long responses
3. **Caching**: Cache expensive computations
4. **Timeouts**: Set appropriate timeouts for external calls

## Reference Links

- **Official Documentation**: https://solacelabs.github.io/solace-agent-mesh/
- **GitHub Repository**: https://github.com/SolaceLabs/solace-agent-mesh
- **Core Plugins**: https://github.com/SolaceLabs/solace-agent-mesh-core-plugins
- **Codelab**: https://codelabs.solace.dev/codelabs/solace-agent-mesh/ (88 min)

## Further Reading

- [Gateway Development](./references/gateways.md)
- [Plugin Development](./references/plugins.md)
- [Orchestrator Configuration](./references/orchestrator.md)
- [Enterprise Features](./references/enterprise.md)
