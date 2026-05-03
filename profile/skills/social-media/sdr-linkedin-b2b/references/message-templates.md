# Message Templates Reference — SDR LinkedIn B2B

## Inline Templates (Strict Format — EXACT requirements)

These JavaScript template strings guarantee exact line counts and zero forbidden phrases. Use in `generate_message` script step.

**MANDATORY RULES**:
- WARM: EXACTLY 4 lines, no exceptions
- HOT: EXACTLY 5 lines, no exceptions
- VERY_HOT: EXACTELY 5 lines, no exceptions
- Never use: "en tant que", "je me permets", "j'espère", "bonne journée", "bien à vous", "sans effort manuel", "ton rôle de"
- Max 140 characters per line
- Start with first name only: "Prénom,"
- End with `{OFFRE_CTA}` (must be a question)

---

### WARM (4 lines exact)

```javascript
message = `${prenom},\n` +
          `J'ai vu que tu développes une agence SEO locale.\n` +
          `Je peux t'aider à stabiliser ton pipeline avec des rendez-vous qualifiés via LinkedIn.\n` +
          `${OFFRE_CTA}`;
```

**Line-by-line specification**:
- Line 1: `Jean,` (first name from prospect)
- Line 2: Fixed phrase: "J'ai vu que tu développes une agence SEO locale."
- Line 3: Fixed phrase: "Je peux t'aider à stabiliser ton pipeline avec des rendez-vous qualifiés via LinkedIn."
- Line 4: `{OFFRE_CTA}` (from environment variable — must be a question)

**Validation checklist**:
- [ ] Exactly 4 non-empty lines
- [ ] No banned phrases anywhere
- [ ] Every line ≤ 140 characters
- [ ] Ends with `?` (since OFFRE_CTA is a question)

---

### HOT (5 lines exact)

```javascript
const signal = (prospect.signal_detecte || "prospection LinkedIn").substring(0, 80);
message = `${prenom},\n` +
          `J'ai vu ton activité récente autour de ${signal}.\n` +
          `Ça montre que l'acquisition reste un sujet important pour ton agence.\n` +
          `Je peux t'aider à créer un flux régulier de rendez-vous qualifiés via LinkedIn.\n` +
          `${OFFRE_CTA}`;
```

**Line-by-line specification**:
- Line 1: `Jean,`
- Line 2: "J'ai vu ton activité récente autour de {signal_detecte}." (signal truncated to 80 chars)
- Line 3: "Ça montre que l'acquisition reste un sujet important pour ton agence."
- Line 4: "Je peux t'aider à créer un flux régulier de rendez-vous qualifiés via LinkedIn."
- Line 5: `{OFFRE_CTA}`

**Validation checklist**:
- [ ] Exactly 5 non-empty lines
- [ ] `{signal_detecte}` replaced (truncated if needed)
- [ ] No banned phrases
- [ ] Every line ≤ 140 characters
- [ ] Ends with `?`

---

### VERY_HOT (5 lines exact)

```javascript
const signal = (prospect.signal_detecte || "pipeline instable").substring(0, 80);
message = `${prenom},\n` +
          `J'ai vu ton signal autour de ${signal}.\n` +
          `Si ton pipeline dépend encore beaucoup du bouche-à-oreille, ça peut vite devenir instable.\n` +
          `Mon système aide les agences SEO locales à obtenir des rendez-vous qualifiés via LinkedIn.\n` +
          `${OFFRE_CTA}`;
```

**Line-by-line specification**:
- Line 1: `Jean,`
- Line 2: "J'ai vu ton signal autour de {signal_detecte}." (signal truncated to 80 chars)
- Line 3: "Si ton pipeline dépend encore beaucoup du bouche-à-oreille, ça peut vite devenir instable."
- Line 4: "Mon système aide les agences SEO locales à obtenir des rendez-vous qualifiés via LinkedIn."
- Line 5: `{OFFRE_CTA}`

**Validation checklist**:
- [ ] Exactly 5 non-empty lines
- [ ] `{signal_detecte}` replaced (truncated if needed)
- [ ] No banned phrases
- [ ] Every line ≤ 140 characters
- [ ] Ends with `?`

---

## Forbidden Phrases (Never Use — Strictly Enforced)

The following phrases MUST NEVER appear in any generated message, even in comments:

- `"en tant que"` → use direct address or omit
- `"je me permets"` → omit entirely
- `"j'espère"` → omit entirely
- `"j'espère que vous allez bien"` → omit entirely
- `"bonne journée"` → omit entirely
- `"bien à vous"` → omit entirely
- `"sans effort manuel"` → use "automatisé", "sans charge", or omit
- `"ton rôle de"` → omit entirely
- Any sales brochure language: "Je suis", "Nous sommes", "Notre mission", "nous proposons"

---

## Post-Processing Safety Net (Optional but Recommended)

Even with inline templates, add a cleanup pass:

```javascript
const banned = ["en tant que", "je me permets", "j'espère", "bonne journée", "bien à vous", "sans effort manuel", "ton rôle de"];
let cleaned = message;
for (const phrase of banned) {
  if (cleaned.toLowerCase().includes(phrase)) {
    const lines = cleaned.split('\n');
    cleaned = lines.filter(l => !l.toLowerCase().includes(phrase)).join('\n');
  }
}
return cleaned.trim();
```

---

## Line-Count Enforcement Table

| Score    | Target Lines | Allowance | Template Method          |
|----------|--------------|-----------|--------------------------|
| WARM     | 4            | ±0        | 4-line string concat     |
| HOT      | 5            | ±0        | 5-line string concat     |
| VERY_HOT | 5            | ±0        | 5-line string concat     |

**Enforcement**: `message.split('\n').filter(l => l.trim() !== "").length` must equal target exactly. No empty lines allowed.

---

## Customization Variables

- `${prenom}` — prospect first name (required)
- `${signal_detecte}` — from `prospect.signal_detecte` (HOT/VERY_HOT only, truncate to 80 chars)
- `${OFFRE_CTA}` — call-to-action from `.env` (must end with `?`)
- `${OFFRE_NOM}` — offer name (not used in current templates but available)
- `${OFFRE_VALEUR}` — offer value (not used in current templates but available)

**Note**: Current templates use only `${prenom}`, `${signal_detecte}`, and `${OFFRE_CTA}`. Do NOT add `${titre}` or `${entreprise}` — they are not in the mandatory templates.

---

## Revision History

- **2026-05-01** — Enforced exact templates per user specification. Removed LLM-based generation for WARM/HOT/VERY_HOT. Templates now inline, fixed, validated. Banned phrases updated.
