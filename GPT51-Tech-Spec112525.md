AuditFlow AI – Streamlit System Technical Specification
1. Executive Summary
AuditFlow AI is a browser-based, agentic application designed to assist in the creation of high-quality, ISO 13485 / GMP–compliant medical device audit reports. It is implemented as a Streamlit app and is suitable for deployment on Hugging Face Spaces or Streamlit Cloud.

Key capabilities:

Multi-stage agent pipeline (Layout Mapper → CAR Writer → Polisher, etc.) driven by agents.yaml.
Multi-provider LLM integration:
Google Gemini (gemini-2.5-flash, gemini-2.5-flash-lite)
OpenAI (gpt-5-nano, gpt-4o-mini, gpt-4.1-mini)
Anthropic (e.g., claude-3.5-haiku, claude-3.5-sonnet)
xAI / Grok (grok-4-fast-reasoning, grok-3-mini)
Smart Replacement module for high-precision template filling with change highlighting.
Human-in-the-loop (HITL) editing at each agent step.
WOW UI:
20 flower-based themes with accent colors.
Light / dark mode.
English / Traditional Chinese localization.
“Lucky Flower Wheel” random theme picker.
Analytics dashboard for latency and token usage plus rolling logs.
Robust error handling, especially for Gemini’s non-text responses and Streamlit state synchronization.
2. System Architecture
2.1 Technology Stack
Language: Python 3.x
Web Framework: Streamlit
Deployment Targets:
Hugging Face Spaces
streamlit.io (Streamlit Cloud)
AI Providers & SDKs:
Google Gemini: google-generativeai
OpenAI: openai
Anthropic: anthropic
xAI (Grok): xai-sdk
Configuration & Serialization:
agents.yaml for pipeline configuration
pyyaml for YAML parsing
Visualization:
Plotly Express (plotly.express) for interactive charts
Environment / Secrets:
Environment variables for provider API keys:
GEMINI_API_KEY
OPENAI_API_KEY
ANTHROPIC_API_KEY
XAI_API_KEY
UI fallback for keys (password fields in sidebar), without ever echoing env keys back to the UI.
2.2 High-Level Components
Streamlit UI

Sidebar: configuration, API keys, theme, language, agents.yaml tools.
Tabs:
Pipeline Tab: multi-agent orchestration and editing.
Smart Replace Tab: template–data merging with visual highlights.
Dashboard & Logs Tab: metrics, charts, event log.
Global State

st.session_state stores:
pipeline: PipelineState
smart_result: str
smart_template: str, smart_list: str
Manual API keys
UI state (theme, dark_mode, language)
Log entries
Per-agent output widget state (e.g., out_<agent.id>)
AI Service Layer

Generic call_llm dispatches to:
call_gemini
call_openai
call_anthropic
call_grok
Normalized response structure: {"text", "latency_ms", "input_tokens", "output_tokens"}.
Pipeline Configuration

agents.yaml defines:
Global template & observations defaults.
Ordered list of AgentConfig objects (id, name, provider, model, prompts, etc.).
UI can import/export this YAML.
3. Data Flow
3.1 Execution Flow Overview
Initialization:

On app start, init_session_state():
Attempts to load agents.yaml into PipelineState.
Seeds Smart Replace inputs from pipeline template & observations.
Initializes UI language, theme, dark_mode, logs, and manual API key placeholders.
User Input:

User edits:
Audit template Markdown.
Raw observations.
User configures:
Each agent’s provider, model, max tokens, temperature, user prompt, system suffix.
Smart Replace provider & model.
Agent Run:

When an agent is run:
build_agent_input constructs:
Combined system prompt = BASE_SYSTEM_PROMPT_ZH + system_prompt_suffix.
User content = Template + Observations + Previous Agent Output + Agent-specific instructions.
run_agent calls call_llm with constructed prompts.
Provider SDK returns a text response (or error).
AgentRunState is updated with:
status (running → completed or error)
output (LLM text)
latency_ms, input_tokens, output_tokens
error message if any
The corresponding st.session_state["out_<agent.id>"] is synchronized to show new output.
HITL Editing:

