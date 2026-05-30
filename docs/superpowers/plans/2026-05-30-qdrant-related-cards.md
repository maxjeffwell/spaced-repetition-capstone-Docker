# Qdrant Related-Card Recommendations — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a per-user, multilingual semantic "related cards" feature to IntervalAI, backed by Qdrant as a complementary vector index alongside MongoDB.

**Architecture:** MongoDB stays the system-of-record. One embedding per card (text = `"{question} {answer}"`) is stored in an existing in-cluster Qdrant pod. Embeddings are produced by a multilingual model served on debian-marmoset's Intel iGPU via OpenVINO Model Server (OVMS), never marmoset's NVIDIA GPU. A new `GET /api/questions/:id/related` endpoint returns the nearest cards in the user's own deck. Cards are indexed at user registration (the only card-creation event) and via a one-off backfill. Writes are fail-open; reads are fail-soft; everything is behind `VECTOR_SEARCH_ENABLED`.

**Tech Stack:** Node.js, Express 4, Mongoose 8, Mocha + Chai + chai-http; `@qdrant/js-client-rest`, `@xenova/transformers` (tokenizer only), `uuid`; OpenVINO Model Server (KServe v2 REST); Qdrant.

**Spec:** `docs/superpowers/specs/2026-05-30-qdrant-related-cards-design.md`

**Working directory for all paths below:** `spaced-repetition-capstone-server/` (the server submodule). Commands assume you are `cd`'d into it.

---

## File Structure

| File | Status | Responsibility |
|---|---|---|
| `config.js` | modify | Add vector-search config keys |
| `ml/card-id.js` | create | Deterministic MongoDB ObjectId → UUIDv5 point-id mapping |
| `ml/embedding-client.js` | create | text → 384-dim unit vector (tokenize → OVMS infer → mean-pool → L2-normalize) |
| `ml/qdrant-service.js` | create | Collection lifecycle; `upsertCard`, `deleteCard`, `indexUserDeck`, `related` |
| `routes/questions.js` | modify | New `GET /:id/related` endpoint |
| `routes/users.js` | modify | Index seeded deck after `POST /users` (fail-open) |
| `index.js` | modify | Ensure Qdrant collection on startup (fail-open) |
| `scripts/backfill-embeddings.js` | create | One-off: embed all existing users' cards |
| `test/card-id.test.js` | create | Unit tests for point-id mapping |
| `test/embedding-client.test.js` | create | Unit tests for mean-pool + normalize |
| `test/qdrant-service.test.js` | create | Unit tests for service (mock Qdrant client) |
| `test/related.test.js` | create | Integration tests for `/related` + registration hook |
| `k8s/ovms/` | create | OVMS deployment manifests (Intel iGPU). May be relocated to the portfolio GitOps repo. |

**Conventions to follow (already in this codebase):**
- Logger: `const logger = require('../utils/logger').child('Name');` then `logger.info/warn/debug/error(msg, { meta })`.
- Errors: `const { NotFoundError, BadRequestError } = require('../utils/errors');` then `throw new NotFoundError('...')` inside a `try { } catch (err) { next(err); }`.
- Config: read from `require('./config')` (NOT `process.env` directly in app code).
- Tests: import the app with `const { app } = require('../index');`, gate DB tests with `if (!isDatabaseConnected()) this.skip();`. Authenticate by minting `generateAccessToken(user)` (from `lib/auth/jwt`) and sending it as the `accessToken` cookie (`.set('Cookie', \`accessToken=${token}\`)`) — the live `requireAuth` reads a cookie, not a Bearer header.

---

## Task 1: Add dependencies

**Files:**
- Modify: `package.json` (via npm)

- [ ] **Step 1: Install runtime dependencies**

Run:
```bash
npm install @qdrant/js-client-rest @xenova/transformers uuid
```
Expected: three packages added to `dependencies` in `package.json`; `npm install` exits 0.

- [ ] **Step 2: Verify they load**

Run:
```bash
node -e "require('@qdrant/js-client-rest'); require('uuid'); console.log('deps ok')"
```
Expected: prints `deps ok` with no error. (`@xenova/transformers` is ESM and loaded lazily via dynamic `import()` in the embedding client, so we do not `require` it here.)

- [ ] **Step 3: Commit**

```bash
git add package.json package-lock.json
git commit -m "build: add qdrant, transformers, uuid deps for related-cards"
```

---

## Task 2: Vector-search configuration

**Files:**
- Modify: `config.js:44` (extend the exported object)
- Test: `test/config-vector.test.js` (create)

- [ ] **Step 1: Write the failing test**

Create `test/config-vector.test.js`:
```javascript
'use strict';
const chai = require('chai');
const expect = chai.expect;

describe('Vector search config', function() {
  it('exposes vector-search defaults', function() {
    delete require.cache[require.resolve('../config')];
    const config = require('../config');
    expect(config.QDRANT_COLLECTION).to.equal('intervalai_cards');
    expect(config.EMBED_DIM).to.equal(384);
    expect(config.EMBED_MODEL_NAME).to.equal('text_embed');
    expect(config.RELATED_K_DEFAULT).to.equal(5);
    expect(config.RELATED_K_MAX).to.equal(20);
    expect(config.RELATED_MIN_SCORE).to.be.a('number');
    expect(config.VECTOR_SEARCH_ENABLED).to.equal(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_ENV=test npx mocha --exit test/config-vector.test.js`
