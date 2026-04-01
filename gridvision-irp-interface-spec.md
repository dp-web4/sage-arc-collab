# GridVisionIRP Interface Specification

**Version**: 0.1 (draft for Andy review)
**Date**: 2026-03-30
**Status**: Proposal — lock after Andy confirms

---

## Overview

This spec defines the contract between Andy's perception layer (grid → objects → embedding → cartridge) and SAGE's consciousness loop. Two integration points:

1. **GridVisionIRP** — sensor plugin that feeds observations INTO the loop
2. **GridCartridge** — memory plugin that answers queries FROM the loop

---

## Andy's Four Questions, Answered

### Q1: What data does SAGE want to write per step?

**Both structured JSON and free text.** SAGE's consciousness loop produces two kinds of output per step:

```python
# Structured (always present)
step_record = {
    "step": 14,
    "action_taken": 3,
    "action_rationale": "move_right",  # from action selection
    "salience": {
        "surprise": 0.72,      # frame change was unexpected
        "novelty": 0.45,       # seen similar patterns before
        "arousal": 0.60,       # moderate difficulty
        "reward": 0.30,        # no obvious progress
        "conflict": 0.15       # no contradictions
    },
    "metabolic_state": "FOCUS",
    "atp_spent": 0.03,
    "trust_posture": {
        "confidence": 0.65,
        "dominant_modality": "vision",
        "label": "cautious"
    },
    "policy_gate": "approved",  # or "blocked" with reason
    "timestamp": 1711814400.0
}

# Free text (when reasoning plugin fires)
reasoning_text = "Object #2 appeared after select action. Same color as goal marker in level 1. Testing: does selecting near it trigger state change?"
```

**For your odd-pattern protected rows**: Store the full `step_record` JSON + `reasoning_text`. The structured fields let you filter/query programmatically. The free text lets SAGE explain itself to its future self.

### Q2: What queries will SAGE make against the cart?

Three query types, in order of frequency:

1. **Similar state lookup** (every step, fast path)
   ```
   query: embedding of current frame
   returns: top-k similar states + their step_records
   purpose: "Have I seen this before? What happened?"
   ```

2. **Action-outcome search** (before action selection)
   ```
   query: "action 3 near object color 5" (natural language)
   returns: states where action 3 was taken near color-5 objects + outcomes
   purpose: "What happened last time I did this?"
   ```

3. **Cross-level pattern search** (at level start, slow path)
   ```
   query: embedding of first frame of new level
   returns: similar level-start frames from OTHER levels + outcomes
   purpose: "Does this level remind me of a previous one?"
   ```

All three return the same format: matched states with paired even/odd patterns. SAGE reads the odd pattern to get its own prior reasoning.

### Q3: How does GridVisionIRP receive observations — push or pull?

**Push model.** Andy's perception layer runs as a co-process. Each frame, it:

1. Receives raw grid from ARC-AGI-3 SDK
2. Runs connected component analysis
3. Embeds the frame
4. Pushes a `GridObservation` to SAGE

SAGE's consciousness loop polls for observations at the start of each cycle (step 1: sense). The push lands in a buffer; the loop reads from the buffer.

```python
# Andy's side pushes:
grid_vision_irp.submit_observation(grid_observation)

# SAGE's side polls (inside _gather_observations):
obs = grid_vision_irp.get_latest()  # non-blocking, returns None if no new frame
```

**Interface for the push**:

```python
@dataclass
class GridObservation:
    """What Andy's perception layer hands to SAGE each frame."""

    # Raw grid (for storage/embedding comparison)
    frame_raw: np.ndarray          # (64, 64) uint8, values 0-15

    # Structured perception (Andy's connected component analysis)
    objects: list                   # [{"id": int, "color": int, "cells": [[r,c],...], "bbox": [r1,c1,r2,c2]}, ...]

    # Frame diff (what changed)
    changes: list                   # [{"cell": [r,c], "was": int, "now": int}, ...]
    moved: list                     # [{"id": int, "from_bbox": [...], "to_bbox": [...], "delta": [dr,dc]}, ...]

    # Embedding (for cartridge search)
    embedding: np.ndarray           # Andy's CLIP or custom encoding

    # Context
    step_number: int
    action_taken: int               # 0 = no action yet (first frame), 1-5 = last action
    level_id: str                   # Which level/environment

    # Optional: Andy's perception notes
    perception_notes: Optional[str] = None  # "3 objects detected, 1 moved right"
```

### Q4: Object classification — raw or classified?

**Raw connected components. Let SAGE classify.**

Andy reports: objects with id, color, cells, bbox. SAGE's consciousness loop determines what each object IS (enemy, tool, wall, goal) through interaction and memory. This is the whole point — classification from experience, not from labels.

One exception: if Andy's embedding model naturally clusters similar object types (e.g., CLIP recognizes "walls" vs "agents" from visual features), include that as a `perception_notes` field, not as a hard classification. SAGE can use it as a weak signal.

---

## Integration Architecture

```
ARC-AGI-3 SDK
    │
    ▼ raw 64×64 grid
Andy's Perception Layer
    ├── Connected component analysis → objects, changes, moved
    ├── Frame embedding (CLIP / custom) → embedding vector
    └── GridCartridge write (even pattern: embedding+objects, odd pattern: reserved for SAGE)
    │
    ▼ GridObservation (push)
GridVisionIRP (SAGE plugin)
    ├── init_state(): register as vision sensor, allocate observation buffer
    ├── step(): read latest GridObservation → produce SensorObservation
    ├── energy(): inverse of frame-change magnitude (big change = low energy = high salience)
    └── halt(): after one step (no iterative refinement — perception is immediate)
    │
    ▼ SensorObservation(modality='vision', data=GridObservation)
SAGE Consciousness Loop
    ├── Step 2: SNARC salience scoring on the observation
    ├── Step 3: Metabolic state check
    ├── Step 3.5: Trust posture (vision trust shapes strategy)
    ├── Step 4: Attention selection (which observation to focus on)
    ├── Step 5: ATP budget for this cycle
    ├── Step 6: Execute plugins (reasoning, memory search)
    │   └── GridCartridge search: "seen this state before?"
    ├── Step 7: PolicyGate (approve/block proposed action)
    ├── Step 8: Learn (update SNARC experience buffer)
    └── Step 9: Act → GameActionEffector → ARC-AGI-3 action (1-5)
    │
    ▼ action + step_record + reasoning_text
GridCartridge (Andy's odd-pattern write)
    └── Store SAGE's reasoning alongside the frame for future recall
```

