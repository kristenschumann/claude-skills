---
name: scope
version: 1.0.0
description: "Declares EXPAND/HOLD/REDUCE mode before planning sessions, monitors for scope drift, and maintains a parked ideas list"
---
Declare a scope mode before planning. This prevents scope drift during strategy sessions.

## Instructions

Parse $ARGUMENTS for: topic and mode. Format: `{topic} {MODE}` where MODE is one of EXPAND, HOLD, or REDUCE.

If $ARGUMENTS is missing or incomplete, ask for both the topic and the mode.

### Modes

- **EXPAND** — Exploring new possibilities. Brainstorming encouraged. New ideas welcome. Challenge assumptions.
- **HOLD** — Refining what exists. No new ideas. Optimize the current plan. Tighten execution details.
- **REDUCE** — Cutting. Everything must justify its existence. If it's not essential, it's gone.

### Step 1: Acknowledge Scope

State the scope declaration clearly:

```
## Scope Declaration

**Topic:** {topic}
**Mode:** {MODE}

### In Scope
{2-4 bullets of what this mode allows, specific to the topic}

### NOT in Scope
{3-5 bullets of what this mode explicitly excludes, specific to the topic}
```

Generate the "NOT in scope" section based on the mode:
- EXPAND: Exclude implementation details, timelines, resource constraints. Focus on what's possible, not what's practical yet.
- HOLD: Exclude new product ideas, new partnerships, new revenue lines. Focus only on what's already in motion.
- REDUCE: Exclude "nice to haves", future phases, stretch goals, anything without clear ROI evidence. In REDUCE mode, apply Elon's 5-step efficiency framework as the cutting lens: (1) Are the requirements actually dumb? Who said this had to exist? (2) What can be deleted entirely? (3) What can be simplified? (4) What slows cycle time? (5) What should be automated — but only after steps 1-4.

### Step 2: Monitor for Drift

Throughout the rest of this conversation, actively monitor for scope drift. If the user or you introduce an idea that falls outside the declared scope:

1. Interrupt immediately with: **"Scope check: That's a {EXPAND/HOLD/REDUCE} idea. You declared {current MODE}. Switch modes or park it?"**
2. If the user wants to switch modes, re-declare scope with the new mode.
3. If the user says "park it", acknowledge and continue in the current mode.

### Integration with Critique Skills

If `/executive-critique` (or `/nick-critique` / `/john-critique`) is used later in this conversation:
- In HOLD mode: Flag any critique suggestions that propose new initiatives (those are EXPAND ideas).
- In REDUCE mode: Flag any critique suggestions that don't directly cut or simplify (those are HOLD or EXPAND ideas).
- In EXPAND mode: Flag any critique suggestions that prematurely constrain options (those are HOLD ideas).

### Parked Ideas

Maintain a running list of parked ideas throughout the conversation. When the user asks to see parked ideas or when the conversation naturally concludes, surface them:

```
## Parked Ideas (raised during {MODE} session)
- {idea 1} — suggested mode: {appropriate mode}
- {idea 2} — suggested mode: {appropriate mode}
```
