# LangChain + AgentCore Observability Setup


LangChain/LangGraph agents deployed on AgentCore don't appear in the CloudWatch GenAI Observability dashboard by default because:
- The dashboard requires OTEL spans flowing through the ADOT collector
- LangChain doesn't auto-instrument without an explicit instrumentor
- Azure OpenAI calls bypass Bedrock SDK (no automatic interception)

## Solution

### 1. Add dependencies to `requirements.txt`
Be careful about versions
```
aws-opentelemetry-distro        # AgentCore's ADOT collector picks this up
opentelemetry-instrumentation-langchain  # Auto-instruments LangGraph/LangChain calls
```

### 2. Instrument at startup in `agent.py`
```python
from opentelemetry.instrumentation.langchain import LangchainInstrumentor
from opentelemetry import baggage, context

LangchainInstrumentor().instrument()  # Must be called before agent is created
```

### 3. Set session baggage in the entrypoint
```python
@app.entrypoint
def invoke(payload):
    session_id = payload.get("session_id", "default-session")
    ctx = baggage.set_baggage("session.id", session_id)
    token = context.attach(ctx)
    try:
        # ... agent logic
    finally:
        context.detach(token)
```

The `session.id` baggage is what populates the **Sessions View** in the dashboard.