User edits each agent’s output in the textarea.
Edits are mirrored back into PipelineState.history[agent.id].output.
Subsequent agents receive the edited output as “previous agent output”.
Smart Replace:

User sets Template A and List B (pre-seeded from pipeline).
On run:
A synthetic AgentConfig with provider/model is created.
call_llm is invoked with SMART_REPLACE_SYSTEM_PROMPT and combined Template A + List B.
The result is stored in st.session_state.smart_result.
User can view result as:
Rendered HTML (st.markdown, showing <span style="color: coral"> highlights).
Raw Markdown (st.code).
Result is downloadable as Markdown.
Analytics & Logs:

Dashboard tab:
Aggregates latency and token usage across agents.
Visualizes per-agent metrics via bar charts (Plotly).
Logs:
log_event(level, message) appends to st.session_state.log_entries.
Displayed in reverse chronological order with color-coding per level.
4. Core Modules & UI Design
4.1 Sidebar
Responsibilities:

API Keys:
Password fields for:
Gemini
OpenAI
Anthropic
Grok (xAI)
Effective key resolution:
If env var exists, it is used (and never shown).
Else, the UI value is used.
Localization & Theme:
Language toggle (zh / en).
Theme selector (20 flower themes).
Dark Mode toggle.
Lucky Wheel:
Randomly selects a flower theme.
Logs selection event.
agents.yaml Management:
Reload from disk (load_agents_from_yaml).
Download current pipeline config as YAML.
Upload YAML to replace current pipeline:
YAML is validated with yaml.safe_load.
Invalid YAML => st.error + no crash.
Valid => new PipelineState is constructed.
4.2 Pipeline Tab
Key elements:

Template & Observations Inputs:

Two text_area fields:
“稽核報告模板 (Markdown)” / “Audit Template (Markdown)”
“原始觀察紀錄” / “Raw Observations”
Values bound to pipeline.template and pipeline.observations.
Agents Panel:
For each AgentConfig:

Config Controls:
Provider selection: gemini | openai | anthropic | grok.
Model dropdown:
Gemini: gemini-2.5-flash, gemini-2.5-flash-lite.
OpenAI: gpt-5-nano, gpt-4o-mini, gpt-4.1-mini.
Anthropic: e.g., claude-3.5-haiku, claude-3.5-sonnet.
Grok: grok-4-fast-reasoning, grok-3-mini.
Max tokens: numeric input (256–32000).
Temperature: slider (0.0–1.0).
User Prompt: multi-line text.
System Prompt Suffix: multi-line text appended to the base ISO 13485 system prompt.
API Key Warning:
If effective key for selected provider is missing, shows st.warning.
Execution Status:
Button: “僅執行此代理” / “Run This Agent Only”.
Status badge:
IDLE | RUNNING | DONE | ERROR with color-coded pill.
Latency caption if available.
Error message (st.error) showing error string if state.error is set.
Editable Output (HITL):
Uses a robust Streamlit pattern to avoid the “value ignored after first render” issue:
Unique key: out_<agent.id> in st.session_state.
On initial render: widget state seeded with state.output.
After each run (completed or error): widget state is overwritten with state.output.
text_area is rendered without value=..., driven solely by session state.
Changes in the widget are mirrored back into state.output.
Download button: export agent output as Markdown.
Full Pipeline Run:

Button: “執行整體管線” / “Run Full Pipeline”.
Calls run_full_pipeline sequentially for every agent.
4.3 Smart Replace Tab
Components:

Inputs (Left Column):

Template A (smart_template), pre-seeded from pipeline template.
List B (smart_list), pre-seeded from pipeline observations.
Provider, Model, Max Tokens, Temperature.
Provider key availability warning.
Run Button:

Executes run_smart_replace:
Creates a dummy AgentConfig bound to selected provider/model.
Calls call_llm with SMART_REPLACE_SYSTEM_PROMPT and combined Template A + List B.
On success:
Sets st.session_state.smart_result to returned text.
Log success event.
If result is empty, warn user.
On error:
Logs error event.
Displays st.error("Smart Replace error: ...").
Output View (Right Column):

