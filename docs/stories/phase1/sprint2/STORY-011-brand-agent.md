# Story 011: Brand Agent

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-011 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | Brand Generation Agent |
| **Priority** | High |
| **Story Points** | 8 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-009, STORY-010 |

## User Story

**As a** store creator
**I want** an AI agent that generates a complete brand identity from my niche
**So that** my store has professional branding without hiring a designer

## Acceptance Criteria

### AC1: Brand Input Schema
- [ ] `BrandInput` model with niche (required)
- [ ] Optional style preference field
- [ ] Optional tone field
- [ ] Optional target_audience field

### AC2: Brand Output Schema
- [ ] `BrandColors` model with primary, secondary, accent, background, text, muted
- [ ] `BrandFonts` model with heading and body fonts
- [ ] `BrandVoice` model with tone, personality traits, writing_style
- [ ] `BrandOutput` model combining all elements plus name, tagline, logo_prompt, story, domain_suggestions

### AC3: Brand Prompts
- [ ] System prompt establishing the AI as a brand strategist
- [ ] User prompt template with placeholders for niche and preferences
- [ ] JSON response format specification
- [ ] `build_brand_prompt()` helper function

### AC4: Brand Agent Implementation
- [ ] Extends `BaseAgent[BrandInput, BrandOutput]`
- [ ] `name` property returns "Brand Generator"
- [ ] `_execute()` method implements generation logic
- [ ] Progress updates at key stages (10%, 30%, 70%, 90%, 100%)
- [ ] Color hex code validation
- [ ] Error handling for invalid LLM responses

### AC5: Brand Generation Quality
- [ ] Brand names are unique and memorable (1-2 words)
- [ ] Taglines are catchy (5-10 words)
- [ ] Color palettes are cohesive and appropriate for niche
- [ ] Font pairings are from approved list
- [ ] Domain suggestions are relevant

## Technical Details

### Files to Create

1. **`app/agents/brand/schemas.py`**
   - BrandInput
   - BrandColors
   - BrandFonts
   - BrandVoice
   - BrandOutput

2. **`app/agents/brand/prompts.py`**
   - BRAND_GENERATION_SYSTEM_PROMPT
   - BRAND_GENERATION_USER_PROMPT
   - build_brand_prompt()

3. **`app/agents/brand/agent.py`**
   - BrandAgent class

### Generation Flow

```
Input: { niche: "Cyberpunk home decor", style?: "futuristic", tone?: "edgy" }
                    ↓
        [1. Build Prompt]
        - Construct system and user prompts
        - Include preferences if provided
                    ↓
        [2. LLM Generation]
        - Call LLM with JSON mode
        - Temperature: 0.8 (creative)
                    ↓
        [3. Parse & Validate]
        - Parse JSON response
        - Validate color hex codes
        - Validate font names
                    ↓
Output: BrandOutput { name, tagline, colors, fonts, voice, story, ... }
```

### Approved Font List

```python
APPROVED_FONTS = [
    "Inter", "Poppins", "Montserrat", "Playfair Display",
    "Roboto", "Open Sans", "Lato", "Raleway", "Oswald", "Merriweather"
]
```

### Sample Output

```json
{
    "name": "NeonNest",
    "tagline": "Illuminate Your Space",
    "logo_prompt": "Minimalist neon sign logo with geometric bird shape, magenta and cyan gradient, dark background",
    "colors": {
        "primary": "#FF00FF",
        "secondary": "#00FFFF",
        "accent": "#FFFF00",
        "background": "#1a1a1a",
        "text": "#ffffff",
        "muted": "#888888"
    },
    "fonts": {
        "heading": "Poppins",
        "body": "Inter"
    },
    "voice": {
        "tone": "futuristic",
        "personality": ["bold", "innovative", "edgy"],
        "writing_style": "Concise and impactful with tech-inspired vocabulary"
    },
    "story": "NeonNest was born from a passion for the intersection of technology and home design...",
    "domain_suggestions": ["neonnest.com", "neon-nest.co", "neonnesthome.com"]
}
```

## Definition of Done

- [ ] All schemas created with proper validation
- [ ] Prompts generate consistent, high-quality output
- [ ] Agent extends BaseAgent correctly
- [ ] Progress updates fire at appropriate stages
- [ ] Invalid colors are logged and handled
- [ ] Test coverage for success and failure cases
- [ ] Integration test with real LLM API
- [ ] Code review completed

## Testing

### Unit Tests

```python
@pytest.mark.asyncio
async def test_brand_agent_success(mock_llm_service, sample_context):
    agent = BrandAgent(llm_service=mock_llm_service)
    input_data = BrandInput(niche="Cyberpunk home decor")
    result = await agent.run(input_data, sample_context)

    assert result.success is True
    assert result.data.name is not None
    assert result.data.colors.primary.startswith("#")
```

### Manual Testing

1. Set up LLM API key
2. Create agent: `agent = BrandAgent()`
3. Run with test input: `await agent.run(BrandInput(niche="Pet accessories"), context)`
4. Verify output quality and completeness

## Notes

- Use higher temperature (0.8) for creative brand generation
- Consider caching results for same niche to save API costs
- Logo prompt should be detailed enough for image generation
- Domain suggestions are for reference only, not checked for availability

## Related Stories

- Depends on: STORY-009 (Base Agent), STORY-010 (LLM Service)
- Used by: STORY-014 (Genesis Orchestrator)
