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

---

## Primary Goal

Ensure Gemini API calls **succeed reliably** by:
- Dynamically selecting available models
- Avoiding experimental or deprecated versions
- Enforcing strict, machine-parseable JSON outputs

Failure is assumed by default.
Success is achieved through controlled fallback.

---

## 1. The "Success" Pattern (JavaScript Implementation)

This pattern is currently used in the **Office AI Suite** to handle document translations.

```javascript
/*
 * Robust Gemini API Client Pattern
 * Features: Dynamic Model Discovery, Auto-fallback, Retry Logic
 */

async function getModels(key) {
    try {
        const r = await fetch(`https://generativelanguage.googleapis.com/v1beta/models?key=${key}`);
        if (!r.ok) return ['gemini-3.7-flash', 'gemini-3.6-flash'];
        const data = await r.json();
        
        // Filter: Must support 'generateContent' and NOT be an experimental (-exp) model
        const valid = data.models
            .filter(m => m.supportedGenerationMethods.includes('generateContent') && !m.name.includes('-exp'))
            .map(m => m.name.replace('models/', ''));

        // Priority Order preference: quality first (3.7/3.6), then high-quota Lite fallbacks
        const prio = ['gemini-3.7-flash', 'gemini-3.6-flash', 'gemini-3.5-flash-lite', 'gemini-3.1-flash-lite'];
        const res = prio.filter(p => valid.some(v => v.includes(p)));
        
        return res.length > 0 ? res : ['gemini-3.7-flash'];
    } catch (e) { 
        return ['gemini-3.7-flash', 'gemini-3.6-flash']; 
    }
}

async function translateWithFallback(payload, keysRaw, retryCount = 0) {
    const keys = keysRaw.split(/[\n,]/).map(k => k.trim()).filter(k => k !== "");
    
    for (const key of keys) {
        const models = await getModels(key);
        for (const model of models) {
            try {
                const r = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/${model}:generateContent?key=${key}`, {
                    method: 'POST', 
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: payload.prompt }] }],
                        generationConfig: { temperature: 0.1, responseMimeType: "application/json" }
                    })
                });
                
                if (!r.ok) continue;
                const res = await r.json();
                
                // Clean markdown code fences if LLM adds them
                let text = res.candidates[0].content.parts[0].text.replace(/^```json\s*|\s*```$/gi, "");
                return JSON.parse(text);
            } catch (e) { 
                continue; // Try next model or next key
            }
        }
    }

    // Final Retry Logic
    if (retryCount < 2) {
        await new Promise(r => setTimeout(r, 2000));
        return await translateWithFallback(payload, keysRaw, retryCount + 1);
    }
    throw new Error("All API attempts and retries failed.");
}
```

---

## 2. Operational Rules (NON-NEGOTIABLE)

- **Do not trust model names**: Version suffixes change without notice. Always use partial matching.
- **Experimental models are forbidden**: Any model containing `-exp` must be excluded from production.
- **JSON enforcement is strict**: Markdown-wrapped JSON must be cleaned before parsing.
- **Single-model usage is forbidden**: All production calls must use fallback to a secondary model (e.g., from Flash to Pro).

---

## 3. When This Logic MUST Be Applied

- Whenever performing batch translations.
- In environments where network stability or API quotas may fluctuate.
- When multi-API key rotation is required for higher throughput.
