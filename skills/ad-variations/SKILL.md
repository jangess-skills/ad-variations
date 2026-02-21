---
name: ad-variations
description: Generate diverse Facebook/Meta ad creative variations using HAF framework.
  Use when the user asks for ad variations, creative testing assets,
  hook-angle-format combinations, Meta ad creatives, or Facebook ad images
  for a product or brand.
---

# Generate Ad Variations

## Overview

This skill generates diverse Meta/Facebook ad creatives that are optimized for Andromeda's Entity ID system. It uses the HAF (Hook-Angle-Format) framework to ensure each variation is genuinely distinct, earning separate auction tickets.

**Research foundation**: `data/Ad variation engine/Meta Andromeda Ads Best Practices`
**Directive**: `directives/generate_ad_variations.md`
**Script**: `execution/generate_ad_variations.py`

## Guided Flow

Follow these phases in order. Do NOT skip the human-in-the-loop checkpoint.

### Phase A: Brand Setup

1. Ask: "What brand are we creating ads for?"
2. Check if brand brief exists at `data/Ad variation engine/brands/<brand>/brand_brief.md`
   - **If exists**: Read it, confirm with user ("I found your brand brief for X. Still accurate?")
   - **If not**: Ask for product/service description and target audience, then save as:

```markdown
# <Brand Name>

## Product/Service
<description>

## Target Audience
<audience>
```

Save to `data/Ad variation engine/brands/<brand>/brand_brief.md`

### Phase B: Angle Suggestions

Based on the brand brief, generate 3-5 suggestions for EACH of these categories:

- **Pain points** — problems the audience faces that the product solves
- **Social proof** — credibility signals, numbers, testimonials
- **Transformation** — before/after scenarios, results
- **Urgency** — time-sensitive reasons to act now
- **Lifestyle** — aspirational outcomes, identity

Present as a checklist using AskUserQuestion with multiSelect. Let the user:
- Select which angles to use
- Add custom angles
- Skip categories

Save selected angles to `data/Ad variation engine/brands/<brand>/selected_angles.md`

### Phase C: Quick Campaign Questions

Ask these in sequence:

1. "Any offers? (yes/no)" — if yes, ask what the offer is
2. "How many variations?" — default 6, suggest range 4-12
3. "Which ad sizes?" — default 4:5 (vertical feed), options: 1:1, 4:5, 9:16

### Phase D: Generate Review Document

Write the brief JSON to `.tmp/ad_brief.json` with this schema:

```json
{
  "project_name": "<slug>",
  "brand": "<brand-slug>",
  "brand_brief_path": "data/Ad variation engine/brands/<brand>/brand_brief.md",
  "product_description": "<from brand brief>",
  "target_audience": "<from brand brief>",
  "selected_angles": {
    "pain_points": ["..."],
    "social_proof": ["..."],
    "transformation": ["..."],
    "urgency": ["..."],
    "lifestyle": ["..."]
  },
  "offer": "<offer or null>",
  "num_variations": 6,
  "formats": ["4:5"],
  "language": "en",
  "style_references": ["data/Ad variation engine/Reference images/Style reference/"],
  "subject_references": ["data/Ad variation engine/Reference images/Subject reference/"]
}
```

Then run:
```bash
python3 execution/generate_ad_variations.py --brief .tmp/ad_brief.json --mode review
```

This generates the variation matrix, all ad copy, and a review document.

### Phase E: Human Checkpoint (CRITICAL)

Read the generated `review.md` and present it to the user. Ask:

"Here's the creative plan with all variations and copy. Review it and let me know:
- **Approve** to start image generation
- **Request changes** to specific variations
- **Add/remove** variations"

**Do NOT proceed to image generation until the user explicitly approves.**

### Phase F: Image Generation

After approval, run:
```bash
python3 execution/generate_ad_variations.py \
  --brief .tmp/ad_brief.json \
  --output-dir <same output dir from review> \
  --mode images
```

The script auto-detects **edit mode** (when `reference_layout` exists in `variation_matrix.json`) vs **generate mode**:
- **Edit mode**: Edits the reference image, swapping only the headline text per variation. Uses Layout Analyzer → Editor → Similarity Critic pipeline.
- **Generate mode**: Generates images from scratch using PaperBanana 5-agent pipeline (Retriever → Stylist → Visualizer → Critic).

Then read and present the `summary.md` to the user.

## Reference Images

Check these folders before generating:
- `data/Ad variation engine/Reference images/Style reference/` — ad visual style examples (also used as base for edit mode)
- `data/Ad variation engine/Reference images/Subject reference/` — product photos (optional subject fidelity reference)

If a style reference is provided, the pipeline uses **edit mode** (modifies the reference ad, only changing text). If empty, it falls back to **generate mode** (creates images from scratch via text prompts).

## Notes

- Edit mode cost: ~$0.13 per edit attempt (Gemini)
- Generate mode cost: ~$0.13 per image (Gemini)
- LLM calls for matrix + copy cost ~$0.10-0.30 total
- All output goes to `output/ad variations/<brand>/<project>/`
