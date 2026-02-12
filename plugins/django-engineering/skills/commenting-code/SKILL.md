---
name: commenting-code
description: Use when writing, reviewing, or cleaning up code comments and docstrings in Python. Covers decisions about which comments to keep and which to remove.
---

# Code Commenting Rules

## Decision Rule

> **If information can be extracted from code (function name, parameters, types, return type), the comment is USELESS. If it explains WHY, documents for non-developers (UI), or warns about non-obvious behavior, KEEP IT.**

## ❌ REMOVE - Useless Comments

### 1. Visual Separator Comments
```python
# =============================================================================
# Section Name
# =============================================================================
```
Noise. Code structure should be self-evident from organization.

### 2. Single-line Docstrings Repeating Signature
```python
def enable(self, user: str | None = None) -> None:
    """Enable this ingestor configuration."""  # REMOVE

def get(self, config_id: UUID) -> IngestorConfig | None:
    """Get an ingestor config by its ID."""  # REMOVE

def delete(self, config_id: UUID) -> bool:
    """Delete an ingestor config by ID. Returns True if deleted."""  # REMOVE
```
Function name + parameters + return type already tell the story.

### 3. Comments Stating WHAT Code Does
```python
# Loop through items  # REMOVE
for item in items:
    process(item)
```
Code already shows this.

## ✅ KEEP - Useful Comments

### 1. Django help_text on Model Fields
```python
enabled = models.BooleanField(
    help_text="Whether this ingestor is active"  # KEEP
)
```
**UI documentation for admin users**, not developer comments. Serves runtime purpose.

### 2. Contract/Interface Documentation
```python
class IngestorConfigRepository(ABC):
    """
    Repository interface for IngestorConfig persistence.

    Returns Django models directly (no mapping layer).
    """  # KEEP
```
Guides implementers. Documents design decisions and constraints.

### 3. WHY Explanations
```python
# Minimum 60s to avoid API rate limiting  # KEEP
if poll_interval_seconds < 60:
    raise ValueError(...)
```
Intent not obvious from code alone.

### 4. Side Effects / Warnings
```python
def update_cursor(self, new_cursor: str | None) -> None:
    # Also resets error_count and last_error_message  # KEEP
```
Non-obvious behavior that callers should know.

### 5. Business Rules
```python
# Health thresholds: 0 = healthy, 1-4 = warning, 5+ = failing  # KEEP
```
Domain knowledge not evident from code.

### 6. Multi-line Class Docstrings with Value
```python
class IngestorService:
    """
    Orchestrates ingestor execution.

    Responsibilities:
    - Execute ingestion cycles (polling mode)
    - Test external connections
    - Track run history and metrics
    """  # KEEP - explains purpose and responsibilities
```

### 7. NOTE/WARNING Comments
```python
# NOTE: This admin is NOT registered by default.
# Raw events have short retention (14 days).  # KEEP
```
Important warnings for developers.

## Quick Test

Before writing or reviewing a comment, ask:

1. **Can I get this info from the function signature?** → REMOVE
2. **Does it explain WHY, not WHAT?** → KEEP
3. **Is it for end users (help_text, admin UI)?** → KEEP
4. **Does it warn about non-obvious behavior?** → KEEP
5. **Is it just visual decoration?** → REMOVE

## Usage

This skill activates when:
- Reviewing code comments for quality
- Cleaning up docstrings
- Checking if comments add value
- Refactoring code documentation

Apply these rules to identify and remove useless comments while preserving meaningful documentation.
