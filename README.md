# Looker Embedding Skills Repository

This repository contains a collection of "skills" designed to assist AI agents and developers in implementing Looker embedding. Each skill encapsulates specific instructions, best practices, and examples for different aspects of Looker embedding, including SSO, Cookieless, and Visualization Components.

## Installation & Usage

To use these skills in your own projects—especially to enhance AI assistants like **Gemini**, **Cursor**, or **Claude Code**—we recommend adding this repository to your project.

### Installation via CLI

```bash
npx skills add lkrdev/looker-embed-skills
```

## Available Skills

### Core Embedding
*   **[sso-embed](skills/sso-embed/SKILL.md)**: Setup, implementation, and troubleshooting of Looker SSO (signed) and Cookieless embedding using the Looker Embed SDK.
*   **[embed-themes](skills/embed-themes/SKILL.md)**: Programmatic management of Looker themes via API for automated styling and brand management.
*   **[embed-javascript-events-api](skills/embed-javascript-events-api/SKILL.md)**: Implementing interactive communication between host applications and embedded Looker iframes.

### API & Direct Browser Integration (CORS)
*   **[cors-api](skills/cors-api/SKILL.md)**: Direct browser-to-Looker API calls using OAuth2 with PKCE for secure client-side authentication.
*   **[cors-api-proxy](skills/cors-api-proxy/SKILL.md)**: Direct browser-to-Looker API calls using a backend proxy to delegate scoped authentication tokens.

### Visualization & UI Components
*   **[visualization-components](skills/visualization-components/SKILL.md)**: Building custom data experiences using Looker's React-based visualization components.

### Conversational Analytics & Agents
*   **[conversational-analytics-api](skills/conversational-analytics-api/SKILL.md)**: Building AI-powered chat interfaces that query Looker data using natural language.
*   **[ca-adk-streaming](skills/ca-adk-streaming/SKILL.md)**: Building streaming, data-driven agentic applications using Looker CA API and Google ADK.
