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

## Voice Call Pipeline

**Description:** End-to-end workflow for making AI-powered phone calls with real-time transcription, safety validation, and call monitoring.

**Input:**
- Phone number (any format)
- First message (greeting)
- Optional system prompt

**Output:**
- Call initiated with call_id
- Real-time transcript
- Call completion report

### Pipeline Flow

```
User: "Call +1-415-555-1234 and introduce yourself"
      |
      v
┌──────────────────────────────────────────┐
│  make_phone_call Tool Invocation          │
│  Parameters:                              │
│  - phone_number: "+1-415-555-1234"        │
│  - first_message: "Hello, I'm calling..." │
│  - system_prompt: (optional)              │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 1: Phone Number Normalization       │
│                                           │
│  normalize_phone_number()                 │
│  - Parse with phonenumbers library        │
│  - Try multiple strategies:               │
│    * Explicit + prefix                    │
│    * Regional parsing (US, GB, IN...)     │
│    * 00 prefix conversion                 │
│  - Validate: is_valid_number()            │
│  - Extract country info                   │
│                                           │
│  Output:                                  │
│  - E.164 format: "+14155551234"           │
│  - Country code: "1"                      │
│  - Country name: "United States"          │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 2: Safety Validation                │
│                                           │
│  validate_call_safety()                   │
│                                           │
│  Check 1: Emergency Number Detection      │
│  - Compare against EMERGENCY_NUMBERS      │
│  - Match EMERGENCY_PATTERNS               │
│  - Block: 911, 999, 112, 000, etc.       │
│                                           │
│  Check 2: Content Safety (First Message) │
│  - Scan for PROHIBITED_KEYWORDS           │
│    (illegal, scam, fraud, threat...)      │
│  - Detect scam patterns:                  │
│    * Urgent payment requests              │
│    * Verification of sensitive info       │
│    * Prize/lottery claims                 │
│    * Government impersonation             │
│                                           │
│  Check 3: Content Safety (System Prompt) │
│  - Same validation as first message       │
│                                           │
│  If any check fails → BLOCK CALL          │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 3: Prompt Enhancement               │
│                                           │
│  Build Enhanced System Prompt:            │
│                                           │
│  1. Base Prompt (user or default)         │
│     DEFAULT_SYSTEM_PROMPT:                │
│     "You are a professional AI assistant  │
│      making a phone call. Be natural,     │
│      conversational, concise..."          │
│                                           │
│  2. + Country Context                     │
│     "You are calling {country_name}       │
│      (code +{country_code}).              │
│      Be aware of cultural differences,    │
│      time zones, language preferences."   │
│                                           │
│  3. + Safety Guidelines (MANDATORY)       │
│     "ETHICAL GUIDELINES:                  │
│      - NEVER request SSN, passwords,      │
│        credit cards, bank accounts        │
│      - NEVER discuss illegal activities   │
│      - NEVER impersonate government       │
│      - NEVER create urgency               │
│      - NEVER request payments             │
│      - Be honest about being AI"          │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 4: VAPI Configuration               │
│                                           │
│  vapi_config.get_assistant_config()       │
│                                           │
│  {                                        │
│    "firstMessage": enhanced_first_msg,    │
│    "model": {                             │
│      "provider": "openai",                │
│      "model": "gpt-5-mini",               │
│      "temperature": 0.7,                  │
│      "messages": [{                       │
│        "role": "system",                  │
│        "content": enhanced_system_prompt  │
│      }]                                   │
│    },                                     │
│    "voice": {                             │
│      "provider": "playht",                │
│      "voiceId": "jennifer-playht"         │
│    },                                     │
│    "transcriber": {                       │
│      "provider": "deepgram",              │
│      "model": "nova-2",                   │
│      "language": "en",                    │
│      "smartFormat": true                  │
│    },                                     │
│    "maxDurationSeconds": 600              │
│  }                                        │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 5: Initiate Call via VAPI           │
│                                           │
│  POST https://api.vapi.ai/call/phone      │
│  {                                        │
│    "phoneNumberId": VAPI_PHONE_NUMBER_ID, │
│    "customer": {                          │
│      "number": "+14155551234"             │
│    },                                     │
│    "assistant": assistant_config          │
│  }                                        │
│                                           │
│  Response:                                │
│  {                                        │
│    "id": "call_abc123...",                │
│    "status": "queued",                    │
│    "createdAt": "2025-01-..."             │
│  }                                        │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 6: Database Recording               │
│                                           │
│  Store in vapi_calls table:               │
│  {                                        │
│    "call_id": "call_abc123...",           │
│    "agent_id": current_agent_id,          │
│    "thread_id": current_thread_id,        │
│    "phone_number": "+14155551234",        │
│    "direction": "outbound",               │
│    "status": "queued",                    │
│    "transcript": [],                      │
│    "started_at": timestamp                │
│  }                                        │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  STEP 7: Thread Message                   │
│                                           │
│  Add message to conversation:             │
│  "📞 Initiating call to +14155551234      │
│   🌍 Country: United States               │
│   Call ID: call_abc1...                   │
│   The conversation will appear here       │
│   in real-time."                          │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  Return Success                           │
│  {                                        │
│    "call_id": "call_abc123...",           │
│    "status": "queued",                    │
│    "phone_number": "+14155551234",        │
│    "country": "United States",            │
│    "next_action": "MONITOR_CALL",         │
│    "instructions": "Use                   │
│     wait_for_call_completion..."          │
│  }                                        │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  Agent calls wait_for_call_completion     │
│  Parameters:                              │
│  - call_id: "call_abc123..."              │
│  - check_interval: 2 seconds              │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  MONITORING LOOP (Every 2 seconds)        │
│                                           │
│  Query vapi_calls table:                  │
│  - Check status (queued → ringing →      │
│                   in-progress → ended)    │
│  - Get transcript updates                 │
│                                           │
│  If new transcript messages:              │
│  - Stream to conversation thread:         │
│    "🤖 AI: Hello, I'm calling..."         │
│    "👤 Caller: Hi, who is this?"          │
│    "🤖 AI: I'm an AI assistant..."        │
│                                           │
│  Continue until status ∈ [ended,          │
│   completed, failed, error]               │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  CALL COMPLETION                          │
│                                           │
│  Calculate metrics:                       │
│  - Duration: 127 seconds                  │
│  - Cost: $0.10 (base + trans + voice +    │
│           model costs)                    │
│  - Credits: $0.10 * TOKEN_PRICE_MULT      │
│                                           │
│  Add completion message:                  │
│  "📞 Call Completed                       │
│   Status: ended                           │
│   Duration: 127 seconds                   │
│   Credits Used: $0.4000"                  │
└──────────────────────────────────────────┘
      |
      v
┌──────────────────────────────────────────┐
│  Return Final ToolResult                  │
│  {                                        │
│    "call_id": "call_abc123...",           │
│    "final_status": "ended",               │
│    "duration_seconds": 127,               │
│    "transcript_messages": 8,              │
│    "cost": 0.10,                          │
│    "message": "Call completed..."         │
│  }                                        │
└──────────────────────────────────────────┘
```

