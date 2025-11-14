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

## File Editing Pipeline

**Description:** AI-powered file editing using Morph-v3-large model for precise code modifications based on natural language instructions.

**Input:**
- Target file path
- Natural language instructions
- Code edit snippet with `// ... existing code ...` markers

**Output:** Modified file with applied edits

### Pipeline Flow

```
User Request: "Update function to handle error cases"
      |
      v
┌─────────────────────────────────────┐
│  Agent calls edit_file tool          │
│  Parameters:                         │
│  - target_file: "src/main.py"       │
│  - instructions: "Add error handling"│
│  - code_edit: "try/except block"     │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────────────┐
│  Read Current File Content           │
│  - Load from sandbox filesystem      │
│  - Validate file exists              │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────────────┐
│  Prepare Morph API Request           │
│                                      │
│  Format:                             │
│  <instruction>{instructions}</instruction>
│  <code>{file_content}</code>         │
│  <update>{code_edit}</update>        │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────────────┐
│  Call Morph-v3-large Model           │
│                                      │
│  Model: morph-v3-large               │
│  Temperature: 0.0 (deterministic)    │
│  Timeout: 30 seconds                 │
│  Provider: Morph API or OpenRouter   │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────────────┐
│  Response Processing                 │
│  - Extract edited code               │
│  - Remove markdown code blocks       │
│  - Validate changes                  │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────────────┐
│  Write Updated Content               │
│  - Upload to sandbox filesystem      │
│  - Preserve file permissions         │
└─────────────────────────────────────┘
      |
      v
┌─────────────────────────────────────┐
│  Return ToolResult                   │
│  {                                   │
│    "success": true,                  │
│    "message": "File edited",         │
│    "file_path": "src/main.py",       │
│    "original_content": "...",        │
│    "updated_content": "..."          │
│  }                                   │
└─────────────────────────────────────┘
```

### Data Flow Example

**Step 1: Input**
```python
{
  "target_file": "src/calculator.py",
  "instructions": "Add error handling for division by zero",
  "code_edit": """
// ... existing code ...
def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
// ... existing code ...
"""
}
```

**Step 2: File Content**
```python
# Original file content
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def divide(a, b):
    return a / b

def multiply(a, b):
    return a * b
```

**Step 3: Morph API Request**
```xml
<instruction>Add error handling for division by zero</instruction>
<code>
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def divide(a, b):
    return a / b

def multiply(a, b):
    return a * b
</code>
<update>
// ... existing code ...
def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
// ... existing code ...
</update>
```

**Step 4: Model Response**
```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def multiply(a, b):
    return a * b
```

**Step 5: Updated File**
File is saved with the new content, preserving all unmodified sections.

### Key Features

**Intelligent Editing:**
- Model understands `// ... existing code ...` markers
- Preserves unchanged sections automatically
- Handles multiple edits in single call
- Context-aware code modifications

**Fallback Mechanisms:**
- TipTap document manual updates
- Title/content/metadata field updates
- Error message propagation

**Validation:**
- File existence check
- Content comparison (detect no-op edits)
- Markdown code block extraction

---

