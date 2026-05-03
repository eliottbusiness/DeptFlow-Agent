# Gemini 2.5 Flash Quirks for SDR Message Generation

**Model**: `google/gemini-2.5-flash` (native Gemini API)  
**Observed**: 2026-05-01, profile `deptflow`, Hermes Agent

## Problem 1: Overly Concise Outputs

**Symptom**: Model returns 1-2 lines even with "4 lignes maximum" prompt.

**Example**:
```
Prompt: "Écris un message de 4 lignes pour Jean, fondateur SEO..."
Output: "Jean, en tant que fondateur d'agence SEO, tu cherches..."
```
→ Only 1 line, truncated mid-sentence.

**Cause**: Flash variant's concision bias + token limit early stop.

**Fix**:
- Use **explicit line enumeration** in prompt:
  ```
  Ligne 1 : "Jean," + accroche personnalisée
  Ligne 2 : [douleur métier]
  Ligne 3 : [solution]
  Ligne 4 : [CTA question]
  ```
- Phrase as: "EXACTEMENT 4 lignes" (not "environ 4")
- Set `maxOutputTokens ≥ 600` to avoid cut-off
- Add `stopSequences: []` (empty) to prevent premature stop

## Problem 2: Line Count Drift (5+ lines)

**Symptom**: Model produces 5 lines when asked for 3-4.

**Example**:
```
Jean,
En tant que fondateur d'agence SEO, tu cherches à stabiliser ton pipeline.
Mon système automatise ta prospection LinkedIn et génère des rendez-vous qualifiés en continu.
Sans effort manuel, 24h/24.
Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence ?
```
→ 5 lines (line 3 split across two logical parts).

**Cause**: Flash counts line breaks as separate tokens; adds extra newline.

**Fix**:
- In prompt, specify: "4 lignes EXACTEMENT — pas de cinquième ligne"
- Combine lines 3+4 in template: "Ligne 3 : [solution] + [bénéfice]. Ligne 4 : [CTA]"
- Post-process: merge lines 3+4 if >4 lines detected
- As last resort: truncate to first 4 lines (risk losing CTA)

## Problem 3: Truncated Mid-Sentence

**Symptom**: Output ends with incomplete word (e.g., "pros...").

**Cause**: `maxOutputTokens` too low for sentence boundary.

**Fix**:
- `maxOutputTokens: 800` for 4-line messages
- Monitor `finishReason` in response: if `MAX_TOKENS`, increase tokens

## Proven Prompt Template (WARM, 4 lines)

```
Tu es Alexa, agent SDR LinkedIn B2B.

Prospect : Jean, fondateur d'une agence SEO locale.

Offre : système de prospection LinkedIn automatisé générant des rendez-vous qualifiés.

Format EXIGÉ (4 lignes) :

Jean,
[Ligne 2 : accroche sur son rôle + douleur]
[Ligne 3 : solution + bénéfice principal]
[Ligne 4 : "Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence ?"]

Règles :
- Commencer par "Jean,"
- EXACTEMENT 4 lignes (compter retours \n)
- Pas "Je me permets", "J'espère", "Bonne journée"
- Ton direct, humain

Génère UNIQUEMENT les 4 lignes, commence par "Jean,".
```

**Result**: ~80% compliance (1/5 attempts still drift to 5 lines; acceptable in prod with post-merge).

## Alternative: Use gemini-2.5-pro

`gemini-2.5-pro` is less concise but:
- Returns 200 status with `finishReason: MAX_TOKENS` and NO text content in some cases (bug observed 2026-05-01)
- Slower, more expensive
- No clear advantage over tuned Flash

**Recommendation**: Stick with `gemini-2.5-flash` + prompt engineering.

## Model Availability (as of 2026-05-01)

| Model | Available | Notes |
|-------|-----------|-------|
| gemini-2.5-flash | ✅ yes | Stable, fast, concise |
| gemini-2.5-pro | ⚠️ partial | MAX_TOKENS bug in v1beta |
| gemini-2.0-flash | ✅ yes | Fallback option |
| gemini-1.5-flash | ❌ no | 404 for new accounts |
| gemini-2.5-flash-preview | ❌ no | Preview models blocked for new keys |

**Use**: `google/gemini-2.5-flash` in Hermes config.
