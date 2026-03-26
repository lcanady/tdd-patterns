---
id: agentic-unsafe-model-load
domain: agentic
severity: critical
stack: "python, node.js"
date_added: 2026-03-26
project: tdd-audit
---

# Unsafe Model Load (pickle / torch.load)

## Problem
`torch.load()` without `weights_only=True`, `pickle.load()`, and similar deserialisation functions execute arbitrary code embedded in the model file. A malicious `.pt`, `.pkl`, or `.pkl.gz` file can achieve RCE when loaded.

```python
# WRONG
import torch
model = torch.load('model.pt')          # executes arbitrary __reduce__

import pickle
data = pickle.load(open('data.pkl', 'rb'))   # RCE
```

## Root Cause
Model files are treated like benign binary blobs. The PyTorch default until v2.0 was unsafe deserialization; older code and tutorials still show the unsafe form.

## Fix
```python
# Safe — weights only, no arbitrary code execution
import torch
model = torch.load('model.pt', weights_only=True)

# For arbitrary objects, use safetensors instead of pickle
from safetensors.torch import load_file
model_state = load_file('model.safetensors')
```

For Node.js ONNX loading, verify model checksums (SHA-256) before inference.

## Test
```python
def test_torch_load_uses_weights_only(monkeypatch):
    calls = []
    original = torch.load
    monkeypatch.setattr(torch, 'load', lambda *a, **kw: calls.append(kw) or original(*a, **kw))
    load_my_model('model.pt')
    assert all(c.get('weights_only') is True for c in calls), "weights_only must be True"
```

## Detection
```
torch\.load\([^)]*\)(?!\s*,\s*weights_only\s*=\s*True)
pickle\.load\(
pickle\.loads\(
joblib\.load\(
np\.load\([^)]*allow_pickle\s*=\s*True
```
