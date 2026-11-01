# Pronunciation Assessment Service - API Quick Reference

This document summarizes how to fetch dictionary entries via the read-only endpoints, with and without organization scoping, and how avatar-aware resolution works.

## Base URL

- Local development: `http://127.0.0.1:8000`
- All endpoints are under `/api/v1`

---

## GET /api/v1/dictionary/{engine}

List dictionary entries for a TTS engine, optionally scoped to an org, with optional global fallback.

Query params:
- `org_id` (optional): organization identifier. If omitted, only the global dictionary is returned.
- `include_global` (optional, default: `true`): when `org_id` is provided, merge global entries; org entries take precedence on conflicts.
- `limit` (default: `500`), `offset` (default: `0`): pagination window.
- `prefix`, `q` (optional): filter by word prefix or substring.

Behavior:
- No `org_id` → returns global entries only.
- `org_id` + `include_global=true` → returns org entries plus global entries; org entries override global on conflicts.
- `org_id` + `include_global=false` → returns org entries only.

Examples (assuming `BASE=http://127.0.0.1:8000`):
```bash
# Global only (no org)
curl -sS "$BASE/api/v1/dictionary/kokoro?limit=3" | jq

# Org-only (no global fallback)
curl -sS "$BASE/api/v1/dictionary/kokoro?org_id=test2&include_global=false&limit=3" | jq

# Org + global (org overrides on conflicts)
curl -sS "$BASE/api/v1/dictionary/kokoro?org_id=test2&include_global=true&limit=3" | jq

# With filters
curl -sS "$BASE/api/v1/dictionary/google?org_id=test2&include_global=true&prefix=A&limit=50" | jq
```

---

## GET /api/v1/dictionary/{engine}/word/{word}

Resolve a single word’s pronunciation with org/voice precedence rules.

Query params:
- `org_id` (optional): when present, returns the org entry if it exists; if not found in org scope this endpoint returns 404.
- `voice_id` (optional): returns a voice-specific override if set; otherwise the org/global entry.
- `avatar_id` (optional): overrides `engine` from the avatar’s provider (and may resolve a default `voice_id` if omitted).

Behavior:
- No `org_id` → returns the global entry only (404 if not present).
- With `org_id` → returns the org entry if present; to inspect the merged org+global view use the list endpoint with `include_global=true`.
- With `voice_id` → voice override (if exists) takes precedence.

Examples:
```bash
# Global only (no org)
curl -sS "$BASE/api/v1/dictionary/kokoro/word/NASA" | jq

# Org-scoped (404 if word not found in org scope)
curl -sS "$BASE/api/v1/dictionary/google/word/AWS?org_id=test2" | jq

# Voice-specific override lookup
curl -sS "$BASE/api/v1/dictionary/google/word/AWS?org_id=test2&voice_id=google_en-US-Chirp3-HD-Leda" | jq
```

---

## Avatar-aware resolution (optional)

Some clients prefer to select the engine (and voice) based on an avatar identifier. When `avatar_id` is provided the backend resolves the engine from the avatar mapping, and may also provide a default `voice_id` if one isn’t supplied.

Examples:
```bash
# Engine overridden by the avatar’s provider; merged view with org fallback
curl -sS "$BASE/api/v1/dictionary/placeholder?avatar_id=217&org_id=test2&include_global=true&limit=3" | jq

# Single-word lookup with avatar-based engine resolution
curl -sS "$BASE/api/v1/dictionary/placeholder/word/AWS?avatar_id=217&org_id=test2" | jq
```

Notes:
- Replace `placeholder` with any string; when `avatar_id` is supplied the router resolves the actual engine from the avatar profile.
- The avatar mapping is fetched once on first use (cached for ~1 hour). You can pre-warm it at startup if desired.

---

## POST /api/v1/dictionary/{engine}

Add (or update) a pronunciation entry in the specified scope.

Query/Form fields:
- `word` (required)
- `pronunciation` (required) — format depends on engine:
  - kokoro: `[WORD](/ipa/)`
  - google: bare IPA `ipa`
  - elevenlabs: English spelling
- `org_id` (optional) — when present, the entry is written under that org (the org namespace is created implicitly if it doesn’t exist yet)
- `voice_id` (optional) — for setting voice-specific entries
- `avatar_id` (optional) — engine/voice may be resolved from avatar
- `force` (optional, default `false`) — confirm overwrite when an entry already exists

Behavior:
- If `org_id` exists, the entry is added/updated in that org.
- If `org_id` does not exist, the org namespace is implicitly created and the entry is added there.
- If the same entry already exists at that scope and `force=false`, returns a `confirmation_required` payload; resend with `force=true` to overwrite.

Examples:
```bash
# Add to an org (implicit org create if needed)
curl -sS -X POST "$BASE/api/v1/dictionary/kokoro" \
  --data-urlencode "word=NASA" \
  --data-urlencode "pronunciation=[NASA](/ˈnæsə/)" \
  --data-urlencode "org_id=test2" | jq

# Overwrite when confirmation is required
curl -sS -X POST "$BASE/api/v1/dictionary/google" \
  --data-urlencode "word=Azure" \
  --data-urlencode "pronunciation=ˈæʒər" \
  --data-urlencode "org_id=test2" | jq
# If response shows confirmation_required, resend with force=true:
curl -sS -X POST "$BASE/api/v1/dictionary/google" \
  --data-urlencode "word=Azure" \
  --data-urlencode "pronunciation=ˈæʒər" \
  --data-urlencode "org_id=test2" \
  --data-urlencode "force=true" | jq

# Avatar-aware add (engine resolved from avatar)
curl -sS -X POST "$BASE/api/v1/dictionary/placeholder" \
  --data-urlencode "word=AWS" \
  --data-urlencode "pronunciation=ˈeɪˈdʌbəljuːˈɛs" \
  --data-urlencode "avatar_id=217" \
  --data-urlencode "org_id=test2" | jq
```

---

## Tips
- Engines: `kokoro` entries are stored as `[WORD](/ipa/)`; `google` entries as bare IPA (no slashes); `elevenlabs` entries as English spellings.
- For bulk merged views (org + global), prefer the list endpoint with `include_global=true`.
- For exact resolution with overrides, use the word endpoint with `voice_id`.
