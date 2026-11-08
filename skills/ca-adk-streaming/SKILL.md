---
name: ca-adk-streaming
description: This skill enables agents to assist users in building streaming, data-driven agentic applications using the Looker Conversational Analytics (CA) API and Google ADK. Use this when you need to orchestrate multi-step data workflows with real-time feedback and conditional post-processing (e.g., visualization, analysis).
---

# Looker CA API with Google ADK Streaming

This skill provides guidance for integrating the **Looker Conversational Analytics (CA) API** with the **Google Agent Development Kit (ADK)** to build sophisticated, streaming data agents.

## Overview
By combining the CA API's natural language-to-query capabilities with ADK's orchestration and streaming features, you can create applications that:
- **Stream Real-time Responses**: Provide immediate feedback as the CA API processes and generates data.
- **Orchestrate Multi-step Workflows**: Chain data fetching with analysis, visualization, or external tool calls.
- **Implement Conditional Logic**: Use `run_when` predicates to execute sub-agents only when specific data conditions are met.
- **Normalize Session State**: Share data between agents using ADK's `session.state`.

## 1. Prerequisites
- **Looker Instance**: Configured with the CA API enabled and appropriate IAM roles.
- **Google ADK**: Installed (`pip install google-adk`).
- **GCP Project**: Vertex AI and Gemini Data Analytics APIs enabled.
- **Environment Variables**: `LOOKERSDK_BASE_URL`, `LOOKERSDK_CLIENT_ID`, `LOOKERSDK_CLIENT_SECRET`, `GCP_PROJECT_ID`, `GCP_LOCATION`.

## 2. Implementation Pattern: CA Query Agent
The `ConversationalAnalyticsQueryAgent` bridges the CA API with ADK's streaming interface.

```python
from google.adk.agents import BaseAgent, InvocationContext
from google.cloud import geminidataanalytics_v1beta as geminidataanalytics

class ConversationalAnalyticsQueryAgent(BaseAgent):
    def run(self, input_text: str, context: InvocationContext):
        # 1. Initialize CA API Client
        client = geminidataanalytics.DataChatServiceClient()
        
        # 2. Build Chat Request (Inline or Stateful)
        request = geminidataanalytics.ChatRequest(
            parent=f"projects/{PROJECT_ID}/locations/{LOCATION}",
            messages=[geminidataanalytics.Message(
                user_message=geminidataanalytics.UserMessage(text=input_text)
            )],
            inline_context=self._build_inline_context()
        )

        # 3. Stream from CA API and Bridge to ADK
        stream = client.chat(request=request)
        for response in stream:
            # Extract text/data from response
            text_part = response.message.text
            if text_part:
                context.stream(text_part) # Stream to ADK output
            
            # Store data in session state for downstream agents
            if response.data_result:
                context.session.state["temp:data_result"] = response.data_result
```

## 3. Implementation Pattern: Root Orchestration
The `RootAgent` manages the deterministic execution flow and conditional sub-agents.

```python
from google.adk.agents import RootAgent

# Define Optional Sub-Agents
sub_agents = [
    OptionalSubAgentSpec(
        agent=visualization_agent,
        run_when=lambda state: "temp:data_result" in state, # Predicate
        description="Generates charts if data is available"
    )
]

# Build Root Agent
root_agent = RootAgent(
    primary_agent=ca_query_agent,
    optional_sub_agents=sub_agents
)
```

## 4. Key Orchestration Patterns

### Deterministic CA-First
1. **Primary Agent**: Always executes the CA Query Agent first.
2. **Session State**: The primary agent populates `temp:data_result` or `temp:summary_data`.
3. **Conditional Sub-agents**: Executed sequentially based on `run_when` predicates.

### Real-time Sanitization
During streaming, ensure the output is compatible with the target interface:
- **Image Conversion**: Convert inline bytes from CA API into Markdown image URIs.
- **Text Cleaning**: Strip unnecessary code blocks or formatting from partial streams.

## 5. Best Practices
- **Resilient Flows**: Ensure the `RootAgent` continues even if an optional sub-agent fails.
- **Predicate Precision**: Keep `run_when` logic simple and focused on state keys.
- **Context Management**: Use `inline_context` for testing and `ConversationReference` for stateful multi-turn chats.
- **Normalization**: Standardize the structure of data stored in `session.state` to make sub-agents reusable across different data sources.

## 6. Troubleshooting
- **Streaming Interruption**: Check if intermediate proxies or gateways support long-lived HTTP/2 streams.
- **State Mismatch**: Verify that the keys used in `run_when` match exactly what is set by the primary agent.
- **Permission Errors**: Ensure the service account has both `roles/geminidataanalytics.user` and Looker `access_data` permissions.

## 7. Modern ADK `App` Packaging & Gemini Enterprise OAuth Token Injection

### Wrapping Agents in `google.adk.apps.App` with `ResumabilityConfig`
For production deployments on Vertex AI Agent Engine or Cloud Run, wrap your root agent in `google.adk.apps.App` with `ResumabilityConfig(is_resumable=True)` and use explicit `temp:` (turn-scoped) vs `user:` (cross-session) state prefixes:

```python
from google.adk.apps import App, ResumabilityConfig

app = App(
    name="looker_ca_streaming_app",
    root_agent=root_agent,
    resumability_config=ResumabilityConfig(is_resumable=True),
)
```

> **Latency Optimization for Single-Agent Wrappers**: If your ADK application only queries a single Looker Conversational Analytics agent without multi-domain tool routing, subclass `BaseAgent` (model-free pass-through) rather than wrapping `DataChatServiceClient` inside an outer `LlmAgent` tool—this eliminates a redundant outer LLM invocation (saving 2–4 seconds per turn).

### Extracting Injected Looker OAuth Tokens in Gemini Enterprise
When registered in Gemini Enterprise with a Looker OAuth 2.0 PKCE authorization resource (`AUTH_ID`), Gemini Enterprise automatically injects the end user's Looker OAuth `access_token` into `context.session.state[AUTH_ID]`. Extract it dynamically with a fallback to Looker API client credentials for local testing:

```python
def resolve_looker_credentials(context: InvocationContext, auth_id: str) -> geminidataanalytics.Credentials:
    creds = geminidataanalytics.Credentials()
    user_token = context.session.state.get(auth_id)
    if user_token:
        creds.oauth.access.access_token = user_token
    else:
        creds.oauth.secret.client_id = os.environ["LOOKERSDK_CLIENT_ID"]
        creds.oauth.secret.client_secret = os.environ["LOOKERSDK_CLIENT_SECRET"]
    return creds
```
