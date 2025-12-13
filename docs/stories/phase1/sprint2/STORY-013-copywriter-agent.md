# Story 013: Copywriter Agent

## Story Information

| Field | Value |
|-------|-------|
| **Story ID** | STORY-013 |
| **Sprint** | Phase 1 - Sprint 2 |
| **Title** | Copywriter Agent for Content Generation |
| **Priority** | High |
| **Story Points** | 8 |
| **Status** | Ready for Development |
| **Dependencies** | STORY-009, STORY-010, STORY-011 |

## User Story

**As a** store creator
**I want** an AI agent that generates product descriptions and store pages
**So that** my store has professional, SEO-optimized content

## Acceptance Criteria

### AC1: Copywriter Input/Output Schemas
- [ ] `CopywriterInput` with store_name, niche, brand_identity, products
- [ ] `GeneratedPage` model with type, title, slug, content, meta_title, meta_description
- [ ] `CopywriterOutput` with enhanced products and generated pages

### AC2: Product Copy Prompts
- [ ] System prompt for e-commerce copywriter persona
- [ ] User prompt template for individual product copy
- [ ] JSON response format for title, description, meta tags, slug
- [ ] Guidelines for brand voice consistency

### AC3: Page Generation Prompts
- [ ] About Us page prompt with brand story integration
- [ ] Privacy Policy prompt with standard legal sections
- [ ] Terms of Service prompt with e-commerce specifics
- [ ] Refund Policy prompt with customer-friendly language

### AC4: Copywriter Agent Implementation
- [ ] Extends `BaseAgent[CopywriterInput, CopywriterOutput]`
- [ ] `_generate_product_copy()` for individual products
- [ ] `_generate_about_page()` with brand integration
- [ ] `_generate_privacy_policy()` with legal template
- [ ] `_generate_terms_of_service()` with e-commerce terms
- [ ] `_generate_refund_policy()` with return details
- [ ] `_generate_slug()` for URL-friendly slugs
- [ ] Progress tracking across all content generation

### AC5: Content Quality
- [ ] Product titles are SEO-optimized (60 chars max)
- [ ] Descriptions are 150-300 words with HTML formatting
- [ ] Meta descriptions are 155 chars max
- [ ] Brand voice is consistent across all content
- [ ] Legal pages include standard required sections

### AC6: Error Handling
- [ ] Graceful fallback for failed product copy generation
- [ ] Fallback content for failed page generation
- [ ] Logging of all generation failures

## Technical Details

### Files to Create

1. **`app/agents/copywriter/schemas.py`**
   - CopywriterInput
   - GeneratedPage
   - CopywriterOutput

2. **`app/agents/copywriter/prompts.py`**
   - PRODUCT_COPY_SYSTEM
   - PRODUCT_COPY_USER
   - PAGE_GENERATION_SYSTEM
   - ABOUT_PAGE_PROMPT
   - PRIVACY_POLICY_PROMPT
   - TERMS_OF_SERVICE_PROMPT
   - REFUND_POLICY_PROMPT

3. **`app/agents/copywriter/agent.py`**
   - CopywriterAgent

### Generation Flow

```
Input: { store_name, niche, brand_identity, products[] }
                    ↓
        [1. Product Copy] (for each product)
        - Rewrite title (SEO + brand voice)
        - Generate description (features, benefits, CTA)
        - Create meta title + description
        - Generate URL slug
                    ↓
        [2. About Page]
        - Incorporate brand story
        - Establish credibility
        - Include call to action
                    ↓
        [3. Legal Pages]
        - Privacy Policy (template + customization)
        - Terms of Service
        - Refund Policy
                    ↓
Output: { products: EnhancedProduct[], pages: GeneratedPage[] }
```

### Product Copy JSON Format

```json
{
    "title": "Neon LED Wall Art - Cyberpunk City Skyline",
    "description": "<p>Transform your space with this stunning...</p>",
    "meta_title": "Neon LED Wall Art | Cyberpunk Decor | NeonNest",
    "meta_description": "Illuminate your room with our premium neon LED wall art featuring a stunning cyberpunk city skyline. Shop now at NeonNest.",
    "slug": "neon-led-wall-art-cyberpunk-city-skyline"
}
```

### Page Types

| Page | Slug | Key Sections |
|------|------|--------------|
| About Us | about | Story, Mission, Values, CTA |
| Privacy Policy | privacy-policy | Collection, Use, Cookies, Security |
| Terms of Service | terms-of-service | Terms, Orders, Shipping, Liability |
| Refund Policy | refund-policy | Returns, Conditions, Process, Contact |

### Progress Updates

```python
# Product copy: 5% + (i/total * 50)  = 5-55%
# About page: 60%
# Privacy: 70%
# Terms: 80%
# Refund: 90%
# Complete: 100%
```

## Definition of Done

- [ ] All schemas created with validation
- [ ] Product copy maintains brand voice
- [ ] Page content is professional and complete
- [ ] SEO guidelines are followed
- [ ] Slug generation handles special characters
- [ ] Fallback handling for LLM failures
- [ ] Test coverage for all generation methods
- [ ] Code review completed

## Testing

### Unit Tests

```python
def test_slug_generation():
    agent = CopywriterAgent()
    slug = agent._generate_slug("Neon LED Wall Art - Cyberpunk!")
    assert slug == "neon-led-wall-art-cyberpunk"
    assert len(slug) <= 50
```

### Integration Tests

```python
@pytest.mark.asyncio
async def test_copywriter_generates_content(mock_llm_service, sample_context):
    agent = CopywriterAgent(llm_service=mock_llm_service)
    result = await agent.run(copywriter_input, sample_context)

    assert len(result.data.products) == len(input_products)
    assert any(p.type == "about" for p in result.data.pages)
    assert any(p.type == "privacy-policy" for p in result.data.pages)
```

## Notes

- Use lower temperature (0.5-0.7) for consistent content
- Legal pages should be templates with customization
- Consider adding FAQ page generation in future
- Meta descriptions should include brand name
- HTML formatting should be valid and semantic

## Related Stories

- Depends on: STORY-009 (Base Agent), STORY-010 (LLM), STORY-011 (Brand for voice)
- Used by: STORY-014 (Orchestrator coordinates copywriting)
