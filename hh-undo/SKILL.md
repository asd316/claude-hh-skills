---
name: hh-undo
description: Add batch tracking and rollback to data-modifying pipelines. Records each operation batch with affected items, provides --undo CLI, maintains state auto-recovery. Use when user asks for "回退", "rollback", "undo", or when building any data pipeline that modifies persistent state.
---

# hh-undo — Data Pipeline Rollback

Every data-modifying operation should be reversible. This skill designs and adds batch tracking + undo capability.

## When to Use

- User says "支持回退吗", "rollback", "undo", "撤回"
- Building a new data pipeline that modifies persistent data
- User expresses anxiety about irreversible operations
- After `hh-diagnose` finds data corruption — add undo to prevent it happening again

## Core Concepts

### Batch
A single run of the pipeline. Uniquely identified by `batch_id` (e.g. `b-20260603-143052`).

### State File
Per-entity JSON tracking what batches have been applied:
```json
{
  "last_batch_id": "b-20260603-143052",
  "batch_history": ["b-20260602-120000", "b-20260603-143052"],
  "last_processed_input": "data/messages/group/2026-06-03_14-30-52.json"
}
```

### Undo Operation
- Remove all items created by a specific batch
- Roll back state to previous batch
- Re-trigger downstream processes (e.g. re-run report generation)

## Design Template

### 1. Batch Tracking

```python
# In generate.py or equivalent pipeline entry point
import uuid
from datetime import datetime

batch_id = f"b-{datetime.now().strftime('%Y%m%d-%H%M%S')}-{uuid.uuid4().hex[:6]}"

# Record every created/updated item with batch_id
item = {
    "id": "faq_xxx",
    "batch_id": batch_id,
    # ... other fields
}
```

### 2. State Management

```python
# state/{group_name}.json
{
    "group": "群名",
    "last_batch_id": "b-20260603-143052-abc123",
    "batch_history": [
        {"batch_id": "b-20260602-...", "items_count": 5, "timestamp": "..."},
        {"batch_id": "b-20260603-...", "items_count": 3, "timestamp": "..."}
    ],
    "last_input_cursor": "data/messages/group/2026-06-03.json"
}
```

### 3. Undo CLI

```bash
python generate.py --undo                  # Undo last batch (any group)
python generate.py --undo --group 群名      # Undo last batch for specific group
python generate.py --undo --batch b-xxx    # Undo specific batch
```

Undo logic:
```python
def undo(batch_id=None, group=None):
    batches = find_batches_to_undo(batch_id, group)
    for batch in reversed(batches):  # newest first
        items = load_items_by_batch(batch)
        delete_items(items)           # physical delete from knowledge base
        rollback_state(batch)         # restore state to previous batch
    rerun_downstream()                # re-generate reports etc.
```

### 4. Safety Checks

- **Confirmation prompt**: show what will be deleted before executing
- **Dry-run mode**: `--undo --dry-run` shows what WOULD be deleted
- **Batch is atomic**: undo all items in a batch, never partial
- **State always consistent**: after undo, state matches actual data

## Integration Pattern

```
Data Pipeline:
  generate.py
    ├── batch_id = create_batch()
    ├── for each new message:
    │     item = process(message)
    │     item["batch_id"] = batch_id
    │     save(item)
    ├── update_state(batch_id)
    ├── auto-trigger downstream  (e.g. report.py)
    └── if error → undo(batch_id) + raise

CLI:
  --undo              → undo last batch
  --undo --group X    → undo last batch for group X
  --undo --batch B    → undo specific batch
  --undo --dry-run    → preview what would be deleted
```

## Red Flags

- Undo that leaves state inconsistent with actual data
- Not recording batch_id on every created item
- Undo that doesn't re-trigger downstream processes
- No confirmation before deletion
