# Qdrant-Powered Related-Card Recommendations — Design Spec

**Date:** 2026-05-30
**Status:** Approved (design)
**Component:** `spaced-repetition-capstone-server`

## 1. Summary

Add semantic **related-card recommendations** to IntervalAI to personalize
Spanish→English vocabulary study. A multilingual embedding model converts each
flashcard into a vector; vectors are stored in an existing in-cluster **Qdrant**
pod. A new endpoint returns the cards most semantically similar to a given card,
scoped to the requesting user's own deck.

Qdrant is a **complementary vector index**, not a replacement for MongoDB.
MongoDB remains the system-of-record for users, cards, review history, and
stats. Qdrant holds one vector per card purely for similarity search and is
always reconcilable from MongoDB.

## 2. Goals / Non-Goals

### Goals
- Per-user "related cards" for a given card, by semantic similarity.
- Multilingual embeddings so ES and EN surface forms share one vector space
  (`casa`/`house`/`hogar`/`home` cluster together). Card direction is
  Spanish→English; embedding direction is irrelevant since both surfaces are
  embedded together.
- Self-hosted embedding inference on **debian-marmoset's Intel iGPU** via
  OpenVINO Model Server (OVMS).
- Additive feature: must not change or risk the existing SM-2 / ML scheduling
  or the add/answer-card flows.

### Non-Goals (explicitly out of scope)
- Using marmoset's **NVIDIA GPU** for any IntervalAI workload. The embedding
  path targets the Intel iGPU only and never the GTX 1080.
- Migrating the existing `interval_ai` interval-prediction model off marmoset's
  NVIDIA GPU. Tracked as a separate follow-up.
- Influencing the study queue / scheduling with relatedness (possible phase 2).
- Global cross-user recommendations (per-user scope only).
- Free-text semantic search UI (the embedding client supports it, but no
  endpoint is built here).

## 3. Constraints & Context

- **Stack:** Express 5, MongoDB (Atlas), Mongoose; Mocha/Chai/chai-http tests.
  Deployed on K3s (`intervalai.el-jefe.me`) via ArgoCD + Doppler + ESO + Helm.
- **Data model:** a `User` document embeds `questions[]`; each card is
  `{ _id, question, answer, ...SM-2/ML fields }`. There is no top-level cards
  collection. Card `_id` is a MongoDB ObjectId and is unique within the user.
- **Existing inference clients:** `ml/triton-client.js` (KServe v2, NVIDIA GPU
  tiers + fallback) and `ml/openvino-client.js` (OVMS, TF-Serving REST API).
  The OVMS client is the precedent for this work.
- **Qdrant:** already runs as a pod in the K3s cluster. Reached via in-cluster
  service DNS (exact service name/namespace to be confirmed at implementation).

## 4. Architecture

New units, each with a single responsibility:

| Unit | Responsibility | Depends on |
|---|---|---|
| `ml/embedding-client.js` | text → 384-dim L2-normalized vector: tokenize → OVMS `/v2/infer` → mean-pool → normalize | OVMS (Intel iGPU); `@xenova/transformers` (tokenizer only) |
| `ml/qdrant-service.js` | collection lifecycle; `upsert`, `delete`, `related()`; all `userId`-filtered | `@qdrant/js-client-rest`; embedding-client |
| `routes/questions.js` (edit) | new `GET /:id/related` endpoint | qdrant-service |
| `routes/users.js` (edit) | index seeded deck after registration (`POST /users`) | qdrant-service |
| `scripts/backfill-embeddings.js` | one-off: embed all existing users' cards into Qdrant | qdrant-service; Mongoose models |
| `config.js` (edit) | new config keys (see §9) | — |

### Why these boundaries
- `embedding-client` knows nothing about Qdrant or cards — just text→vector.
  Testable by mocking the OVMS HTTP call.
- `qdrant-service` knows nothing about HTTP routes — just card vectors and
  similarity. Testable by mocking the Qdrant client.
- Routes orchestrate; they hold no embedding or vector math.

## 5. Data Flow

### Card lifecycle reality (as-built)
Cards are **not** created one at a time. The only card-creation event is **user
registration** (`POST /users`, `routes/users.js`), which seeds a fixed 30-card
Spanish→English vocabulary set (`db/seed/questions.js`) — identical text for
every user. There are **no add/edit/delete-card endpoints**, so card content is
immutable after registration. Therefore sync reduces to: **index a user's deck
once at registration** + a **backfill** for existing users. `qdrant-service`
still exposes `upsert`/`delete` for the backfill and for future card-mutation
routes, but no edit/delete route hooks are built here (YAGNI). Because the seed
text repeats, embeddings are cached by text string within a run.