---

## IRP Contract Details

### GridVisionIRP (dp builds, uses Andy's perception)

```python
class GridVisionIRP(IRPPlugin):
    """Bridges Andy's grid perception into SAGE's consciousness loop."""

    def init_state(self, x0: Any, task_ctx: Dict) -> IRPState:
        """
        x0: GridObservation or None
        task_ctx: {"level_id": str, "max_steps": int, "actions": [1,2,3,4,5]}
        Returns: IRPState with x = latest GridObservation
        """

    def step(self, state: IRPState) -> IRPState:
        """
        Read latest observation from buffer.
        Compute frame diff magnitude.
        Return updated state with new observation.
        """

    def energy(self, state: IRPState) -> float:
        """
        Energy = 1.0 - normalized_change_magnitude
        Big frame change → low energy → high salience (surprising)
        No change → high energy → low salience (boring)
        """

    def halt(self, history: List[IRPState]) -> bool:
        """Always halt after one step. Perception is immediate."""
        return len(history) >= 1
```

### GridCartridge (Andy builds, SAGE queries)

Wraps Andy's paired lattice cartridge as a searchable memory:

```python
class GridCartridgeIRP(IRPPlugin):
    """Memory search over game state history."""

    def init_state(self, x0: Any, task_ctx: Dict) -> IRPState:
        """
        x0: {"query_embedding": np.ndarray, "query_text": str, "level_id": str}
        Mount the cartridge for this level (or cross-level assembly).
        """

    def step(self, state: IRPState) -> IRPState:
        """
        Search cartridge using embedding similarity (primary) + text (secondary).
        Return top-k matches with their paired odd-pattern data (SAGE's prior reasoning).
        state.x = {"results": [...], "scores": [...], "prior_reasoning": [...]}
        """

    def energy(self, state: IRPState) -> float:
        """1.0 - best_match_score. Perfect recall = 0 energy."""

    def halt(self, history: List[IRPState]) -> bool:
        """One search per cycle."""
        return len(history) >= 1
```

---

## Data Flow Per Game Step

```
1. SDK returns new frame
2. Andy's layer: parse objects, diff, embed → GridObservation
3. Andy's layer: write even pattern to GridCartridge (frame + embedding + objects)
4. GridVisionIRP: buffer receives GridObservation
5. SAGE consciousness loop runs:
   a. Sense: read GridObservation as SensorObservation(modality='vision')
   b. Salience: score on 5D SNARC (surprise from frame diff, novelty from embedding distance to recent frames)
   c. Posture: vision trust shapes strategy
   d. Select: attend to this observation (or defer if low salience)
   e. Budget: allocate ATP for reasoning + memory search
   f. Execute:
      - GridCartridgeIRP searches for similar states → returns prior reasoning
      - Reasoning plugin considers: observation + memory + game context → proposed action
   g. PolicyGate: approve action
   h. Learn: store experience in SNARC buffer
   i. Act: dispatch action (1-5) to GameActionEffector
6. Andy's layer: write odd pattern to GridCartridge (SAGE's step_record + reasoning_text)
7. SDK executes action → new frame → loop
```

---

## Embedding Dimensions

Andy's CLIP embeddings are 768-dim (Nomic). For the GridCartridge search, this is the primary similarity signal. The consciousness loop doesn't touch the embedding directly — it sees structured objects + frame diff. The embedding lives in the cartridge for Andy's search layer.

If Andy wants to experiment with custom encodings (CNN on game frames, hand-crafted features), the GridObservation.embedding field is agnostic to encoding method. Just needs to be a fixed-dimension float vector that supports cosine similarity.

---

## Open Questions (for next sync)

1. **Embedding model for competition submission**: CLIP requires ~400MB. Custom CNN could be much smaller. What fits in 32GB alongside SAGE + reasoning model?

2. **Cartridge segmentation**: One cart per level? One cart per environment? One mega-cart with level tags? Andy's modular cartridge RFC addresses this — follow that design.

3. **Step timing**: Andy's perception (parse + embed) needs to complete before SAGE's cycle. Target: perception < 50ms, SAGE cycle < 50ms, total < 100ms per step. Need to benchmark on RTX 5090 equivalent.

4. **Cross-level assembly**: When a new level starts, does SAGE request a cross-level cartridge mount? Or is cross-level search always available? ATP cost implications.

---

## Next Steps

1. **Andy**: Build GridObservation producer (grid → objects → embedding → push). Test with SDK demo games.
2. **dp**: Build GridVisionIRP skeleton (buffer + sense integration). Wire into consciousness loop.
3. **Joint**: Agree on serialization format for GridObservation (msgpack? JSON? shared memory?). Benchmark.
4. **Andy**: Build GridCartridge paired lattice format. Test write/search with synthetic game data.
5. **dp**: Build GridCartridgeIRP wrapper. Wire memory search into consciousness loop step 6.

---

*"Every recalled pattern = fewer wasted actions = higher RHAE score. Memory IS the scoring advantage." — Andy*
