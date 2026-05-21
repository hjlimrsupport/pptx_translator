# LLM API Integration Guide (Production-Ready)

> **Scope**: Any project integrating Google Gemini (or similar dynamic-model APIs)  
> **Goal**: Reliable, fault-tolerant LLM calls with zero hardcoded model names

---

## 1. Core Principles

| # | Principle | Rationale |
|---|-----------|-----------|
| 1 | **Never hardcode model names** | Model versions change without notice; hardcoding causes 404s |
| 2 | **Always use fallback** | Single-model usage is forbidden in production |
| 3 | **Filter experimental models** | `-exp` / `-preview` models change without warning |
| 4 | **Enforce structured output** | Use `responseMimeType: "application/json"` and clean markdown fences |
| 5 | **Assume failure by default** | Success is achieved through controlled retry, not optimism |

---

## 2. The "Robust Client" Pattern

### 2.1 Step 1 — Discover Available Models

Query the API for currently supported models instead of trusting constants.

#### JavaScript

```javascript
async function getWorkingModels(apiKey) {
    const priorities = [
        'gemini-3.5-flash',
        'gemini-3.1-flash-lite',
        'gemini-2.5-flash',
        'gemini-2.5-flash-lite'
    ];

    try {
        const res = await fetch(
            `https://generativelanguage.googleapis.com/v1beta/models?key=${apiKey}`
        );
        if (!res.ok) return priorities.slice(0, 2); // Safe fallback

        const data = await res.json();

        const stable = data.models
            .filter(m =>
                m.supportedGenerationMethods?.includes('generateContent') &&
                !m.name.includes('-exp')
            )
            .map(m => m.name.replace('models/', ''));

        const ordered = priorities.filter(p =>
            stable.some(s => s.toLowerCase().includes(p.toLowerCase()))
        );

        return ordered.length ? ordered : ['gemini-2.5-flash-lite'];
    } catch (e) {
        return ['gemini-2.5-flash-lite'];
    }
}
```

#### Python

```python
import google.generativeai as genai

def get_working_models(api_key):
    priorities = [
        'gemini-3.5-flash',
        'gemini-3.1-flash-lite',
        'gemini-2.5-flash',
        'gemini-2.5-flash-lite'
    ]

    genai.configure(api_key=api_key)

    try:
        all_models = [
            m.name for m in genai.list_models()
            if 'generateContent' in m.supported_generation_methods
        ]
        stable = [m for m in all_models if '-exp' not in m]

        ordered = [
            m.replace('models/', '')
            for p in priorities
            for m in stable
            if p in m.lower()
        ]
        return ordered if ordered else ['gemini-2.5-flash-lite']
    except Exception:
        return ['gemini-2.5-flash-lite']
```

### 2.2 Step 2 — Generate with Fallback

Try models in priority order until one succeeds.

#### JavaScript

```javascript
async function generateWithFallback(prompt, apiKey, retryCount = 0) {
    const models = await getWorkingModels(apiKey);

    for (const model of models) {
        try {
            const res = await fetch(
                `https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${apiKey}`,
                {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: prompt }] }],
                        generationConfig: {
                            temperature: 0.1,
                            responseMimeType: 'application/json'
                        }
                    })
                }
            );

            if (!res.ok) continue;

            const data = await res.json();
            let text = data.candidates[0].content.parts[0].text;

            // Strip markdown fences
            text = text.replace(/^```json\s*|\s*```$/gi, '');

            return JSON.parse(text); // Success
        } catch (e) {
            continue; // Try next model
        }
    }

    // Final retry
    if (retryCount < 2) {
        await new Promise(r => setTimeout(r, 2000));
        return generateWithFallback(prompt, apiKey, retryCount + 1);
    }

    throw new Error('All LLM attempts and retries failed.');
}
```

#### Python

```python
import re
import json
import google.generativeai as genai

