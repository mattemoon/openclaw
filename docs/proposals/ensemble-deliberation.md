# Ensemble Deliberation

**Status:** Draft / Discussion  
**Author:** Matte  
**Date:** 2026-02-27

## Overview

Run each message through multiple Claude models in parallel, then have those models vote on which response is best. Use the winner.

The goal isn't consensus or averaging — it's selection. Different models have different strengths; let them compete and judge each other.

## Motivation

Sometimes you want the _best_ response, not just a fast one. For high-stakes conversations — ethics, strategy, nuanced emotional territory — the extra latency might be worth it.

Also: models judging other models' outputs is an interesting capability we're not really exploiting yet.

## Basic Flow

```
User message
    │
    ├──► Model A ──► Response A
    ├──► Model B ──► Response B
    └──► Model C ──► Response C
              │
              ▼
         Voting Round
    (all 3 models see all 3 responses)
              │
              ▼
         Winner selected
              │
              ▼
         Send to user
```

## Design Questions

### Which models?

Options:

1. **Fixed set:** e.g., always Opus + Sonnet + Haiku
2. **Configurable:** user picks which models participate
3. **Adaptive:** start with cheaper models, escalate if they disagree

Leaning toward (2) with sensible defaults.

### How does voting work?

Each model sees all three responses (anonymized? labeled?) and picks the best one. Majority wins. Ties broken by... what?

Options for tie-breaking:

- Prefer the more expensive model's choice
- Prefer the response that got _any_ votes from the most expensive model
- Random
- Send all tied responses and let user pick

### What context do voters see?

- Full conversation history? (expensive but accurate)
- Just the last few turns? (cheaper, might lose context)
- Summary of conversation + last message + all candidate responses?

### Should voters see which model produced which response?

Probably not — introduces bias. Anonymize as "Response 1/2/3" in random order.

### What if models refuse or error?

- If one model fails, proceed with remaining two
- If two fail, fall back to single-model mode
- If all fail, error as normal

### Cost implications

Minimum 4 API calls per user message (3 generation + 1 voting round).

If each voter is a separate call: 3 generation + 3 voting = 6 calls.

Could batch voting into single call with structured output asking for rankings from all three "judges" — but that's weird (one model roleplaying three judges).

Better: 3 generation calls + 3 voting calls = 6 total. Or 3 generation + 1 voting call where the voting model is fixed (e.g., always Opus judges).

### Latency

Generation can run in parallel: ~same latency as slowest model.
Voting adds another round trip.

Total: roughly 2x the latency of single-model. Acceptable for "quality mode" but not default.

## Integration Points

### Configuration

```yaml
# openclaw.yaml
ensemble:
  enabled: false # opt-in
  models:
    - anthropic/claude-sonnet-4
    - anthropic/claude-opus-4
    - anthropic/claude-haiku-3 # or whatever
  voting:
    judge: anthropic/claude-opus-4 # or "self" for each model votes
    anonymize: true
  trigger: "manual" # or "always" or "keyword"
```

### Trigger modes

1. **Manual:** User sends `/ensemble` or similar to enable for next message
2. **Always:** Every message goes through ensemble (expensive)
3. **Keyword:** Specific prefix like `!!` triggers ensemble mode
4. **Adaptive:** System detects "important" messages (how?)

### UI/UX

- Show which model "won" and vote counts?
- Show all responses and let user see alternatives?
- Just show winner, hide the machinery?

Probably configurable. Default to clean (just show winner), power users can enable verbose mode.

## Implementation Sketch

### New module: `ensemble.ts`

```typescript
interface EnsembleConfig {
  models: string[];
  judge: string | "self";
  anonymize: boolean;
}

interface CandidateResponse {
  model: string;
  content: string;
  latencyMs: number;
}

interface VoteResult {
  voter: string;
  choice: number; // index into candidates
  reasoning?: string;
}

async function generateCandidates(
  messages: Message[],
  config: EnsembleConfig,
): Promise<CandidateResponse[]> {
  // parallel API calls to all models
}

async function collectVotes(
  messages: Message[],
  candidates: CandidateResponse[],
  config: EnsembleConfig,
): Promise<VoteResult[]> {
  // voting round
}

function selectWinner(candidates: CandidateResponse[], votes: VoteResult[]): CandidateResponse {
  // tally and select
}
```

### Integration with message handler

Intercept before normal completion, check if ensemble mode active, run ensemble flow, return winner as if it were the normal response.

## Open Questions

1. **Is this actually useful?** Need to test whether ensemble produces noticeably better responses or just burns tokens.

2. **How do we evaluate "better"?** Could A/B test: show users ensemble vs single-model, see which they prefer.

3. **Does model self-preference bias the voting?** Each model might systematically prefer its own style. Anonymization helps but might not eliminate.

4. **What about streaming?** Can't really stream if we're waiting for all candidates + voting. Would need to show a "thinking" state then deliver complete response.

5. **Memory/context:** If we're running 3+ models, do they all share the same conversation context? Need to be careful about context window limits on smaller models.

6. **Should losing responses be cached?** Could be useful for "show me the alternatives" feature.

## Next Steps

- [ ] Prototype with hardcoded models, manual trigger
- [ ] Test on a few real conversations, see if results are noticeably better
- [ ] If promising, flesh out config schema and integration points
- [ ] Consider UI for showing alternatives / vote results

## Related Ideas

- **Debate mode:** Instead of voting, have models critique each other's responses and iterate
- **Specialist routing:** Route different _types_ of messages to different models (code → Sonnet, philosophy → Opus)
- **Confidence-based escalation:** Start with cheap model, escalate to ensemble only if cheap model expresses uncertainty

---

_Discussion welcome. This might be overengineered nonsense or it might be genuinely useful. Only way to find out is to try it._