Expected: FAIL — `expected undefined to equal 'intervalai_cards'`.

- [ ] **Step 3: Add the config keys**

In `config.js`, change the end of the exported object (currently ending at line 44 with `AI_API_KEY: process.env.AI_API_KEY || ''`). Add a trailing comma after that line and append:
```javascript
  AI_API_KEY: process.env.AI_API_KEY || '',

  // Vector search (Qdrant + OVMS embeddings) — see docs/superpowers/specs/2026-05-30-qdrant-related-cards-design.md
  VECTOR_SEARCH_ENABLED: process.env.VECTOR_SEARCH_ENABLED === 'true',
  QDRANT_URL: process.env.QDRANT_URL || 'http://qdrant:6333',
  QDRANT_API_KEY: process.env.QDRANT_API_KEY || '',
  QDRANT_COLLECTION: process.env.QDRANT_COLLECTION || 'intervalai_cards',
  OVMS_EMBED_URL: process.env.OVMS_EMBED_URL || 'http://ovms-embed:8000',
  EMBED_MODEL_NAME: process.env.EMBED_MODEL_NAME || 'text_embed',
  EMBED_OUTPUT_NAME: process.env.EMBED_OUTPUT_NAME || 'last_hidden_state',
  EMBED_TOKENIZER: process.env.EMBED_TOKENIZER || 'Xenova/paraphrase-multilingual-MiniLM-L12-v2',
  EMBED_DIM: parseInt(process.env.EMBED_DIM, 10) || 384,
  RELATED_K_DEFAULT: parseInt(process.env.RELATED_K_DEFAULT, 10) || 5,
  RELATED_K_MAX: parseInt(process.env.RELATED_K_MAX, 10) || 20,
  RELATED_MIN_SCORE: process.env.RELATED_MIN_SCORE !== undefined
    ? parseFloat(process.env.RELATED_MIN_SCORE)
    : 0.5
```
(Note: the line that was `AI_API_KEY: ... || ''` must now end with a comma.)

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_ENV=test npx mocha --exit test/config-vector.test.js`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add config.js test/config-vector.test.js
git commit -m "feat: add vector-search config keys"
```

---

## Task 3: Deterministic card point-id mapping

Qdrant point ids must be a u64 or a UUID; a Mongo ObjectId (24-hex) is neither. Map it to a deterministic UUIDv5 so upsert/delete are idempotent.

**Files:**
- Create: `ml/card-id.js`
- Test: `test/card-id.test.js`

- [ ] **Step 1: Write the failing test**

Create `test/card-id.test.js`:
```javascript
'use strict';
const chai = require('chai');
const expect = chai.expect;
const { cardPointId } = require('../ml/card-id');

describe('cardPointId', function() {
  const UUID_RE = /^[0-9a-f]{8}-[0-9a-f]{4}-5[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/;

  it('produces a valid v5 UUID', function() {
    expect(cardPointId('507f1f77bcf86cd799439011')).to.match(UUID_RE);
  });
  it('is deterministic for the same id', function() {
    expect(cardPointId('507f1f77bcf86cd799439011'))
      .to.equal(cardPointId('507f1f77bcf86cd799439011'));
  });
  it('differs for different ids', function() {
    expect(cardPointId('507f1f77bcf86cd799439011'))
      .to.not.equal(cardPointId('507f1f77bcf86cd799439012'));
  });
  it('accepts ObjectId-like objects via String()', function() {
    const fake = { toString: () => '507f1f77bcf86cd799439011' };
    expect(cardPointId(fake)).to.equal(cardPointId('507f1f77bcf86cd799439011'));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_ENV=test npx mocha --exit test/card-id.test.js`
Expected: FAIL — `Cannot find module '../ml/card-id'`.

- [ ] **Step 3: Write the implementation**

Create `ml/card-id.js`:
```javascript
'use strict';

const { v5: uuidv5 } = require('uuid');

// Fixed namespace UUID for IntervalAI card point ids. Constant on purpose:
// changing it would orphan every existing Qdrant point. Must be a valid
// RFC-4122 UUID (uuid v14's parse() rejects non-compliant version nibbles).
const CARD_NAMESPACE = 'f1d2c3b4-5a6e-4b7c-8d9e-0a1b2c3d4e5f';

/**
 * Map a MongoDB card ObjectId to a deterministic UUIDv5 usable as a Qdrant
 * point id. Pure and stable: same ObjectId always yields the same UUID.
 * @param {string|object} cardObjectId
 * @returns {string} UUIDv5
 */
function cardPointId(cardObjectId) {
  return uuidv5(String(cardObjectId), CARD_NAMESPACE);
}

module.exports = { cardPointId, CARD_NAMESPACE };
```

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_ENV=test npx mocha --exit test/card-id.test.js`
Expected: PASS (4 passing).

- [ ] **Step 5: Commit**

```bash
git add ml/card-id.js test/card-id.test.js
git commit -m "feat: deterministic ObjectId to UUIDv5 card point-id mapping"
```

---

## Task 4: Embedding client (pure pooling first)

Split the embedding math (pure, testable) from the I/O (tokenizer + OVMS).

**Files:**
- Create: `ml/embedding-client.js`
- Test: `test/embedding-client.test.js`

- [ ] **Step 1: Write the failing test (pure pooling/normalize)**

Create `test/embedding-client.test.js`:
```javascript
'use strict';
const chai = require('chai');
const expect = chai.expect;
const { meanPoolNormalize } = require('../ml/embedding-client');

