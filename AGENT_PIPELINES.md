# Agent Pipelines Documentation

This document describes all possible agent workflow pipelines in the Suna.so application, including data flow, prompt sequences, and tool orchestration.

## Table of Contents
- [Agent Creation Pipeline](#agent-creation-pipeline)
- [File Editing Pipeline](#file-editing-pipeline)
- [Voice Call Pipeline](#voice-call-pipeline)
- [Agent Self-Configuration Pipeline](#agent-self-configuration-pipeline)
- [Main Agent Execution Pipeline](#main-agent-execution-pipeline)

---

## Agent Creation Pipeline

**Description:** Automated workflow for creating a new AI agent from natural language description, generating name, system prompt, icon, and colors in parallel.

**Input:** User's natural language description of desired agent

**Output:** Configured agent with:
- Unique agent_id
- Generated name (2-4 words)
- Custom system prompt
- Icon and color scheme

### Pipeline Flow

```
User Description
      |
      v
┌─────────────────────────────────────┐
│  generate_agent_config_from_description  │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────┬─────────────────────────────┐
│   PARALLEL EXECUTION        │                             │
│                             │                             │
v                             v                             │
┌─────────────────────────┐   ┌─────────────────────────┐   │
│ generate_agent_name_    │   │ generate_icon_and_      │   │
│ and_prompt()            │   │ colors()                │   │
│                         │   │                         │   │
│ Model: gpt-5-nano       │   │ Model: gpt-5-nano       │   │
│ Temperature: 0.7        │   │ Temperature: 0.7        │   │
│ Max Tokens: 2000        │   │ Max Tokens: 4000        │   │
│ Format: JSON            │   │ Format: JSON            │   │
└─────────────────────────┘   └─────────────────────────┘   │
      |                             |                        │
      v                             v                        │
┌─────────────────────────┐   ┌─────────────────────────┐   │
│ Prompt:                 │   │ Prompt:                 │   │
│ "You are an AI worker   │   │ "Select icon & colors   │   │
│ configuration expert.   │   │ for AI agent based on   │   │
│ Generate name and       │   │ name and description"   │   │
│ system prompt."         │   │                         │   │
│                         │   │ Available:              │   │
│ Input: {description}    │   │ - 400+ Lucide icons     │   │
│                         │   │ - 15 predefined colors  │   │
└─────────────────────────┘   └─────────────────────────┘   │
      |                             |                        │
      v                             v                        │
┌─────────────────────────┐   ┌─────────────────────────┐   │
│ Output:                 │   │ Output:                 │   │
│ {                       │   │ {                       │   │
│   "name": "Research     │   │   "icon": "search",     │   │
│            Assistant",  │   │   "background_color":   │   │
│   "system_prompt":      │   │          "#6366F1",     │   │
│   "Act as expert..."    │   │   "text_color":         │   │
│ }                       │   │          "#FFFFFF"      │   │
└─────────────────────────┘   └─────────────────────────┘   │
      |                             |                        │
      └─────────────────────────────┴────────────────────────┘
                                    |
                                    v
                    ┌───────────────────────────────┐
                    │  asyncio.gather()             │
                    │  (Wait for both completions)  │
                    └───────────────────────────────┘
                                    |
                                    v
                    ┌───────────────────────────────┐
                    │  Combine Results              │
                    │  {                            │
                    │    "name": "...",             │
                    │    "system_prompt": "...",    │
                    │    "icon_name": "...",        │
                    │    "icon_color": "...",       │
                    │    "icon_background": "..."   │
                    │  }                            │
                    └───────────────────────────────┘
                                    |
                                    v
                    ┌───────────────────────────────┐
                    │  Create Agent in Database     │
                    │  - Generate agent_id          │
                    │  - Store configuration        │
                    │  - Enable default tools       │
                    └───────────────────────────────┘
                                    |
                                    v
                              ┌───────────┐
                              │  Success  │
                              │  Response │
                              └───────────┘
```

### Data Flow

**Step 1: User Input**
```
Input: "I need an agent that can help me with research and fact-checking"
```

**Step 2: Parallel LLM Calls**

*Call 1 - Name & Prompt Generation:*
```json
{
  "model": "openai/gpt-5-nano-2025-08-07",
  "temperature": 0.7,
  "max_tokens": 2000,
  "response_format": {"type": "json_object"},
  "messages": [
    {
      "role": "system",
      "content": "You are an AI worker configuration expert. Generate a name and system prompt..."
    },
    {
      "role": "user",
      "content": "Generate name and system prompt for:\n\nI need an agent that can help me with research and fact-checking"
    }
  ]
}
```

*Call 2 - Icon & Colors Selection:*
```json
{
  "model": "openai/gpt-5-nano-2025-08-07",
  "temperature": 0.7,
  "max_tokens": 4000,
  "response_format": {"type": "json_object"},
  "messages": [
    {
      "role": "system",
      "content": "Select appropriate icons and colors for AI agents...\nAvailable icons: [400+ Lucide icons]\nAvailable colors: [15 hex codes]"
    },
    {
      "role": "user",
      "content": "Select the most appropriate icon and color scheme for:\nDescription: I need an agent that can help me with research and fact-checking"
    }
  ]
}
```

**Step 3: Response Processing**

*Response 1:*
```json
{
  "name": "Research Assistant",
  "system_prompt": "Act as an expert research assistant. Help users find and analyze information. Always verify facts and cite sources clearly."
}
```

*Response 2:*
```json
{
  "icon": "search",
  "background_color": "#6366F1",
  "text_color": "#FFFFFF"
}
```

**Step 4: Combined Configuration**
```json
{
  "name": "Research Assistant",
  "system_prompt": "Act as an expert research assistant. Help users find and analyze information. Always verify facts and cite sources clearly.",
  "icon_name": "search",
  "icon_color": "#FFFFFF",
  "icon_background": "#6366F1"
}
```

### Error Handling

**Fallback Values:**
- If name generation fails: "Custom Assistant"
- If prompt generation fails: "Act as a helpful AI assistant. {description}"
- If icon selection fails: "bot"
- If color selection fails: "#6366F1" (background), "#FFFFFF" (text)

**Validation:**
- Icon names validated against RELEVANT_ICONS list
- Color hex codes validated against frontend_colors list
- Invalid selections trigger fallback to defaults

---