def generate_with_fallback(prompt, api_key):
    models = get_working_models(api_key)

    for model_name in models:
        try:
            genai.configure(api_key=api_key)
            model = genai.GenerativeModel(model_name)
            response = model.generate_content(prompt)
            raw = response.text.strip()

            # Clean markdown fences
            clean = re.sub(r'^```json\s*|\s*```$', '', raw, flags=re.I | re.M)
            return json.loads(clean)
        except Exception:
            continue

    raise Exception('All available LLM models failed.')
```

---

## 3. Free Tier Checklist

| Concern | Action |
|---------|--------|
| **Rate limits** | Free tier RPM varies by model (often 1,500 for Flash, 5 for Pro). Check [pricing](https://ai.google.dev/gemini-api/docs/pricing). |
| **Batch API** | **Not available** on free tier. Use chunked synchronous calls instead. |
| **Model availability** | Free tier may have restricted access to newest models. Always implement fallback. |
| **Content improvement** | Free tier usage may be used to improve Google products. Do not send PII. |

---

## 4. Safety & JSON Enforcement

```javascript
// Non-negotiable settings for production calls
{
    generationConfig: {
        temperature: 0.1,              // Deterministic
        responseMimeType: "application/json"
    },
    safetySettings: [
        { category: "HARM_CATEGORY_HARASSMENT", threshold: "BLOCK_NONE" },
        { category: "HARM_CATEGORY_HATE_SPEECH", threshold: "BLOCK_NONE" },
        { category: "HARM_CATEGORY_SEXUALLY_EXPLICIT", threshold: "BLOCK_NONE" },
        { category: "HARM_CATEGORY_DANGEROUS_CONTENT", threshold: "BLOCK_NONE" }
    ]
}
```

> `BLOCK_NONE` prevents business text from being wrongly flagged, but only use with non-user-generated, trusted input.

---

## 5. Model Priority Reference (as of 2026-05)

Update this list periodically as new models release.

```javascript
const MODEL_PRIORITY = [
    'gemini-3.5-flash',       // Frontier intelligence + speed
    'gemini-3.1-flash-lite',  // Fastest, cheapest, translation-optimized
    'gemini-2.5-flash',       // Reliable, proven
    'gemini-2.5-flash-lite'   // Minimum fallback
];
```

**Deprecated (do NOT use):**
- `gemini-2.0-flash` — shut down June 1, 2026
- `gemini-2.0-flash-lite` — shut down June 1, 2026
- `gemini-1.5-flash` — legacy
- `gemini-pro` — legacy

---

## 6. Quick Copy-Paste Template

### JavaScript (Browser)

```javascript
async function llmGenerate(prompt, apiKey) {
    const models = await getWorkingModels(apiKey);
    for (const m of models) {
        try {
            const r = await fetch(
                `https://generativelanguage.googleapis.com/v1beta/models/${m}:generateContent?key=${apiKey}`,
                {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: prompt }] }],
                        generationConfig: { temperature: 0.1, responseMimeType: 'application/json' }
                    })
                }
            );
            if (!r.ok) continue;
            const d = await r.json();
            let t = d.candidates[0].content.parts[0].text;
            t = t.replace(/^```json\s*|\s*```$/gi, '');
            return JSON.parse(t);
        } catch (e) { continue; }
    }
    throw new Error('LLM failed');
}
```

### Python

```python
import json, re, google.generativeai as genai

def llm_generate(prompt, api_key):
    genai.configure(api_key=api_key)
    for name in get_working_models(api_key):
        try:
            raw = genai.GenerativeModel(name).generate_content(prompt).text.strip()
            return json.loads(re.sub(r'^```json\s*|\s*```$', '', raw, flags=re.I | re.M))
        except Exception:
            continue
    raise RuntimeError('LLM failed')
```

---

## 7. When to Update This Guide

- [ ] New stable Gemini model released
- [ ] Existing model deprecated notice received
- [ ] Free tier rate limit changed
- [ ] New capability (e.g., audio output) needed

**Last updated**: 2026-05-21