### Write path (fail-open)
1. On `POST /users`, after `User.create(newUser)` commits to MongoDB
   (**source of truth**), the new user's seeded cards are indexed.
2. For each card, `embedding-client.embed("{question} {answer}")`
   produces a 384-dim unit vector. **Both sides are embedded together** because
   cards are short word/short-phrase pairs (Spanish term + short English word):
   the English answer disambiguates polysemous Spanish terms (e.g. `banco bench`
   vs `banco bank` cluster differently) and short two-surface input yields more
   stable vectors. (If answers were long sentences, we would embed only the
   target term to avoid the long side dominating the mean-pooled vector — not the
   case here.)
3. `qdrant-service.upsert({ id: cardId, vector, payload })`.
4. Card deleted → `qdrant-service.delete(cardId)`.

If step 2 or 3 fails, the card write in step 1 still succeeds. The failure is
logged; the point is reconciled later by the backfill script. Sync is **never**
allowed to fail a user-facing card operation.

### Read path (fail-soft)
1. `GET /api/questions/:id/related?k=5` (auth required).
2. `qdrant-service.related(cardId, userId, k)` derives the card's point UUID,
   issues a Qdrant **query-by-point** using that already-stored vector, with
   `filter: must userId == <me>`, excluding the source point.
3. Results below `RELATED_MIN_SCORE` (cosine) are dropped, so a card with few
   true neighbors returns fewer than `k` rather than padding with weak matches.
   The remaining results are capped at `k`.
4. Each result's `payload.cardId` (raw ObjectId) is hydrated from MongoDB (so the
   response carries live card content/state).
4. Response: `{ related: [ { id, question, answer, score }, ... ] }`.

The read path does **not** call the embedding model — it reuses stored vectors.

## 6. Qdrant Collection

- **Name:** `intervalai_cards`
- **Vector size:** 384 (matches `paraphrase-multilingual-MiniLM-L12-v2`)
- **Distance:** Cosine
- **Point id:** a **deterministic UUID derived from the card `_id`** (Qdrant only
  accepts u64 or UUID ids — a 24-hex-char ObjectId is neither). Use UUIDv5 over
  the ObjectId hex (or zero-pad the 12 ObjectId bytes to a 16-byte UUID); the
  mapping is pure and reversible-by-recompute, so upsert stays idempotent and
  delete is a direct id removal with no separate mapping table. The raw ObjectId
  is also kept in the payload as `cardId` for MongoDB hydration.
- **Payload:** `{ userId, cardId, question, answer }` (`cardId` = raw ObjectId hex)
- **Payload index:** keyword index on `userId` for fast per-user filtering.

Collection is created idempotently on service startup if absent.

## 7. Embedding Model & Serving (Intel iGPU only)

- **Model:** `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`
  (384-dim, 50+ languages incl. EN/ES, no query/passage prefix needed, suited to
  short vocabulary strings).
- **Export:** convert to OpenVINO IR via `optimum-cli export openvino`.
- **Serving:** OVMS deployment, model name `text_embed`, served via the **KServe
  v2** API (`POST /v2/models/text_embed/infer`). Inputs `input_ids` and
  `attention_mask` (INT64); output is the last-hidden-state token-embedding
  tensor.
- **Placement:** pinned to **debian-marmoset**; requests `gpu.intel.com/i915`
  with `/dev/dri` mounted; `target_device: GPU`. Never NVIDIA.
- **Tokenization:** in Node via `@xenova/transformers` `AutoTokenizer` (tokenizer
  assets only; no model weights loaded in the API pod).
- **Post-processing:** mean-pool token embeddings using `attention_mask`, then
  L2-normalize (so Cosine == dot product).

## 8. Error Handling / Resilience

- **Write path:** fail-open. OVMS/Qdrant errors are caught, logged via Winston,
  and never propagate to the card create/edit/delete response.
- **Read path:** fail-soft. If Qdrant or the lookup fails, `/related` returns
  `200 { related: [] }`, not `500`.
- **Feature flag:** `VECTOR_SEARCH_ENABLED`. When `false`: no sync attempts on
  writes, and `/related` returns `{ related: [] }`. Lets the feature ship dark
  and be enabled per environment.
