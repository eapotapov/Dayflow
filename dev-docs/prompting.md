# LLM Prompting Structure

## Overview

Dayflow uses Large Language Models (LLMs) to analyze screen recordings and generate meaningful activity summaries. This document details the prompting architecture, strategies, and implementation across different providers.

## Table of Contents
- [Prompting Philosophy](#prompting-philosophy)
- [Two-Phase Analysis](#two-phase-analysis)
- [Provider Implementations](#provider-implementations)
- [Prompt Templates](#prompt-templates)
- [Category Integration](#category-integration)
- [Quality Control](#quality-control)

## Prompting Philosophy

### Core Principles
1. **Condensation First**: Merge related activities into meaningful sessions (30-60+ minutes)
2. **Context Preservation**: Use sliding window (1 hour) to maintain continuity
3. **Natural Language**: Write like "texting a friend" - conversational and specific
4. **Time Accuracy**: Strict timestamp validation within video boundaries

### Golden Rules
- **Fewer segments are better** - Aim for 3-5 segments per 15-minute video
- **Group by purpose, not platform** - Trip planning across 5 sites = ONE segment
- **Extend by default** - Only split for major context changes (2-3+ minutes)
- **Include interruptions** - Brief distractions stay within main activity

## Two-Phase Analysis

### Phase 1: Video Transcription
Converts raw video into structured observations.

**Input**: 15-second video chunks combined into ~15-minute segments
**Output**: JSON array of observations with timestamps

**Key Instructions**:
```
- Video duration: Exactly MM:SS long
- All timestamps MUST be within 00:00 to MM:SS
- Detect idle periods (5+ minutes of no activity)
- Group activities by purpose/context
- Target 3-5 segments per 15-minute video
```

### Phase 2: Activity Card Generation
Transforms observations into user-facing timeline cards.

**Input**:
- Recent observations (1-hour window)
- Existing timeline cards
- User-defined categories
- Current timestamp

**Output**: Activity cards with titles, summaries, and categories

**Key Instructions**:
```
- Extend existing cards when possible
- Preserve original startTime when extending
- Rewrite summaries to tell complete story
- Assign exactly one category per card
- Use first-person perspective in summaries
```

## Provider Implementations

### Gemini Provider
Location: `Core/AI/GeminiDirectProvider.swift`

#### Transcription Prompt Structure
```swift
// Lines 195-319
1. Critical timestamp boundaries
2. Golden rule (3-5 segments)
3. Core principles
4. Format specification (JSON)
5. Good/bad examples
6. Idle detection rules
```

#### Card Generation Prompt Structure
```swift
// Lines 472-650+
1. Data integrity rules
2. Extension directives
3. Title guidelines (5-10 words)
4. Summary guidelines (2-3 sentences)
5. Category assignment
6. App/site identification
```

### Ollama Provider (Local)
Location: `Core/AI/OllamaProvider.swift`

Uses multi-step approach due to local model limitations:
1. **Frame extraction** (30 frames)
2. **Individual descriptions** (30 LLM calls)
3. **Observation merging** (1 call)
4. **Title generation** (1 call)
5. **Card creation** (1 call)

Total: 33+ LLM calls vs Gemini's 2 calls

## Prompt Templates

### Title Writing Rules
```
Good Examples:
- "Debugged auth flow in React"
- "Excel budget analysis for Q4 report"
- "Reddit rabbit hole about conspiracy theories"

Bad Examples:
- "Early morning digital drift" (too vague)
- "Extended Browsing Session" (too formal)
- "Continuing from earlier" (references other cards)
```

### Summary Writing Rules
```
Style:
- First person without "I"
- 2-3 sentences maximum
- Just facts: what, which tools, major blockers
- No filler phrases ("kicked off", "dove into")

Good Example:
"Refactored the user auth module in React, added OAuth support.
Debugged CORS issues with the backend API for an hour."

Bad Example:
"Kicked off the morning by diving into some design work before
transitioning to development tasks."
```

## Category Integration

### Category Prompt Section
The system dynamically builds category instructions:

```swift
USER CATEGORIES (choose exactly one label):
1. "Work" — Professional, school, or career-focused tasks
2. "Personal" — Intentional non-work activities
3. "Distraction" — Unplanned, aimless time sinks
4. "Idle" — User is idle for most of this period

Rules:
- Return category exactly as written
- Only use "Idle" when >50% of time is inactive
- Pick closest non-idle label when active
```

### Category Normalization
After LLM response, categories are normalized:
1. Trim whitespace
2. Case-insensitive matching
3. Handle common variations ("idle time" → "Idle")
4. Fallback to first category if no match

## Quality Control

### Timestamp Validation
```swift
// Ensure all timestamps are within video bounds
if endTime > videoDuration {
    throw TimestampExceedsDurationError
}
```

### Observation Coverage
```swift
// Require minimum coverage of video duration
let coverage = totalObservedTime / videoDuration
if coverage < 0.8 { // 80% threshold
    throw InsufficientCoverageError
}
```

### Card Deduplication
The sliding window approach handles duplicates:
1. Fetch existing cards in time range
2. Replace all cards in window with new ones
3. Clean up orphaned video files

### Error Handling
Failed analyses create error cards:
```swift
TimelineCard(
    category: "System",
    subcategory: "Error",
    title: "Processing failed",
    summary: "Failed to process X minutes... can be reprocessed"
)
```

## Prompt Optimization Tips

### For Better Condensation
- Increase minimum segment duration threshold
- Strengthen "extend by default" instruction
- Add examples of properly merged activities

### For Better Categories
- Provide detailed category descriptions
- Include specific app/website examples per category
- Use the idle threshold consistently (>50%)

### For Better Summaries
- Emphasize factual, scannable content
- Prohibit speculation about mental states
- Require specific tool/app names

## Testing Prompts

### Manual Testing
```bash
# Test with sample video
curl -X POST 'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent?key=YOUR_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "contents": [{
      "parts": [{
        "text": "YOUR_PROMPT_HERE"
      }]
    }]
  }'
```

### Debugging Output
The app logs LLM interactions:
- Request/response bodies in `llm_calls` table
- Curl commands in console (debug mode)
- Timing information for rate limit analysis

## Common Issues and Solutions

### Issue: Too Many Short Segments
**Solution**: Strengthen grouping instructions, increase minimum duration

### Issue: Wrong Categories Assigned
**Solution**: Clarify category descriptions, add specific examples

### Issue: Generic Titles
**Solution**: Require specific app names, prohibit generic terms

### Issue: Timestamps Outside Video
**Solution**: Add explicit boundary validation, repeat duration in prompt

### Issue: Missing Idle Detection
**Solution**: Define idle threshold clearly (5+ minutes), add detection examples