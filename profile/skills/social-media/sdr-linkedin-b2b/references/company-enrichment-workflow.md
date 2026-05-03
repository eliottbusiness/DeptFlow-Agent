# BeReach Company Enrichment Workflow — company_size without Sales Navigator

> Audit findings (2026-05-02): what BeReach endpoints return `company_size`, how to recover it without Sales Navigator, and the policy for when it cannot be obtained.

---

## Endpoints and what they return

| Endpoint | Returns `company_size` / `employeeCountRange`? |
|----------|------------------------------------------------|
| `POST /search/linkedin/people` | No — `currentPositions[].company.name` only |
| `POST /visit/linkedin/profile` | **BLOCKED (422)** — requires live BeReach browser session, not accessible via API key alone. Payload formats `profileUrl`, `profile`, `url`, `linkedinUrl` all return 422. Pure API enrichment is completely blocked. |
| `POST /visit/linkedin/company` | **YES** — `employeeCountRange` (e.g. "51-200") — but requires `companyUrl` which cannot be obtained without successful `visit_profile` call |
| `POST /search/linkedin/companies` | No — returns `followersCount` only |
| `POST /search/linkedin/sales-nav` | **FORBIDDEN** — Sales Navigator required, account is free |

**Key finding**: `/visit/linkedin/company` is the only non-Sales-Nav endpoint that reliably returns the company size range (`employeeCountRange`).

**Key finding**: `/visit/linkedin/company` is the only non-Sales-Nav endpoint that reliably returns the company size range (`employeeCountRange`).

## Practical constraints

1. **`/visit/linkedin/profile` returns 422** — confirmed 2026-05-02 with valid API key. All payload formats tested (`profileUrl`, `profile`, `url`, `linkedinUrl`) → 422. This blocks ALL profile enrichment in pure API mode. `collect_profile()` in `bereach.py` is non-functional.

2. **`companyUrl` is not exposed** in the `search/linkedin/people` response. Even if profile enrichment worked, `companyUrl` would be needed for `visit_company` — neither is available.

3. **Search company by name is unreliable**: `POST /search/linkedin/companies` with `keywords=company_name` returns the company `profileUrl`, but multiple companies can share the same name. Not safe for matching.

4. **Sales Navigator is never available**: The account is free. Any logic that depends on Sales Navigator fields (exact employee headcount, detailed company financials) is forbidden.

## Decision policy when `company_size` is unavailable

| Situation | Policy |
|-----------|--------|
| `company_size` returned by `collect_profile` | Use it with normal ICP matching — **currently unreachable (422)** |
| `company_size` absent after `collect_profile` AND `companyUrl` available | Attempt `visit_company` call — **also blocked (requires companyUrl from failed collect_profile)** |
| `company_size` absent AND no `companyUrl` | Do not attempt additional enrichment — accept `company_size=""` with signal: "taille entreprise non renseignée (API limitée)" |
| `company_size` absent AND user ICP requires size filter | Apply "plafond WARM" — prospect cannot reach HOT without size confirmation. Never block on missing company_size as REJECT. |

**Conclusion (2026-05-02):** The entire company enrichment chain (`visit_profile` → extract `companyUrl` → `visit_company`) is blocked at the first step. The `company-enrichment-workflow.md` is therefore **inoperative** for the current API-only setup. `company_size` must come from the search result's `currentPositions[].company` or be treated as `""` with the "taille non vérifiable" signal. **This is not fixable without a BeReach browser extension session.**
| `company_size` returned by `collect_profile` | Use it with normal ICP matching |
| `company_size` absent after `collect_profile` AND `companyUrl` available | Attempt `visit_company` call |
| `company_size` absent AND no `companyUrl` | Do not attempt additional enrichment — accept `company_size=""` with signal: "taille non vérifiable" |
| `company_size` absent AND user ICP requires size filter | Apply "plafond WARM" — prospect cannot reach HOT without size confirmation |

**This is permanently blocked** in pure API mode. `visit_profile` (first step) fails at the API level — the entire chain cannot proceed.

---

## Sales Navigator — Absolute rule

**NEVER use `/search/linkedin/sales-nav`, `/search/linkedin/sales-nav/people`, or `/search/linkedin/sales-nav/companies`.**

The OpenAPI spec tags these as `salesNav` requiring an active Sales Navigator subscription. The account used (`brc_6036...`) is a free account. Any logic depending on Sales Navigator fields must be removed or avoided.