- **Auth:** `/related` reuses the existing cookie-based JWT middleware
  (`middleware/cookie-auth.js`). A user can only retrieve related cards within
  their own deck (enforced by the `userId` filter, not just by input).

## 9. Configuration (config.js additions)

| Key | Default | Purpose |
|---|---|---|
| `VECTOR_SEARCH_ENABLED` | `false` | Master on/off for the feature |
| `QDRANT_URL` | in-cluster service URL | Qdrant REST endpoint |
| `QDRANT_API_KEY` | `''` | Qdrant auth (if enabled on the pod) |
| `QDRANT_COLLECTION` | `intervalai_cards` | Collection name |
| `OVMS_EMBED_URL` | OVMS service URL | Embedding model server (KServe v2) |
| `EMBED_MODEL_NAME` | `text_embed` | OVMS model name |
| `EMBED_DIM` | `384` | Vector dimension (guards collection creation) |
| `RELATED_K_DEFAULT` | `5` | Default number of related cards returned |
| `RELATED_K_MAX` | `20` | Upper clamp on the `k` query param |
| `RELATED_MIN_SCORE` | `0.5` | Minimum cosine score; results below are dropped (tune against real embeddings; lower toward 0 to disable) |

Secrets (`QDRANT_API_KEY`, URLs) flow through Doppler + ESO like existing config.

## 10. API Contract

```
GET /api/questions/:id/related?k=5
Auth: required (cookie JWT)
200 OK
{
  "related": [
    { "id": "<cardId>", "question": "house", "answer": "casa", "score": 0.87 },
    ...
  ]
}
```
- `k` optional, default `RELATED_K_DEFAULT` (5), clamped to `RELATED_K_MAX` (20).
- Results filtered by `RELATED_MIN_SCORE`; **fewer than `k`** (or zero) may be
  returned for outlier cards or small decks — this is expected, not an error.
- `score` is cosine similarity in `[−1, 1]` (effectively `[0, 1]` for these
  embeddings), sorted descending.
- `:id` not found in the user's deck → `404`.
- Feature disabled / backend down → `200 { "related": [] }`.

## 11. Testing

Framework: Mocha + Chai + chai-http, following `test/` conventions
(`test/setup.test.js` bootstrap, separate test DB).

- **Unit — `embedding-client`:** mean-pooling + normalization correctness against
  a known token-embedding tensor (mock OVMS HTTP); verifies output is unit-norm.
- **Unit — `qdrant-service`:** `upsert`/`delete`/`related` call the Qdrant client
  with the correct collection, payload, and `userId` filter (mock client);
  `related` drops results below `RELATED_MIN_SCORE` and caps at `k`.
- **Integration — `test/related.test.js`:** `/related` happy path returns
  user-scoped results; requires auth (401 without cookie); graceful degradation
  returns `200 { related: [] }` when Qdrant is mocked unavailable; another user's
  card id yields 404.

## 12. Deployment

1. Server deps: add `@qdrant/js-client-rest`, `@xenova/transformers`, and `uuid`
   (for the deterministic ObjectId→UUIDv5 point-id mapping; or use Node's
   built-in `crypto` if avoiding the dependency).
2. OVMS `text_embed` model: OpenVINO IR artifact + OVMS model-repo entry; K8s
   manifest pinned to debian-marmoset with `gpu.intel.com/i915` + `/dev/dri`,
   `target_device: GPU`. Lives in the portfolio GitOps repo.
3. Qdrant: pod already exists; confirm in-cluster service DNS and create the
   `intervalai_cards` collection (idempotent on startup).
4. New env/secrets via Doppler + ESO.
5. Run `scripts/backfill-embeddings.js` once after `VECTOR_SEARCH_ENABLED=true`
   to index existing cards.

## 13. Rollout

1. Merge with `VECTOR_SEARCH_ENABLED=false` (ships dark; no behavior change).
2. Deploy OVMS `text_embed` + verify `/v2/models/text_embed/ready` on Intel iGPU.
3. Confirm Qdrant reachability + collection creation.
4. Enable the flag in one environment; run backfill; smoke-test `/related`.
5. Client UI integration (related-cards panel) — separate client-submodule work.

## 14. Future Work (not in this spec)
- Phase 2: fold relatedness into `getNextQuestion` study sequencing.
- Free-text semantic search endpoint.
- Repoint `interval_ai` off marmoset's NVIDIA GPU (separate task).
