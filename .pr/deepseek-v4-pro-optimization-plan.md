# DeepSeek V4 Pro Optimization Plan for OpenHands

## Sources Used
All claims verified against:
- DeepSeek API Docs (api-docs.deepseek.com): Create Chat Completion, Thinking Mode, Tool Calls, Context Caching
- OpenHands SDK source (.venv/.../openhands/sdk/): llm, agent, condenser, prompts
- LiteLLM DeepSeek integration (.venv/.../litellm/llms/deepseek/chat/transformation.py)
- Direct LiteLLM introspection (get_supported_openai_params, get_llm_provider, get_model_info)

## Critical Discovery: Model Prefix Matters

| Prefix | Provider | Message Transform | Result |
|--------|----------|-------------------|--------|
| deepseek/deepseek-v4-pro | deepseek | Content list to string | CORRECT |
| openai/deepseek-v4-pro | openai | No transform | BREAKS |

Using openai/ prefix WILL BREAK because DeepSeek rejects content lists in array format.

## API Facts (from official docs)

Base URL:
- Quick start: base_url="https://api.deepseek.com"
- Strict tool calling: base_url="https://api.deepseek.com/beta"
- LiteLLM default: https://api.deepseek.com/beta (correct for tool calling)

Model name: deepseek-v4-pro (API), deepseek/deepseek-v4-pro (LiteLLM)

Thinking mode:
- Toggle: {"thinking": {"type": "enabled"}}
- Effort: {"reasoning_effort": "high"} or "max"
- Temperature/top_p/presence_penalty/frequency_penalty NOT supported in thinking mode
- OpenHands stripping temperature for reasoning models is CORRECT per DeepSeek API

reasoning_content handling:
- Non-tool-call turns: reasoning_content is "ignored" by API if passed back
- Tool-call turns: reasoning_content MUST be passed back or API returns 400
- OpenHands send_reasoning_content=True for deepseek-v4-pro is CORRECT

Context Caching:
- Automatic and transparent - enabled by default for all users
- Prefix-based matching between requests
- No API parameters needed (prompt_cache_key/retention not required)
- Cache lifetime: hours to days

## Bugs and Inefficiencies (UPDATED after doc review)

B1: No deepseek.j2 system prompt template exists
    Anthropic, Gemini, GPT-5 all have one. DeepSeek gets zero model-specific guidance.

B2: [NOT A BUG] temperature stripped for reasoning models
    DeepSeek API explicitly does not support temperature in thinking mode.
    OpenHands is CORRECT to strip it. No fix needed.

B3: Browser sections loaded in CLI mode (~600 token waste)

B4: Security risk assessment always included (~800 token waste)
    Rendered when llm_security_analyzer is set.

B5: [NOT AN ISSUE] prompt caching "disabled" for DeepSeek
    DeepSeek caching is AUTOMATIC, no API parameters needed.
    Adding to PROMPT_CACHE_RETENTION_MODELS is unnecessary.

B6: Condenser too conservative (max_size=240 events)
    For DeepSeek's 131K context, 240 tool-heavy events can overflow.

B7: deepseek-v4-pro not in verified models (hidden from UI)

B8: Dead max_message_chars field (defined but never read)

B9: Stuck detector may fire on slow thinking (thinking adds 30-60s)

B10: No event collapse after condensation (CLI UI slowdown)

B11: NEW - reasoning_effort should be "max" not "high"
     DeepSeek docs explicitly list "max" as supported and most powerful.

## Implementation Plan (UPDATED)

### Phase 1: System Prompt (~1500 tokens saved)

1. Create model_specific/deepseek.j2 with:
   - Thinking mode guidance (when to reason deeply vs act directly)
   - Tool calling format preferences
   - DeepSeek-specific behavior notes

2. Disable browser: enable_browser=false
   Removes BROWSER_TOOLS and EXTERNAL_SERVICES sections.

3. Disable security analyzer: security_analyzer="none"
   Removes SECURITY_RISK_ASSESSMENT section.

4. Make PULL_REQUESTS and SELF_DOCUMENTATION conditional

### Phase 2: API Configuration

1. Register deepseek-v4-pro in verified_models.py
2. Set reasoning_effort="max" (not "high")
3. Base URL: LiteLLM default /beta is correct
4. Model: deepseek/deepseek-v4-pro
5. Manual token limits: max_input_tokens=120000, max_output_tokens=32000
6. Do NOT add to PROMPT_CACHE_RETENTION_MODELS (automatic caching)
7. Do NOT modify chat_options.py (temperature stripping is correct)

### Phase 3: Condenser

- max_size=80 (was 240), keep_first=1
- Condenser LLM: deepseek/deepseek-v4-pro for consistent style

### Phase 4: Runtime Config

```bash
LLM_MODEL="deepseek/deepseek-v4-pro"
LLM_REASONING_EFFORT="max"
LLM_MAX_INPUT_TOKENS=120000
LLM_MAX_OUTPUT_TOKENS=32000
LLM_NUM_RETRIES=3
LLM_TIMEOUT=300
```

### Phase 5: Code Fixes

1. Create deepseek.j2 system prompt template
2. Register deepseek-v4-pro in verified_models.py
3. Stuck detector: increase patience for reasoning models
4. Wire max_message_chars or remove dead field
5. Event stream collapse after condensation

### Phase 6: lernja Integration

- LocalConversation(agent, workspace="/home/helios/workspace/lernja-gateway")
- AGENTS.md auto-loaded via skill loader
- confirmation_mode=false means NeverConfirm()

## Token Savings

| Optimization | Saved |
|---|---|
| Browser sections | ~600 |
| Security risk assessment | ~800 |
| Self-documentation | ~300 |
| Pull requests section | ~400 |
| System prompt: 4000 to ~2500 | ~1500 |
| Automatic prefix caching (turns 2+) | ~2500 cached |

## Files to Modify

1. deepseek.j2 (NEW) - Model-specific system prompt
2. verified_models.py - Register deepseek-v4-pro
3. config.toml - User runtime config
4. system_prompt.j2 - Conditional flags for PR/docs sections
5. stuck_detector.py - Increase patience
6. llm.py - Wire max_message_chars or remove

## Files NOT to modify (were wrong in earlier plan)

- chat_options.py - Temperature stripping is correct per DeepSeek API
- model_features.py - Prompt caching is automatic, no changes needed
- PROMPT_CACHE_RETENTION_MODELS - Not needed for DeepSeek

## Risks

- deepseek-v4-pro not in LiteLLM cost DB: manual token limits required
- Thinking mode latency: 30-60s per call at "max" effort
- strict function calling: LiteLLM may not pass strict=true on tools
- Model deprecation: deepseek-chat/reasoner deprecated 2026/07/24
