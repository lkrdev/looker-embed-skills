# Looker Embedding Skills Repository

This repository contains a collection of "skills" designed to assist AI agents and developers in implementing Looker embedding. Each skill encapsulates specific instructions, best practices, and examples for different aspects of Looker embedding, including SSO, Cookieless, and Visualization Components.

## Installation & Usage

To use these skills in your own projects—especially to enhance AI assistants like **Gemini**, **Cursor**, or **Claude Code**—we recommend adding this repository to your project.

### Installation via CLI

The `looker_embed_skills` CLI is a tool used to install and manage specialized instructions (skills) for AI agents in your workspace. 

```bash
npx @brettguenther/looker-embed-skills
```

## Core Embedding Skills

### SSO and Cookieless Embedding
*   **[sso-embed](skills/sso-embed/SKILL.md)**: Instructions for setting up Looker SSO (signed) and Cookieless embedding using the Looker Embed SDK.

### Visualization Components
*   **[visualization-components](skills/visualization-components/SKILL.md)**: Guide to building custom data experiences using Looker's React-based visualization components.

### Conversational Analytics
*   **[conversational-analytics-api](skills/conversational-analytics-api/SKILL.md)**: Instructions for building AI-powered chat interfaces that query Looker data via the Gemini Data Analytics API.
