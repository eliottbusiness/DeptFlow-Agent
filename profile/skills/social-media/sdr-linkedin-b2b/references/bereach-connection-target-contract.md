# BeReach Connection Target Contract

This reference locks the internal contract used by the SDR LinkedIn stack for connection requests.

## Canonical identifiers

- `linkedin_id` is the stable CRM identifier used in Airtable and deduplication.
- `bereach_profile_target` is the only value allowed to reach BeReach for connection requests.

## Construction rules

### `linkedin_id`
Use for Airtable and CRM identity only.

Build it as:
1. `profileUrn` if available
2. else `publicIdentifier`
3. else normalized `profileUrl`

Never send `linkedin_id` directly to BeReach unless an explicit adapter transforms it into `bereach_profile_target`.

### `bereach_profile_target`
Use for BeReach connection requests only.

Build it as:
1. `profileUrl` if available
2. else `url_profil`
3. else `profileUrn`
4. else `publicIdentifier`
5. else error `ERROR_MISSING_PROFILE_TARGET`

If `profileUrl` or `url_profil` is used, normalize it:
- remove query parameters
- remove trailing slash
- keep the canonical public LinkedIn profile URL form on `www.linkedin.com`, e.g. `https://www.linkedin.com/in/...`

## Required adapter contract

Function signature:

```text
build_bereach_profile_target(prospect)
```

Input shape:

```json
{
  "profileUrl": "",
  "url_profil": "",
  "profileUrn": "",
  "publicIdentifier": "",
  "linkedin_id": ""
}
```

Output shape:

```json
{
  "bereach_profile_target": "",
  "source": "profileUrl|url_profil|profileUrn|publicIdentifier|none",
  "valid": true,
  "error": ""
}
```

## Validation rules

- If `profileUrl` exists, it wins.
- Otherwise if `url_profil` exists, it wins.
- Otherwise use `profileUrn`.
- Otherwise use `publicIdentifier`.
- If no valid target exists, return `valid=false` and `error=ERROR_MISSING_PROFILE_TARGET`.
- Do not send any connection request when `valid=false`.

## BeReach connection endpoint

- `POST /connect/linkedin/profile`

Strict body:

```json
{
  "profile": "{bereach_profile_target}"
}
```

Forbidden fields in the connection body:
- `linkedin_id`
- `message`
- `note`
- `text`
- `content`

## Dry-run verification example

Input:

```json
{
  "profileUrl": "https://www.linkedin.com/in/example",
  "profileUrn": "urn:li:fsd_profile:ACoAAA...",
  "publicIdentifier": "example"
}
```

Expected output:

```json
{
  "bereach_profile_target": "https://www.linkedin.com/in/example",
  "source": "profileUrl",
  "valid": true,
  "error": ""
}
```

## Operational rule

All connection requests in the SDR pipeline must pass through this adapter before any BeReach call.
