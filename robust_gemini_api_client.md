---
name: Robust Gemini API Client
description: A battle-tested strategy for reliable Google Gemini API communication. Features dynamic model discovery, priority-based fallback, and clean JSON response parsing.
priority: HIGH
scope:
  - semantic_interpretation
  - report_generation
---

# Robust Gemini API Client Strategy

This skill encapsulates **real-world operational experience** integrating with Google's Generative AI.
It is designed to handle model deprecation, service instability, and unreliable output formatting.

This is a **mandatory skill** for all Gemini API usage in this project.

---

## Primary Goal

Ensure Gemini API calls **succeed reliably** by:
- Dynamically selecting available models
- Avoiding experimental or deprecated versions
- Enforcing strict, machine-parseable JSON outputs

Failure is assumed by default.
Success is achieved through controlled fallback.

---

## 1. The "Success" Pattern (Core Logic)

The core of this strategy is **Dynamic Model Discovery**.

Hardcoding model names (e.g. `gemini-pro`) is forbidden.
Instead, the client queries the API for currently available models,
filters for stable ones, and applies a priority-based selection order.

### Python Implementation

```python
import google.generativeai as genai
import re
import json

def get_working_models(api_key):
    """
    Dynamically discovers available models and sorts them by preference.
    This avoids 404 errors caused by model deprecation or version changes.
    """
    genai.configure(api_key=api_key)

    # PRIORITY ORDER:
    # - Prefer flash models for speed and cost efficiency
    # - Fall back to pro models for maximum stability
    priorities = [
        'gemini-1.5-flash',
        'gemini-flash-latest',
        'gemini-2.5-flash',
        'gemini-pro'
    ]

    try:
        # List all models capable of content generation
        all_models = [
            m.name
            for m in genai.list_models()
            if 'generateContent' in m.supported_generation_methods
        ]

        # Exclude experimental models to ensure production stability
        stable_models = [m for m in all_models if '-exp' not in m]

        # Match models using partial name comparison
        sorted_models = [
            m.replace('models/', '')
            for p in priorities
            for m in stable_models
            if p in m.lower()
        ]

        # Safe fallback if filtering yields no results
        if not sorted_models:
            return ['gemini-1.5-flash']

        return sorted_models

    except Exception:
        # Network, auth, or API-level failure fallback
        return ['gemini-1.5-flash']


def generate_with_fallback(prompt, api_key):
    """
    Attempts Gemini generation using prioritized models until one succeeds.
    Any failure (including invalid JSON output) triggers a fallback.
    """
    target_models = get_working_models(api_key)

    for model_name in target_models:
        try:
            genai.configure(api_key=api_key)
            model = genai.GenerativeModel(model_name)

            response = model.generate_content(prompt)
            raw_text = response.text.strip()

            # Remove markdown code fences if present
            clean_json = re.sub(
                r'^```json\s*|\s*```$',
                '',
                raw_text,
                flags=re.I | re.M
            )

            # JSON parsing failure is treated as a model failure
            return json.loads(clean_json)

        except Exception:
            # Try the next available model
            continue

    raise Exception("All available Gemini models failed.")


## 2. Operational Rules (NON-NEGOTIABLE)

- **Do not trust model names**
  - Partial matching is mandatory
  - Version suffixes may change without notice

- **Experimental models are forbidden**
  - Any model containing `-exp` must be excluded

- **JSON enforcement is strict**
  - Markdown-wrapped JSON is invalid
  - Partial or malformed JSON is invalid
  - JSON parsing failure == model failure

- **Single-model usage is forbidden**
  - All Gemini API calls must use fallback logic

---

## 3. When This Skill MUST Be Used

This skill is mandatory whenever:

- Google Gemini API is used
- Structured (JSON) output is required
- The task is part of a production or long-running workflow

**No exceptions.**