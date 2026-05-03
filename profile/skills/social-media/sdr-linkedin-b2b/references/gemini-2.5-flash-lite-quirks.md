# Gemini 2.5 Flash Lite — Model Selection & Testing Notes

## TL;DR

**Use `google/gemini-2.5-flash-lite` for SDR message generation, NOT `gemini-2.5-flash`.**

- ✅ `flash-lite`: produces expected 4-5 line messages, respects line-count prompts
- ❌ `flash`: tends to 1-2 line truncations, fails line-count constraints despite strong prompting
- ✅ Configuration confirmed working with GEMINI_API_KEY from Google AI Studio

## Test Matrix (2026-05-01)

| Model                 | Status  | Lines Output | Forbidden Phrase Compliance | Notes                          |
|-----------------------|---------|--------------|----------------------------|--------------------------------|
| gemini-2.5-flash      | FAIL    | 1-2 lines    | Yes                        | Too concise, ignores line count |
| gemini-2.5-flash-lite | PASS    | 4 lines      | Yes (with strict prompt)   | Respects "EXACTEMENT 4 lignes"  |
| gemini-1.5-flash      | UNAVAIL| n/a          | n/a                        | 404 error — not in model catalog |
| gemini-2.5-pro        | FAIL    | 0 text       | n/a                        | Returns finishReason: MAX_TOKENS, no text output |

## Recommended Skill Configuration

```yaml
model: google/gemini-2.5-flash-lite
provider: gemini
generationConfig:
  maxOutputTokens: 500-600
  temperature: 0.5-0.7
  topP: 0.9
```

## Prompt Strategy That Worked

The winning prompt for `flash-lite` (4-line WARM):

```
Tu es Alexa, agent SDR LinkedIn B2B.

IMPORTANT — RÈGLES STRICTES:

Prospect: Jean, Fondateur chez Agence SEO locale
Offre: Système de prospection LinkedIn autonome 24h/24 7j/7
Valeur: J'aide les fondateurs d'agences SEO locales à stabiliser leur pipeline...
CTA: Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence ?

FORMAT EXACT — 4 lignes:
Ligne 1: "Jean,"
Ligne 2: accroche courte liée à son rôle de fondateur SEO
Ligne 3: valeur en une phrase
Ligne 4: le CTA

INTERDITS ABSOLUS (ne les utilise pas):
- "En tant que"
- "Je me permets"
- "J'espère"
- "Bonne journée"
- "Sans effort manuel"

Retourne UNIQUEMENT les 4 lignes du message.
```

**Result** (4 lines, all checks passed):
```
Jean,
Ton expertise en SEO local est précieuse pour tes clients.
Notre système de prospection LinkedIn automatise la génération de leads qualifiés pour les agences comme la tienne.
Tu veux que je te montre comment ça pourrait générer des rendez-vous qualifiés pour ton agence ?
```

## Fallback: Inline Templates (When LLM Fails)

If the model drifts again, switch to inline JavaScript templates (no LLM call) as implemented in `sdr_linkedin_followup.md`. See `references/message-templates.md`.

## Model Availability Note

`gemini-2.5-flash` is available in the API but produces overly concise outputs suitable only for very short messages (< 3 lines). `gemini-2.5-flash-lite` appears to be the same underlying model with a generation bias toward fuller sentences while maintaining speed.

Both are in the same model family (Gemini 2.5 Flash), but `-lite` is the correct choice for multi-line conversational messages.