### Data Flow Example

**Initial Request:**
```python
{
  "phone_number": "415-555-1234",
  "first_message": "Hello, I'm calling from Acme Corp to discuss your recent inquiry",
  "system_prompt": "You are a friendly customer service representative"
}
```

**After Normalization:**
```python
{
  "normalized_phone": "+14155551234",
  "country_code": "1",
  "country_name": "United States"
}
```

**Enhanced System Prompt:**
```
You are a friendly customer service representative

IMPORTANT: You are calling a phone number in United States (country code +1). Please be aware of potential cultural differences, time zones, and language preferences.

ETHICAL GUIDELINES (MANDATORY):
- NEVER request sensitive personal information (SSN, passwords, credit card numbers, bank accounts, PINs)
- NEVER discuss illegal activities, threats, or emergency services
- NEVER impersonate government agencies, law enforcement, or financial institutions
- NEVER create urgency to manipulate the recipient into taking immediate action
- NEVER request payments, transfers, or financial transactions
- Be respectful, honest, and transparent about being an AI assistant
```

**Live Transcript Stream:**
```
[00:01] 🤖 AI: Hello, I'm calling from Acme Corp to discuss your recent inquiry
[00:05] 👤 Caller: Oh yes, I was asking about your pricing
[00:08] 🤖 AI: Of course! I'd be happy to help with that
[00:12] 👤 Caller: What are your current rates?
[00:15] 🤖 AI: Our basic plan starts at $29 per month
...
[02:07] 👤 Caller: Great, thank you for the information
[02:10] 🤖 AI: You're welcome! Have a great day
[02:12] Call ended
```

### Safety Features

**Automatic Blocking:**
- Emergency numbers (911, 999, 112, etc.)
- Content with prohibited keywords
- Scam pattern detection
- Sensitive information requests

**Real-time Monitoring:**
- Call status updates every 2 seconds
- Live transcript streaming
- Error detection and handling
- Automatic timeout after 1 hour

**Cost Control:**
- Per-minute billing: $0.10/minute total
  - Base: $0.05/min
  - Transcription: $0.01/min
  - Voice: $0.02/min
  - Model: $0.02/min
- Maximum duration: 10 minutes (configurable)
- Cost tracking and reporting

---