Toggle:
“預覽 (HTML)” / “Preview (HTML)”: st.markdown with unsafe_allow_html=True.
“原始 Markdown” / “Raw (Markdown)”: st.code with language markdown.
Download button for smart_replace.md.
4.4 Dashboard & Logs Tab
Displays:

KPIs:

Progress: (# completed agents) / (total agents).
Total Runtime (sum of latencies).
Total Input Tokens.
Total Output Tokens.
Charts (Plotly):

Latency per agent:
Bar chart: x = agent name, y = latency (ms).
Output tokens per agent:
Bar chart: x = agent name, y = output tokens.
Logs:

Displays the last N log entries (log_entries), newest first.
Colors by level:
info: blue
success: green
error: red
5. AI Service Layer
5.1 Common Structure
call_llm(agent, system_prompt, user_content, api_keys):

Dispatches based on agent.provider.
Validates API key presence; raises informative runtime errors if missing.
Normalizes the provider-specific response into a shared dict interface.
5.2 Gemini (Google Generative AI)
SDK: google-generativeai
Function: call_gemini
Key behaviors:
genai.GenerativeModel(model_name=model, system_instruction=system_prompt).
generate_content(user_content, generation_config={...}).
Uses _extract_gemini_text(resp) to handle:
Nested candidates -> content -> parts -> text.
Avoids resp.text quick accessor, which crashes if there are no text parts.
If no text is found:
Raises RuntimeError("Gemini returned no content. (finish_reason=...) ...").
Returns latency and token usage from usage_metadata if available.
5.3 OpenAI
SDK: openai (via openai.OpenAI client).
Function: call_openai
Behavior:
chat.completions.create with system and user messages.
Extracts choices[0].message.content.
Uses resp.usage.prompt_tokens and resp.usage.completion_tokens if present.
5.4 Anthropic
SDK: anthropic
Function: call_anthropic
Behavior:
client.messages.create(model=..., system=..., messages=[{"role": "user", "content": user_content}]).
Concatenates all text blocks in resp.content where block.type == "text".
Reads usage.input_tokens and usage.output_tokens.
5.5 Grok (xAI)
SDK: xai-sdk
Function: call_grok
Pattern based on sample code:
XAIClient(api_key, timeout=3600).
chat = client.chat.create(model=model).
chat.append(xai_system(system_prompt)); chat.append(xai_user(user_content)).
chat.sample(options={"temperature": ..., "max_output_tokens": ...}).
Uses response.content as text and response.usage fields if available.
6. Configuration: agents.yaml
6.1 Structure
pipeline:
  template: "<default template markdown>"
  observations: "<default observations text>"
  agents:
    - id: "layout-mapper"
      name: "Layout Mapper"
      provider: "gemini"
      model: "gemini-2.5-flash"
      max_tokens: 4096
      temperature: 0.1
      user_prompt: |
        ...
      system_prompt_suffix: |
        ...
    - ...
6.2 Loading & Error Handling
load_agents_from_yaml(path="agents.yaml"):
Returns an empty default PipelineState if:
File missing.
File empty.
YAML invalid (logs error to console).
Converts agent mappings to AgentConfig instances.
Upload via UI:
File content read and passed through yaml.safe_load.
Invalid YAML => st.error + no pipeline replacement.
6.3 Export
export_agents_to_yaml(pipeline):
Serializes current PipelineState (template, observations, agents) to YAML.
Downloadable as agents.yaml from the sidebar.
7. Error Handling & UX Safeguards
Gemini safety / empty output:
Avoids using response.text directly.
Raises descriptive error if no parts contain text.
Errors are:
Logged via log_event.
Displayed in Pipeline / Smart Replace UI (st.error).
Streamlit state synchronization:
Agent output textareas use st.session_state keys as the single source of truth to avoid the “value ignored” bug.
Missing API keys:
Clear warnings per agent and per Smart Replace provider.
Malformed YAML:
Does not crash the app; shows UI errors and falls back to default empty pipeline.
Smart Replace:
Errors surfaced inline and logged.
Empty response is specifically detected and explained.
8. Theming & Localization
Themes:
20 flower themes each define:
primary, secondary, accent colors.
Theme and dark_mode applied via injected CSS:
--color-primary, --color-secondary, --color-accent, --bg-color, --text-color.
Localization:
UI_LABELS dict supports en and zh.
All major labels, button texts, and headings translated.
Lucky Flower Wheel:
Randomly picks a theme.
Logs theme selection.
9. Security & Privacy
API Key Handling:
Environment variables are preferred and never rendered in the UI.
Sidebar fields accept overrides but are stored only in st.session_state for the active session.
Client-Side Only:
All calls to providers are made from the Streamlit runtime; no extra backend microservices.
PII:
No built-in PII scrubber yet, but design allows for pre-processing stage before call_llm.
10. Extensibility
Agents:
New agents can be added by:
Editing agents.yaml.
Or configuring them in the UI and re-exporting YAML.
Providers:
call_llm is easily extended to support new providers by adding a new helper and branch.
Prompts:
Both base system prompt and per-agent system/user prompts are user-editable.
UI Features:
Additional tabs (e.g., “Model Health Check”, “PII Scrubber”) can be added with new Streamlit pages or tabs.
Analytics:
More charts or KPIs can be added to the Dashboard tab by reading from PipelineState.history.
20 Comprehensive Follow-Up Questions
Do you want to formalize a versioning scheme for agents.yaml (with version and changelog fields) so you can track changes to pipeline definitions over time?
Should we add a run history persistence layer (e.g., saving completed pipelines and outputs to disk or a DB) for later reopening and re-auditing?
Do you need a role-based access model (auditor vs. reviewer vs. admin), and if so, how should each role’s permissions differ in terms of editing agents and running pipelines?
Would you like to introduce a PII sanitization layer (regex-based or heuristic) before data is sent to external providers, with configurable patterns per organization?
Is there a requirement to support multiple audit templates (ISO 13485, GMP, MDSAP) with a selector that automatically swaps template and agents.yaml profiles?
Do you want to track and display cost estimates per provider/model using public token-pricing tables to support budgeting and governance?
Should the system enforce strict schema validation of agents.yaml using a JSON Schema (or similar) and provide a dedicated validation view for configuration changes?
Are you interested in adding a section-level editor (tree of headings + body) to replace the single big textarea per agent, especially for long reports?
For regulatory traceability, should the app embed metadata (provider, model, timestamp, agent id) into the exported Markdown or a sidecar JSON?
Do you want to implement automatic provider fallback policies (e.g., Gemini → OpenAI → Anthropic) at the agent level, configurable in YAML?
Should we allow per-agent advanced settings (e.g., Gemini safety categories, OpenAI response_format) to be configured in YAML and edited in the UI?
Would you like a test harness page that runs a small synthetic pipeline with known inputs, to verify all providers and models are working after deployment?
Do you want a diff view that compares two full pipeline outputs (before/after prompt or provider changes) for governance and quality assurance?
Should the system provide a clean final export mode that strips all HTML spans from Smart Replace and merges agent outputs into a single final Markdown/PDF?
Do you need to support multilingual reports (e.g., simultaneous Traditional Chinese and English versions) with agents configured to produce bilingual content?
Would you like a “strict compliance check” agent that only verifies ISO 13485 clauses and tags nonconformities, without rewriting narrative content?
Is there a need for batch processing (e.g., feeding a CSV of observations for 50 audits and running the pipeline on each, with progress tracking)?
Should we support multimodal inputs (images of nonconformities) via Gemini or Grok, with a dedicated upload section and image-aware agents?
Do you want to integrate authentication (OAuth or simple access tokens) for shared deployments so only authorized auditors can access the app?
Over the longer term, would you prefer to expose AuditFlow AI as a REST/GraphQL API service (for programmatic use) in addition to the current interactive Streamlit UI?
