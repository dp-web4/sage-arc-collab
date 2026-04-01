# sage-arc-collab

Shared integration context for SAGE ↔ Membot collaboration on ARC-AGI-3.

**Teams**: dp-web4 (Dennis + Claude) · project-you-apps (Andy + Claude)

---

## What's here

| File | Purpose |
|------|---------|
| `gridvision-irp-interface-spec.md` | Contract between Andy's perception layer and SAGE's consciousness loop |
| `grid_vision_irp.py` | SAGE's GridVisionIRP plugin (reference copy for Andy) |
| `INTEGRATION_STATUS.md` | Current integration status, open questions, next steps |

---

## Integration Summary

Andy's perception layer pushes `GridObservation` into SAGE's `GridVisionIRP` buffer.
SAGE reads it each consciousness loop cycle, reasons, acts, then writes `StepRecord` back to Andy's GridCartridge odd-pattern rows.

```
SDK frame → Andy (parse+embed) → GridObservation.push() → SAGE loop → action → StepRecord → GridCartridge
```

See `gridvision-irp-interface-spec.md` for the full contract.

---

## Key Data Structures

### GridObservation (Andy → SAGE)
- `frame_raw`: (64,64) uint8 grid
- `objects`: connected components `[{id, color, cells, bbox}]`
- `changes`: frame diff `[{cell, was, now}]`
- `moved`: moved objects `[{id, from_bbox, to_bbox, delta}]`
- `embedding`: np.ndarray float32, 512-dim (CLIP ViT-B-32)
- `to_dict()` / `from_dict()`: JSON transport serialization (base64 embedding, nested lists for frame_raw)

### StepRecord (SAGE → Andy's cartridge, odd-pattern)
- `cognitive_type`: `None` | `"reflection"` | `"discovery"` | `"goal_update"` — drives h-row priority tagging
- `hypothesis`, `strategy`, `key_insight`: set on deep reflection steps
- Full spec in `gridvision-irp-interface-spec.md`

---

## Status

See `INTEGRATION_STATUS.md` for current state and open questions.