function l2(v) { return Math.sqrt(v.reduce((s, x) => s + x * x, 0)); }

describe('meanPoolNormalize', function() {
  it('ignores masked (0) tokens and returns a unit vector', function() {
    // two tokens kept, one padding token masked out
    const tokenEmbeddings = [
      [1, 0, 0],
      [0, 1, 0],
      [9, 9, 9]   // masked, must be ignored
    ];
    const attentionMask = [1, 1, 0];
    const out = meanPoolNormalize(tokenEmbeddings, attentionMask);
    expect(out).to.have.lengthOf(3);
    expect(l2(out)).to.be.closeTo(1, 1e-6);
    // mean of kept = [0.5,0.5,0] -> normalized -> [~0.707,~0.707,0]
    expect(out[0]).to.be.closeTo(out[1], 1e-6);
    expect(out[2]).to.be.closeTo(0, 1e-6);
  });

  it('handles an all-masked input without NaN', function() {
    const out = meanPoolNormalize([[1, 2, 3]], [0]);
    out.forEach(v => expect(Number.isNaN(v)).to.equal(false));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_ENV=test npx mocha --exit test/embedding-client.test.js`
Expected: FAIL — `Cannot find module '../ml/embedding-client'`.

- [ ] **Step 3: Write the implementation**

Create `ml/embedding-client.js`:
```javascript
'use strict';

const axios = require('axios');
const config = require('../config');
const logger = require('../utils/logger').child('EmbeddingClient');

/**
 * Mean-pool token embeddings using the attention mask, then L2-normalize.
 * @param {number[][]} tokenEmbeddings  seqLen x dim
 * @param {number[]} attentionMask      seqLen (1 = keep, 0 = pad)
 * @returns {number[]} dim-length unit vector
 */
function meanPoolNormalize(tokenEmbeddings, attentionMask) {
  const dim = tokenEmbeddings[0].length;
  const pooled = new Array(dim).fill(0);
  let maskSum = 0;
  for (let t = 0; t < tokenEmbeddings.length; t++) {
    const m = attentionMask[t] || 0;
    if (!m) continue;
    maskSum += m;
    const row = tokenEmbeddings[t];
    for (let d = 0; d < dim; d++) pooled[d] += row[d] * m;
  }
  const denom = maskSum > 0 ? maskSum : 1e-9;
  for (let d = 0; d < dim; d++) pooled[d] /= denom;
  let norm = 0;
  for (let d = 0; d < dim; d++) norm += pooled[d] * pooled[d];
  norm = Math.sqrt(norm) || 1e-9;
  return pooled.map(v => v / norm);
}

class EmbeddingClient {
  constructor(opts = {}) {
    this.baseUrl = opts.baseUrl || config.OVMS_EMBED_URL;
    this.modelName = opts.modelName || config.EMBED_MODEL_NAME;
    this.outputName = opts.outputName || config.EMBED_OUTPUT_NAME;
    this.tokenizerName = opts.tokenizerName || config.EMBED_TOKENIZER;
    this.tokenizer = null;
  }

  // Lazy-load the ESM-only tokenizer (no model weights, just vocab/merges).
  async _getTokenizer() {
    if (this.tokenizer) return this.tokenizer;
    const { AutoTokenizer } = await import('@xenova/transformers');
    this.tokenizer = await AutoTokenizer.from_pretrained(this.tokenizerName);
    return this.tokenizer;
  }

  /**
   * Embed a single text into a 384-dim unit vector.
   * @param {string} text
   * @returns {Promise<number[]>}
   */
  async embed(text) {
    const tokenizer = await this._getTokenizer();
    const enc = await tokenizer(text, { add_special_tokens: true });
    // enc.input_ids / enc.attention_mask are Tensors; flatten to plain arrays.
    const ids = Array.from(enc.input_ids.data, Number);
    const mask = Array.from(enc.attention_mask.data, Number);
    const seqLen = ids.length;

    const resp = await axios.post(
      `${this.baseUrl}/v2/models/${this.modelName}/infer`,
      {
        inputs: [
          { name: 'input_ids', shape: [1, seqLen], datatype: 'INT64', data: ids },
          { name: 'attention_mask', shape: [1, seqLen], datatype: 'INT64', data: mask }
        ]
      },
      { timeout: 10000 }
    );

    const out = (resp.data.outputs || []).find(o => o.name === this.outputName)
      || resp.data.outputs[0];
    if (!out) throw new Error('OVMS returned no outputs');

    // out.shape is [1, seqLen, dim]; out.data is a flat row-major array.
    const dim = out.shape[out.shape.length - 1];
    const tokenEmbeddings = [];
    for (let t = 0; t < seqLen; t++) {
      tokenEmbeddings.push(out.data.slice(t * dim, (t + 1) * dim));
    }
    return meanPoolNormalize(tokenEmbeddings, mask);
  }
}

// Export singleton + the pure helper + the class (for tests/custom instances).
const embeddingClient = new EmbeddingClient();
module.exports = embeddingClient;
module.exports.meanPoolNormalize = meanPoolNormalize;
module.exports.EmbeddingClient = EmbeddingClient;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_ENV=test npx mocha --exit test/embedding-client.test.js`
Expected: PASS (2 passing).

- [ ] **Step 5: Commit**

```bash
git add ml/embedding-client.js test/embedding-client.test.js
git commit -m "feat: embedding client with mean-pool + L2-normalize over OVMS"
```

---

## Task 5: Qdrant service

**Files:**
- Create: `ml/qdrant-service.js`
- Test: `test/qdrant-service.test.js`

- [ ] **Step 1: Write the failing test (mock the Qdrant client + embedding client)**

Create `test/qdrant-service.test.js`:
```javascript
'use strict';
const chai = require('chai');
const expect = chai.expect;
const { QdrantService } = require('../ml/qdrant-service');
const { cardPointId } = require('../ml/card-id');

describe('QdrantService', function() {
  let svc, calls;

  beforeEach(function() {
    calls = { upsert: [], delete: [], recommend: [] };
    svc = new QdrantService();
    svc.enabled = true;
    svc.collection = 'test_cards';
    // Stub the Qdrant REST client
    svc.client = {
      upsert: async (coll, body) => { calls.upsert.push({ coll, body }); },
      delete: async (coll, body) => { calls.delete.push({ coll, body }); },
      recommend: async (coll, body) => {
        calls.recommend.push({ coll, body });
        return [
          { id: 'x', score: 0.9, payload: { cardId: 'c1', question: 'casa', answer: 'house' } },
          { id: 'y', score: 0.2, payload: { cardId: 'c2', question: 'sol', answer: 'sun' } }
        ];
      }
    };
    // Stub embeddings (avoid network/tokenizer)
    svc.embeddingClient = { embed: async () => [0.1, 0.2, 0.3] };
  });

  it('upsertCard writes a point keyed by the card UUID with userId payload', async function() {
    await svc.upsertCard({ _id: '507f1f77bcf86cd799439011', question: 'casa', answer: 'house' }, 'user1');
    expect(calls.upsert).to.have.lengthOf(1);
    const pt = calls.upsert[0].body.points[0];
    expect(pt.id).to.equal(cardPointId('507f1f77bcf86cd799439011'));
    expect(pt.payload).to.deep.include({ userId: 'user1', cardId: '507f1f77bcf86cd799439011' });
    expect(pt.vector).to.deep.equal([0.1, 0.2, 0.3]);
  });

  it('deleteCard removes by mapped UUID', async function() {
    await svc.deleteCard('507f1f77bcf86cd799439011');
    expect(calls.delete[0].body.points[0]).to.equal(cardPointId('507f1f77bcf86cd799439011'));
  });

  it('related filters by userId, applies score floor, caps k', async function() {
    const out = await svc.related('507f1f77bcf86cd799439011', 'user1', 5, 0.5);
    const body = calls.recommend[0].body;
    expect(body.filter.must[0]).to.deep.equal({ key: 'userId', match: { value: 'user1' } });
    expect(body.positive[0]).to.equal(cardPointId('507f1f77bcf86cd799439011'));
    // 0.2 result dropped by score floor
    expect(out).to.have.lengthOf(1);
    expect(out[0]).to.deep.equal({ cardId: 'c1', question: 'casa', answer: 'house', score: 0.9 });
  });

  it('related returns [] when disabled', async function() {
    svc.enabled = false;
    const out = await svc.related('507f1f77bcf86cd799439011', 'user1', 5, 0.5);
    expect(out).to.deep.equal([]);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_ENV=test npx mocha --exit test/qdrant-service.test.js`
Expected: FAIL — `Cannot find module '../ml/qdrant-service'`.

- [ ] **Step 3: Write the implementation**

Create `ml/qdrant-service.js`:
```javascript
'use strict';

const { QdrantClient } = require('@qdrant/js-client-rest');
const config = require('../config');
const embeddingClient = require('./embedding-client');
const { cardPointId } = require('./card-id');
const logger = require('../utils/logger').child('QdrantService');

class QdrantService {
  constructor(opts = {}) {
    this.enabled = opts.enabled !== undefined ? opts.enabled : config.VECTOR_SEARCH_ENABLED;
    this.collection = opts.collection || config.QDRANT_COLLECTION;
    this.dim = opts.dim || config.EMBED_DIM;
    this.embeddingClient = opts.embeddingClient || embeddingClient;
    this.client = null;
    if (this.enabled) {
      this.client = new QdrantClient({
        url: config.QDRANT_URL,
        apiKey: config.QDRANT_API_KEY || undefined
      });
    }
  }

  // Create the collection + userId payload index if absent. Fail-open.
  async ensureCollection() {
    if (!this.enabled) return;
    try {
      const existing = await this.client.getCollections();
      const names = (existing.collections || []).map(c => c.name);
      if (!names.includes(this.collection)) {
        await this.client.createCollection(this.collection, {
          vectors: { size: this.dim, distance: 'Cosine' }
        });
        await this.client.createPayloadIndex(this.collection, {
          field_name: 'userId',
          field_schema: 'keyword'
        });
        logger.info('Created Qdrant collection', { collection: this.collection });
      }
    } catch (err) {
      logger.error('ensureCollection failed (continuing)', { error: err.message });
    }
  }

  async upsertCard(card, userId) {
    if (!this.enabled) return;
    const text = `${card.question} ${card.answer}`;
    const vector = await this.embeddingClient.embed(text);
    await this.client.upsert(this.collection, {
      points: [{
        id: cardPointId(card._id),
        vector,
        payload: {
          userId: String(userId),
          cardId: String(card._id),
          question: card.question,
          answer: card.answer
        }
      }]
    });
  }

  async deleteCard(cardId) {
    if (!this.enabled) return;
    await this.client.delete(this.collection, { points: [cardPointId(cardId)] });
  }

  // Index a whole user's deck. Embeddings are cached by text within the run
  // (the seed deck repeats the same strings across users). Fail-open per card.
  async indexUserDeck(user) {
    if (!this.enabled) return;
    const cache = new Map();
    const points = [];
    for (const card of user.questions) {
      const text = `${card.question} ${card.answer}`;
      let vector = cache.get(text);
      if (!vector) {
        vector = await this.embeddingClient.embed(text);
        cache.set(text, vector);
      }
      points.push({
        id: cardPointId(card._id),
        vector,
        payload: {
          userId: String(user._id),
          cardId: String(card._id),
          question: card.question,
          answer: card.answer
        }
      });
    }
    if (points.length) {
      await this.client.upsert(this.collection, { points });
    }
    return points.length;
  }

  // Recommend by the card's stored vector, scoped to the user. `recommend`
  // excludes the example point from results. Apply the score floor + cap.
  async related(cardId, userId, k, minScore) {
    if (!this.enabled) return [];
    const res = await this.client.recommend(this.collection, {
      positive: [cardPointId(cardId)],
      filter: { must: [{ key: 'userId', match: { value: String(userId) } }] },
      limit: k,
      with_payload: true
    });
    return (res || [])
      .filter(p => p.score >= minScore)
      .slice(0, k)
      .map(p => ({
        cardId: p.payload.cardId,
        question: p.payload.question,
        answer: p.payload.answer,
        score: p.score
      }));
  }
}

const qdrantService = new QdrantService();
module.exports = qdrantService;
module.exports.QdrantService = QdrantService;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_ENV=test npx mocha --exit test/qdrant-service.test.js`
Expected: PASS (4 passing).

- [ ] **Step 5: Commit**

```bash
git add ml/qdrant-service.js test/qdrant-service.test.js
git commit -m "feat: qdrant-service (upsert/delete/indexUserDeck/related)"
```

---

## Task 6: `GET /api/questions/:id/related` endpoint

**Files:**
- Modify: `routes/questions.js` (imports at top; new route before `module.exports`)
- Test: `test/related.test.js`

- [ ] **Step 1: Write the failing integration test**

Create `test/related.test.js`:
```javascript
'use strict';
const chai = require('chai');
const chaiHttp = require('chai-http');
const mongoose = require('mongoose');

const { app } = require('../index');
const User = require('../models/user');
const { generateAccessToken } = require('../lib/auth/jwt');
const qdrantService = require('../ml/qdrant-service');
const { isDatabaseConnected, TEST_TIMEOUT } = require('./setup.test');

const expect = chai.expect;
chai.use(chaiHttp);

describe('GET /api/questions/:id/related', function() {
  this.timeout(TEST_TIMEOUT);

  let userId, token, cardId;
  const savedEnabled = qdrantService.enabled;
  let savedRelated;

  before(async function() {
    if (!isDatabaseConnected()) this.skip();
    const password = await User.hashPassword('testpassword123');
    const user = await User.create({
      firstName: 'Rel', lastName: 'Tester',
      username: `reltest_${Date.now()}`,
      password, head: 0,
      questions: [
        { _id: new mongoose.Types.ObjectId(), question: 'casa', answer: 'house', next: 1 },
        { _id: new mongoose.Types.ObjectId(), question: 'perro', answer: 'dog', next: 0 }
      ]
    });
    userId = user.id;
    cardId = user.questions[0]._id.toString();
    // Mint a real access token (correct issuer/audience + user.id claim) and
    // send it as the httpOnly cookie the middleware actually reads.
    token = generateAccessToken(user);
  });

  after(async function() {
    qdrantService.enabled = savedEnabled;
    if (savedRelated) qdrantService.related = savedRelated;
    if (userId) await User.findByIdAndDelete(userId);
  });

  it('401 without auth', function() {
    return chai.request(app).get(`/api/questions/${cardId}/related`)
      .then(res => expect(res).to.have.status(401));
  });

  it('returns [] when the feature is disabled', function() {
    qdrantService.enabled = false;
    return chai.request(app)
      .get(`/api/questions/${cardId}/related`)
      .set('Cookie', `accessToken=${token}`)
      .then(res => {
        expect(res).to.have.status(200);
        expect(res.body.related).to.deep.equal([]);
      });
  });

  it('404 for a card not in the user deck', function() {
    qdrantService.enabled = true;
    const ghost = new mongoose.Types.ObjectId().toString();
    return chai.request(app)
      .get(`/api/questions/${ghost}/related`)
      .set('Cookie', `accessToken=${token}`)
      .then(res => expect(res).to.have.status(404));
  });

  it('returns related results from the service (happy path)', function() {
    qdrantService.enabled = true;
    savedRelated = qdrantService.related;
    qdrantService.related = async () => ([
      { cardId: 'c2', question: 'perro', answer: 'dog', score: 0.88 }
    ]);
    return chai.request(app)
      .get(`/api/questions/${cardId}/related?k=3`)
      .set('Cookie', `accessToken=${token}`)
      .then(res => {
        expect(res).to.have.status(200);
        expect(res.body.related).to.have.lengthOf(1);
        expect(res.body.related[0].answer).to.equal('dog');
      });
  });

  it('fails soft (200 []) when the service throws', function() {
    qdrantService.enabled = true;
    qdrantService.related = async () => { throw new Error('qdrant down'); };
    return chai.request(app)
      .get(`/api/questions/${cardId}/related`)
      .set('Cookie', `accessToken=${token}`)
      .then(res => {
        expect(res).to.have.status(200);
        expect(res.body.related).to.deep.equal([]);
      });
  });
});
```
> Auth note: `requireAuth` (`middleware/cookie-auth.js`) reads the token from the
> `accessToken` **cookie**, not the `Authorization` header, and `verifyAccessToken`
> requires `issuer: 'intervalai-api'` + `audience: 'intervalai-client'`. That is
> why these tests use `generateAccessToken(user)` + `.set('Cookie', ...)`. Do NOT
> copy the Bearer-header pattern from the older `test/questions.test.js` — it only
> "passes" because it is skipped without a test DB.

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_ENV=test npx mocha --exit --file test/setup.test.js test/related.test.js`
Expected: FAIL — `/related` returns 404 (Express has no such route) where the happy-path/`[]` tests expect 200.

- [ ] **Step 3: Add imports to `routes/questions.js`**

At the top of `routes/questions.js`, after the existing `const { requireAuth } = require('../middleware/cookie-auth');` (line 11), add:
```javascript
const mongoose = require('mongoose');
const config = require('../config');
const { BadRequestError } = require('../utils/errors');
const qdrantService = require('../ml/qdrant-service');
```

- [ ] **Step 4: Add the route**

In `routes/questions.js`, immediately before `module.exports = router;` (line 257), add:
```javascript
/* ========== GET RELATED CARDS ========== */
router.get('/:id/related', requireAuth, async (req, res, next) => {
  try {
    const userId = req.user.id;
    const cardId = req.params.id;

    if (!mongoose.Types.ObjectId.isValid(cardId)) {
      throw new BadRequestError('The card `id` is not valid');
    }

    const user = await User.findById(userId);
    if (!user) {
      throw new NotFoundError('User not found');
    }

    const card = user.questions.id(cardId);
    if (!card) {
      throw new NotFoundError('Card not found');
    }

    if (!qdrantService.enabled) {
      return res.json({ related: [] });
    }

    const requestedK = parseInt(req.query.k, 10);
    const k = Math.min(
      Number.isInteger(requestedK) && requestedK > 0 ? requestedK : config.RELATED_K_DEFAULT,
      config.RELATED_K_MAX
    );

    let related = [];
    try {
      related = await qdrantService.related(cardId, userId, k, config.RELATED_MIN_SCORE);
    } catch (err) {
      logger.warn('Related lookup failed (returning empty)', { cardId, error: err.message });
      related = [];
    }

    res.json({ related });
  } catch (err) {
    next(err);
  }
});
```

- [ ] **Step 5: Run test to verify it passes**

Run: `NODE_ENV=test npx mocha --exit --file test/setup.test.js test/related.test.js`
Expected: PASS (5 passing), or skipped if no test DB.

- [ ] **Step 6: Commit**

```bash
git add routes/questions.js test/related.test.js
git commit -m "feat: GET /api/questions/:id/related endpoint (fail-soft)"
```

---

## Task 7: Index seeded deck at registration (fail-open)

**Files:**
- Modify: `routes/users.js` (import + after `User.create`)
- Test: extend `test/related.test.js` with a registration block (or create `test/registration-index.test.js`)

- [ ] **Step 1: Write the failing test**

Create `test/registration-index.test.js`:
```javascript
'use strict';
const chai = require('chai');
const chaiHttp = require('chai-http');

const { app } = require('../index');
const User = require('../models/user');
const qdrantService = require('../ml/qdrant-service');
const { isDatabaseConnected, TEST_TIMEOUT } = require('./setup.test');

const expect = chai.expect;
chai.use(chaiHttp);

describe('Registration indexes deck (fail-open)', function() {
  this.timeout(TEST_TIMEOUT);
  const savedEnabled = qdrantService.enabled;
  const savedIndex = qdrantService.indexUserDeck;
  let createdUsername;

  after(async function() {
    qdrantService.enabled = savedEnabled;
    qdrantService.indexUserDeck = savedIndex;
    if (createdUsername) await User.deleteOne({ username: createdUsername });
  });

  it('still returns 201 when indexing throws', function() {
    if (!isDatabaseConnected()) this.skip();
    qdrantService.enabled = true;
    qdrantService.indexUserDeck = async () => { throw new Error('qdrant down'); };
    createdUsername = `regidx_${Date.now()}`;
    return chai.request(app).post('/api/users').send({
      firstName: 'Reg', lastName: 'Idx',
      username: createdUsername, password: 'testpassword123'
    }).then(res => {
      expect(res).to.have.status(201);
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `NODE_ENV=test npx mocha --exit --file test/setup.test.js test/registration-index.test.js`
Expected: FAIL — without the fail-open guard, a throwing `indexUserDeck` either has no effect (route never calls it → test is meaningless) or, once wired naively with `await`, surfaces a 500. This test locks in that registration must not await/propagate indexing errors.

- [ ] **Step 3: Wire the hook**

In `routes/users.js`, after the existing imports (after line 10), add:
```javascript
const qdrantService = require('../ml/qdrant-service');
```
Then in the `POST /` handler, change the block around line 38 from:
```javascript
    const result = await User.create(newUser);
    logger.info('User created', { userId: result.id, username: result.username });
    res.status(201).location(`/api/users/${result.id}`).json(result);
```
to:
```javascript
    const result = await User.create(newUser);
    logger.info('User created', { userId: result.id, username: result.username });

    // Fire-and-forget: index the seeded deck into Qdrant. Never block or fail
    // registration on the vector index (MongoDB is the system-of-record).
    if (qdrantService.enabled) {
      qdrantService.indexUserDeck(result).catch(err =>
        logger.warn('Card indexing failed at registration', {
          userId: result.id, error: err.message
        })
      );
    }

    res.status(201).location(`/api/users/${result.id}`).json(result);
```

- [ ] **Step 4: Run test to verify it passes**

Run: `NODE_ENV=test npx mocha --exit --file test/setup.test.js test/registration-index.test.js`
Expected: PASS (201 returned despite the throwing stub).

- [ ] **Step 5: Commit**

```bash
git add routes/users.js test/registration-index.test.js
git commit -m "feat: index seeded deck into Qdrant at registration (fail-open)"
```

---

## Task 8: Ensure collection on startup

**Files:**
- Modify: `index.js` (near the existing `mlService.initialize()` call, ~line 312)

- [ ] **Step 1: Add the startup hook**

In `index.js`, find the block (~line 312):
```javascript
  mlService.initialize().catch(err => {
    logger.error('Failed to initialize ML service', { error: err.message });
```
Immediately after that `mlService.initialize().catch(...)` statement (after its closing `});`), add:
```javascript
  // Ensure the Qdrant collection exists (no-op when VECTOR_SEARCH_ENABLED=false).
  const qdrantService = require('./ml/qdrant-service');
  qdrantService.ensureCollection().catch(err => {
    logger.error('Failed to ensure Qdrant collection', { error: err.message });
  });
```

- [ ] **Step 2: Verify the server still boots**

Run:
```bash
NODE_ENV=test node -e "require('./index'); setTimeout(() => { console.log('boot ok'); process.exit(0); }, 1500)"
```
Expected: prints `boot ok` and exits 0. With `VECTOR_SEARCH_ENABLED` unset, `ensureCollection` returns immediately; no Qdrant connection is attempted.

- [ ] **Step 3: Run the full suite to confirm no regressions**

Run: `npm run test:once`
Expected: existing suites pass; new suites pass (or skip without a test DB). No suite errors.

- [ ] **Step 4: Commit**

```bash
git add index.js
git commit -m "feat: ensure Qdrant collection on startup (fail-open)"
```

---

## Task 9: Backfill script for existing users

**Files:**
- Create: `scripts/backfill-embeddings.js`

- [ ] **Step 1: Write the script**

Create `scripts/backfill-embeddings.js`:
```javascript
'use strict';

/**
 * One-off backfill: embed every existing user's cards into Qdrant.
 * Usage:
 *   VECTOR_SEARCH_ENABLED=true MONGODB_URI=... QDRANT_URL=... \
 *   OVMS_EMBED_URL=... node scripts/backfill-embeddings.js
 */

const mongoose = require('mongoose');
const config = require('./../config');
const { dbConnect, dbDisconnect } = require('../db-mongoose');
const User = require('../models/user');
const qdrantService = require('../ml/qdrant-service');
const logger = require('../utils/logger').child('Backfill');

async function backfillAll() {
  if (!qdrantService.enabled) {
    logger.error('VECTOR_SEARCH_ENABLED is not true; refusing to run.');
    return { users: 0, points: 0 };
  }
  await qdrantService.ensureCollection();

  let users = 0;
  let points = 0;
  const cursor = User.find({}, 'questions').cursor();
  for (let user = await cursor.next(); user != null; user = await cursor.next()) {
    try {
      const n = await qdrantService.indexUserDeck(user);
      users += 1;
      points += n || 0;
      logger.info('Indexed user', { userId: user.id, cards: n });
    } catch (err) {
      logger.warn('Failed to index user (continuing)', { userId: user.id, error: err.message });
    }
  }
  logger.info('Backfill complete', { users, points });
  return { users, points };
}

// Run directly (not when require()'d by a test).
if (require.main === module) {
  (async () => {
    await dbConnect(config.MONGODB_URI);
    try {
      await backfillAll();
    } finally {
      await dbDisconnect();
    }
  })().catch(err => {
    logger.error('Backfill failed', { error: err.message });
    process.exit(1);
  });
}

module.exports = { backfillAll };
```

- [ ] **Step 2: Smoke-check it loads and guards correctly**

Run:
```bash
node -e "const { backfillAll } = require('./scripts/backfill-embeddings'); backfillAll().then(r => console.log('guard ok', r))"
```
Expected: with `VECTOR_SEARCH_ENABLED` unset, logs the refusal and prints `guard ok { users: 0, points: 0 }` without touching Mongo/Qdrant.

- [ ] **Step 3: Commit**

```bash
git add scripts/backfill-embeddings.js
git commit -m "feat: backfill script to index existing users' cards"
```

---

## Task 10: OVMS embedding model + K8s manifests (Intel iGPU)

This is infrastructure, not Node code — no unit tests. Produce the artifacts and verify the model answers KServe v2 inference on the Intel iGPU. These files may ultimately live in the portfolio GitOps repo; staged here under `k8s/ovms/` for review.

- [ ] **Step 1: Export the model to OpenVINO IR** (run on a workstation with Python)

```bash
pip install "optimum[openvino]"
optimum-cli export openvino \
  --model sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2 \
  --task feature-extraction \
  text_embed_ir
# Produces text_embed_ir/openvino_model.{xml,bin} + tokenizer files.
# Confirm the model output tensor name (expected: last_hidden_state):
python -c "import openvino as ov; m=ov.Core().read_model('text_embed_ir/openvino_model.xml'); print([o.get_any_name() for o in m.outputs])"
```
Expected: prints output names including `last_hidden_state`. If it differs, set `EMBED_OUTPUT_NAME` accordingly (Task 2 config).

- [ ] **Step 2: Lay out the OVMS model repository**

```
ovms-models/
  text_embed/
    1/
      openvino_model.xml   # renamed/copy of exported IR
      openvino_model.bin
```
OVMS auto-discovers `text_embed` version `1`. Package this directory into the OVMS image or a PVC mounted at `/models`.

- [ ] **Step 3: Create the OVMS Deployment + Service**

Create `k8s/ovms/ovms-embed.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ovms-embed
  labels: { app: ovms-embed }
spec:
  replicas: 1
  selector: { matchLabels: { app: ovms-embed } }
  template:
    metadata: { labels: { app: ovms-embed } }
    spec:
      # Pin to the node that has the Intel iGPU.
      nodeSelector:
        kubernetes.io/hostname: debian-marmoset
      containers:
        - name: ovms
          image: openvino/model_server:latest-gpu
          args:
            - "--model_name=text_embed"
            - "--model_path=/models/text_embed"
            - "--target_device=GPU"
            - "--port=9000"        # gRPC
            - "--rest_port=8000"   # KServe v2 REST
          ports:
            - { containerPort: 8000, name: rest }
            - { containerPort: 9000, name: grpc }
          resources:
            limits:
              gpu.intel.com/i915: "1"
          volumeMounts:
            - { name: models, mountPath: /models }
      volumes:
        - name: models
          # Replace with the real source (PVC or image-baked path).
          persistentVolumeClaim:
            claimName: ovms-embed-models
---
apiVersion: v1
kind: Service
metadata:
  name: ovms-embed
spec:
  selector: { app: ovms-embed }
  ports:
    - { name: rest, port: 8000, targetPort: 8000 }
    - { name: grpc, port: 9000, targetPort: 9000 }
```
> Confirm the cluster's Intel GPU device-plugin resource name is `gpu.intel.com/i915` (it backs Jellyfin on debian-marmoset). Adjust the namespace to match the app's namespace so `OVMS_EMBED_URL=http://ovms-embed:8000` resolves.

- [ ] **Step 4: Deploy and verify inference on the iGPU**

```bash
kubectl apply -f k8s/ovms/ovms-embed.yaml
kubectl rollout status deploy/ovms-embed
# Readiness (KServe v2):
kubectl exec deploy/ovms-embed -- curl -s localhost:8000/v2/models/text_embed/ready -o /dev/null -w "%{http_code}\n"
```
Expected: rollout succeeds; readiness returns `200`. Logs should show the GPU plugin loading the model (`Available devices ... GPU`).

- [ ] **Step 5: Commit**

```bash
git add k8s/ovms/ovms-embed.yaml
git commit -m "infra: OVMS text_embed model on Intel iGPU (debian-marmoset)"
```

---

## Task 11: Qdrant collection + secrets + rollout

- [ ] **Step 1: Confirm the Qdrant in-cluster endpoint**

```bash
kubectl get svc -A | grep -i qdrant
```
Expected: a Service (e.g. `qdrant:6333`). Set `QDRANT_URL` (Task 2 default `http://qdrant:6333`) to match its name/namespace. If the pod requires an API key, capture it for `QDRANT_API_KEY`.

- [ ] **Step 2: Add secrets/env via Doppler + ESO**

Following the app's existing secret pattern (Doppler → External Secrets Operator → K8s Secret → Deployment env), add:
`VECTOR_SEARCH_ENABLED`, `QDRANT_URL`, `QDRANT_API_KEY`, `OVMS_EMBED_URL`. Wire them into `k8s/server-deployment.yaml`'s env (mirror how `MONGODB_URI`/`JWT_SECRET` are injected). Keep `VECTOR_SEARCH_ENABLED=false` for the first deploy (dark launch).

- [ ] **Step 3: Deploy dark, verify boot**

Deploy with the flag off. Confirm the server boots and `ensureCollection` is a no-op (no Qdrant traffic). Smoke-test `/related` returns `200 { "related": [] }`.

- [ ] **Step 4: Enable + backfill + smoke test**

Flip `VECTOR_SEARCH_ENABLED=true`. Confirm the collection is created (Task 8 startup hook). Run the backfill:
```bash
kubectl exec deploy/<server> -- node scripts/backfill-embeddings.js
```
Then call `/api/questions/:id/related` for a real card and confirm non-empty, user-scoped results. **Inspect the real scores** and tune `RELATED_MIN_SCORE` (the 0.5 default is a guess — set it from observed neighbor scores).

- [ ] **Step 5: Commit deployment changes**

```bash
git add k8s/server-deployment.yaml
git commit -m "infra: wire vector-search env into server deployment (dark launch)"
```

---

## Out of Scope (tracked separately)
- Client UI for the related-cards panel (client submodule).
- Influencing study-queue sequencing with relatedness (spec phase 2).
- Add/edit/delete-card endpoints (would re-activate the `upsertCard`/`deleteCard` hooks already built into `qdrant-service`).
- Repointing the existing `interval_ai` model off marmoset's NVIDIA GPU.
