# Vibe-code a Secure Chats server

Two long, self-contained prompts that instruct a frontier LLM to produce a
complete, runnable HTTP server implementing the
[Secure Chats JSON API](https://secchats.com/api.html) — one in Python
(FastAPI), one in Node.js (Fastify). Paste one into your favourite model
and follow the definition-of-done checklist to verify the output.

Companion piece to the blog post
[*One-Shot Server from a JSON Spec — Vibe Coding's Next Frontier*](https://secchats.com/blog_entry_5.html).

## Why two prompts?

The prompts encode the same contract with the same wire-format
invariants, schema, and zero-trust guarantees. Only the stack differs.
Which one you pick is a taste question. The Python variant reproduces
the run described in the blog post; the Node.js variant is what you'd
reach for if your ops story is npm/Node.

Both prompts are designed to be **one-shot**: paste, run, done. If the
model produces something that fails the "Definition of done" section at
the bottom of each prompt, feed the failing test output back and it
will usually converge in one or two follow-ups.

## Prerequisites

- Any frontier-class LLM. This has been tested with **Claude Opus/Sonnet
  4.6+** and **Gemini 2.5 Pro**; GPT-5 and comparable models should work
  the same way. The prompt is long — pick a model with a large context
  window and strong instruction-following.
- **Python 3.11+** *(for prompt A)* or **Node.js 20+** *(for prompt B)*.
- Ability to run `curl` locally to verify the wire round-trip.

## How to use

1. Copy one of the two prompt blocks below **verbatim**, top to bottom.
   Don't summarise, don't reflow — the definition-of-done rubric at the
   end is doing more work than any single paragraph.
2. Paste it into your LLM as a single message. If your model's UI splits
   long messages, use API access or a client that accepts a single-turn
   prompt.
3. When the model responds with a file tree, save each file at the
   indicated path under a new empty directory.
4. Run the setup + test commands the prompt asks for. All tests should
   pass on the first try. If not, paste the failing pytest / `node --test`
   output back into the same conversation — do **not** start a fresh chat,
   the model's context of the spec matters.
5. Sanity-check against a real client (the Secure Chats iOS or Android app,
   or `curl` scripts of your own) if you plan to deploy.

## Prompt A — Python (FastAPI + Pydantic + sqlite3 + uvicorn)

Save the following into a file and paste as a single message, or paste
inline. **~270 lines, one self-contained block:**

```
You are an expert Python backend engineer. Build a complete, runnable
HTTP API server implementing the Secure Chats JSON API specified at
https://secchats.com/api.html.

The spec is the contract. Read it. The summary in this prompt is a
fallback when you don't have web access — implement the spec's shapes
verbatim, not a paraphrase.

Stack (default — swap only with a written reason)

FastAPI + Pydantic + stdlib sqlite3 + uvicorn. No ORM. Async endpoints
throughout because /longpoll requires a real wait-notify primitive, not
a polling loop.

Endpoints (all POST, all Content-Type: application/json)

  /register   — Create a user → return userId + authToken.   No
  /register2  — Create a group → return groupId.              Yes
  /receive    — Notifications since lastMessageTimestamp.       Yes
  /connect    — Friend request (DH public key) to target user.  Yes
  /connect2   — Reply to a friend request.                      Yes
  /postmsg    — Send an encrypted direct message.               Yes
  /groupinv   — Deliver an encrypted group key to a user.       Yes
  /postgroup  — Send an encrypted group message.                Yes
  /pollgroup  — Group messages since timestamp.                 Yes
  /pollgroups — Multi-group variant of /pollgroup.              Yes
  /longpoll   — Block up to 25s waiting for any of N groups to
                  receive a message for the caller. Plain-text
                  response.                                       Yes

Canonical body shapes (every always-emitted field, in declaration order)

The reference Java serializer (ChatSerializer) emits every declared
field of every model by reflection, in declaration order, with a
leading "object" discriminator. Mirror that exactly.

  // /register response
  { "object":"User", "userId":"<hex>", "name":"",
    "authToken":"<token>", "lastMessageTimestamp":0 }

  // /register2 response (newly created Group)
  { "object":"Group", "groupId":"<id>", "name":null,
    "algorithm":null, "privateKey":null,
    "lastMessageTimestamp":0 }

  // /receive request body
  { "object":"User", "userId":"<id>", "name":"",
    "authToken":"<token>",
    "lastMessageTimestamp":1712345678000 }

  // /receive response (Notifications)
  { "object":"Notifications",
    "friendRequests":[ { "object":"FriendRequest",
        "fromUserId":"<id>", "authToken":null,
        "toUserId":"<id>", "timestamp":1712345678000,
        "protocol":"DH_AND_AES", "cryptoData":"<base64>" } ],
    "friendResponses":[ ... ],
    "groupInvites":[ ... ],
    "messages":[ { "object":"SendMessage", "toId":"<id>",
        "authToken":null, "fromUserId":"<id>",
        "encContent":"<base64>",
        "timestamp":1712345678000 } ] }

  // /connect request
  { "object":"FriendRequest", "fromUserId":"<id>",
    "authToken":"<token>", "toUserId":"<target>",
    "timestamp":0, "protocol":"DH_AND_AES",
    "cryptoData":"<base64-dh-public-key>" }

  // /postmsg request (toId = recipient userId)
  // /postgroup request (toId = groupId — same shape)
  { "object":"SendMessage", "toId":"<target>",
    "authToken":"<token>", "fromUserId":"<id>",
    "encContent":"<base64>", "timestamp":0 }

  // /longpoll response: plain text — group id that fired or "".

Wire-format notes (the spec page elides some always-present fields)

- Every model emits every declared field in declaration order —
  null and 0 included. Mirror that exactly on responses.
- Strip authToken (set to null) on anything echoed back from the
  notification queue, but keep the field in the JSON.
- Clients sometimes omit timestamp:0 and lastMessageTimestamp:0 on
  send. Accept gracefully on parse, emit always on serialise. With
  Pydantic: model_config = ConfigDict(extra='ignore') plus
  Field(default=0) on every long.
- name is "" for users and null (or "" — be tolerant) for newly
  created groups. The server never persists or inspects it.

Storage (zero-trust — server stores the absolute minimum)

CREATE TABLE principals (
  id              TEXT PRIMARY KEY,
  kind            TEXT NOT NULL CHECK (kind IN ('user','group')),
  auth_token_hash BLOB,
  created_at      INTEGER NOT NULL
);

CREATE TABLE notifications (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  to_principal   TEXT NOT NULL REFERENCES principals(id),
  kind           TEXT NOT NULL CHECK (kind IN
                   ('FriendRequest','FriendResponse',
                    'GroupInvite','SendMessage')),
  from_user      TEXT,
  group_id       TEXT,
  protocol       TEXT,
  algorithm      TEXT,
  crypto_data    BLOB,
  enc_content    BLOB,
  ts_ms          INTEGER NOT NULL
);
CREATE INDEX idx_notif_to_ts
  ON notifications(to_principal, ts_ms);

Hard invariants — fail the build mentally if any are violated:

- authToken is never persisted. Store HMAC_SHA256(SECCHATS_HMAC_SECRET,
  token) and constant-time compare via hmac.compare_digest.
- name field is sent and stored as "". Never index, log, or echo
  display names. They live only inside encContent.
- cryptoData / encContent are opaque blobs — store and return
  untouched.
- Per-recipient queues: friend-request / friend-response /
  group-invite / direct-message rows keyed on the target user. Group
  messages keyed on the group.
- No plain auth tokens or message content in logs.

Behavioural invariants

- All non-/register endpoints validate userId + authToken against
  principals.auth_token_hash. Mismatch ⇒ HTTP 429 plain-text body
  "Rate limited." (per the spec's error table).
- /connect and /groupinv ⇒ 404 if target user / group doesn't exist.
- Polling endpoints return only rows with ts_ms > lastMessageTimestamp.
- When emitting any queued item, set authToken to null before
  serialise (mirrors the reference server's clearAuthToken()).
- /longpoll implementation: a dict[str, list[asyncio.Event]] keyed by
  user id. On any insert into notifications for a user, set + drain
  that user's events. Endpoint awaits with
  asyncio.wait_for(..., timeout=25). Return "" on TimeoutError. No
  while True: sleep.
- Per-IP token-bucket rate limit on every endpoint: 30 requests / 5
  seconds. Excess ⇒ 429 plain text.
- Timestamps are int(time.time() * 1000) everywhere.

Deliverables (single project directory)

  server/
    app.py            # FastAPI app, all endpoints, Pydantic models
    store.py          # SQLite helpers + async event registry
    schema.sql        # the DDL above
    test_api.py       # pytest, one happy-path per endpoint
    requirements.txt
    README.md         # venv setup, run, test

Must run with:
  pip install -r requirements.txt
  SECCHATS_HMAC_SECRET=test uvicorn app:app --port 8080
  pytest -q

What to avoid

- ORMs.
- Endpoints not in the spec.
- Polling loops for /longpoll.
- Storing or logging cleartext tokens, display names, or message
  bodies.

Definition of done

- pytest -q is green.
- Round-trip /register x 2 → A /connects B → B /receives the friend
  request with the exact wire shape.
- A second user /longpolling on a shared group is woken within ms
  when the first user /postgroups to it.
- sqlite3 db.sqlite '.dump' shows no plaintext tokens, no display
  names, and no plaintext message content.

Build it. Output the full file tree with file contents inline.
```

## Prompt B — Node.js (Fastify + better-sqlite3 + zod)

Same contract, same zero-trust guarantees, JavaScript stack.

```
You are an expert Node.js backend engineer. Build a complete, runnable
HTTP API server implementing the Secure Chats JSON API specified at
https://secchats.com/api.html.

The spec is the contract. Read it. The summary in this prompt is a
fallback when you don't have web access — implement the spec's shapes
verbatim, not a paraphrase.

Stack (default — swap only with a written reason)

Fastify + better-sqlite3 + zod for request validation. Plain JavaScript,
ES modules ("type": "module" in package.json), Node.js >= 20 (needed
for AbortSignal.timeout and the built-in node:test runner). No ORM.
/longpoll uses events.once() with an AbortSignal, not a polling loop.

Endpoints (all POST, all Content-Type: application/json)

  /register   — Create a user → return userId + authToken.   No
  /register2  — Create a group → return groupId.              Yes
  /receive    — Notifications since lastMessageTimestamp.       Yes
  /connect    — Friend request (DH public key) to target user.  Yes
  /connect2   — Reply to a friend request.                      Yes
  /postmsg    — Send an encrypted direct message.               Yes
  /groupinv   — Deliver an encrypted group key to a user.       Yes
  /postgroup  — Send an encrypted group message.                Yes
  /pollgroup  — Group messages since timestamp.                 Yes
  /pollgroups — Multi-group variant of /pollgroup.              Yes
  /longpoll   — Block up to 25s waiting for any of N groups to
                  receive a message for the caller. Plain-text
                  response.                                       Yes

Canonical body shapes (every always-emitted field, in declaration order)

The reference Java serializer (ChatSerializer) emits every declared
field of every model by reflection, in declaration order, with a
leading "object" discriminator. Mirror that exactly. In Node this
means constructing plain objects with keys inserted in canonical
order — JSON.stringify preserves insertion order and Fastify will not
reorder unless you install a schema-based serializer, so avoid the
fast-json-stringify default (or list the keys in the schema in the
exact order below).

  // /register response
  { "object":"User", "userId":"<hex>", "name":"",
    "authToken":"<token>", "lastMessageTimestamp":0 }

  // /register2 response (newly created Group)
  { "object":"Group", "groupId":"<id>", "name":null,
    "algorithm":null, "privateKey":null,
    "lastMessageTimestamp":0 }

  // /receive request body
  { "object":"User", "userId":"<id>", "name":"",
    "authToken":"<token>",
    "lastMessageTimestamp":1712345678000 }

  // /receive response (Notifications)
  { "object":"Notifications",
    "friendRequests":[ { "object":"FriendRequest",
        "fromUserId":"<id>", "authToken":null,
        "toUserId":"<id>", "timestamp":1712345678000,
        "protocol":"DH_AND_AES", "cryptoData":"<base64>" } ],
    "friendResponses":[ ... ],
    "groupInvites":[ ... ],
    "messages":[ { "object":"SendMessage", "toId":"<id>",
        "authToken":null, "fromUserId":"<id>",
        "encContent":"<base64>",
        "timestamp":1712345678000 } ] }

  // /connect request
  { "object":"FriendRequest", "fromUserId":"<id>",
    "authToken":"<token>", "toUserId":"<target>",
    "timestamp":0, "protocol":"DH_AND_AES",
    "cryptoData":"<base64-dh-public-key>" }

  // /postmsg request (toId = recipient userId)
  // /postgroup request (toId = groupId — same shape)
  { "object":"SendMessage", "toId":"<target>",
    "authToken":"<token>", "fromUserId":"<id>",
    "encContent":"<base64>", "timestamp":0 }

  // /longpoll response: plain text — group id that fired or "".

Wire-format notes (the spec page elides some always-present fields)

- Every model emits every declared field in declaration order —
  null and 0 included. Mirror that exactly on responses.
- Strip authToken (set to null) on anything echoed back from the
  notification queue, but keep the field in the JSON.
- Clients sometimes omit timestamp:0 and lastMessageTimestamp:0 on
  send. Accept gracefully on parse, emit always on serialise. With
  zod: z.number().int().default(0) on every timestamp field, and
  .passthrough() at the top level so unknown keys don't reject.
- name is "" for users and null (or "" — be tolerant) for newly
  created groups. The server never persists or inspects it.
- Do not use fast-json-stringify's default serializer if it would
  reorder keys. Either register serializers with the schema key order
  matching the canonical order above, or bypass Fastify's serializer
  with reply.type('application/json').send(JSON.stringify(payload)).

Storage (zero-trust — server stores the absolute minimum)

CREATE TABLE principals (
  id              TEXT PRIMARY KEY,
  kind            TEXT NOT NULL CHECK (kind IN ('user','group')),
  auth_token_hash BLOB,
  created_at      INTEGER NOT NULL
);

CREATE TABLE notifications (
  id             INTEGER PRIMARY KEY AUTOINCREMENT,
  to_principal   TEXT NOT NULL REFERENCES principals(id),
  kind           TEXT NOT NULL CHECK (kind IN
                   ('FriendRequest','FriendResponse',
                    'GroupInvite','SendMessage')),
  from_user      TEXT,
  group_id       TEXT,
  protocol       TEXT,
  algorithm      TEXT,
  crypto_data    BLOB,
  enc_content    BLOB,
  ts_ms          INTEGER NOT NULL
);
CREATE INDEX idx_notif_to_ts
  ON notifications(to_principal, ts_ms);

Hard invariants — fail the build mentally if any are violated:

- authToken is never persisted. Store the raw bytes of
  crypto.createHmac('sha256', process.env.SECCHATS_HMAC_SECRET)
    .update(token).digest()
  and compare with crypto.timingSafeEqual — never a plain `===`.
- name field is sent and stored as "". Never index, log, or echo
  display names. They live only inside encContent.
- cryptoData / encContent are opaque blobs — store as BLOB, return
  untouched. Prefer base64 in JSON, raw bytes in SQLite.
- Per-recipient queues: friend-request / friend-response /
  group-invite / direct-message rows keyed on the target user. Group
  messages keyed on the group.
- No plain auth tokens or message content in logs. Configure Fastify
  logger with a redact list of ['req.body.authToken', 'authToken',
  'encContent', 'cryptoData'] and log at 'info' by default.

Behavioural invariants

- All non-/register endpoints validate userId + authToken against
  principals.auth_token_hash. Mismatch ⇒ HTTP 429 plain-text body
  "Rate limited." (per the spec's error table).
- /connect and /groupinv ⇒ 404 if target user / group doesn't exist.
- Polling endpoints return only rows with ts_ms > lastMessageTimestamp.
- When emitting any queued item, set authToken to null before
  serialise (mirrors the reference server's clearAuthToken()).
- /longpoll implementation: a Map<string, EventEmitter> keyed by user
  id (or one shared EventEmitter with per-user event names — either
  is fine). On any insert into notifications for a user, emit that
  user's event. Endpoint awaits with:
    await events.once(emitter, userId,
      { signal: AbortSignal.timeout(25000) });
  Catch AbortError and reply with "" (plain text, 200). Never use
  a while(true) sleep loop.
- Per-IP token-bucket rate limit on every endpoint: 30 requests / 5
  seconds. Excess ⇒ 429 plain text. Use @fastify/rate-limit or a
  small in-memory Map, whichever fits your file tree cleanly.
- Timestamps are Date.now() everywhere (int ms since epoch).

better-sqlite3 usage notes

- better-sqlite3 is synchronous. That's fine for point reads/writes
  under Node's event loop; SQLite operations return in microseconds.
- Do NOT block the event loop from inside /longpoll while waiting for
  new messages — the DB call to insert a notification happens on the
  producer's request; the consumer just awaits an EventEmitter event.
- Wrap multi-statement writes in db.transaction(...) so a partial
  failure doesn't leave the queue inconsistent.
- Open the database once at process start:
    const db = new Database('db.sqlite');
    db.pragma('journal_mode = WAL');
    db.pragma('foreign_keys = ON');

Deliverables (single project directory)

  server/
    app.js            # Fastify app + all route handlers
    store.js          # better-sqlite3 helpers + EventEmitter registry
    schema.sql        # the DDL above (applied on first boot)
    test/api.test.js  # node --test, one happy-path per endpoint
                      # using fastify.inject() (no separate HTTP
                      # roundtrip needed)
    package.json      # ES modules, deps: fastify, better-sqlite3, zod
    README.md         # install, run, test

Must run with:
  npm install
  SECCHATS_HMAC_SECRET=test node app.js
  node --test

What to avoid

- ORMs.
- Endpoints not in the spec.
- Polling loops for /longpoll.
- Storing or logging cleartext tokens, display names, or message
  bodies.
- fast-json-stringify with a schema that reorders keys away from the
  canonical order above.
- express-style callback pyramids — Fastify handlers are async.

Definition of done

- `node --test` is green.
- Round-trip /register × 2 → A /connects B → B /receives the friend
  request with the exact wire shape and every always-emitted field
  present, in declaration order.
- A second user /longpolling on a shared group is woken within ms
  when the first user /postgroups to it (assert timing with
  performance.now() in the test).
- sqlite3 db.sqlite '.dump' shows no plaintext tokens, no display
  names, and no plaintext message content.

Build it. Output the full file tree with file contents inline.
```

## Verification checklist (both stacks)

Beyond what each prompt's own "Definition of done" enforces, once the
model emits code you can sanity-check:

- **Wire shape.** `curl -s -X POST http://localhost:8080/register -d '{}'`
  returns a JSON object with keys in this order: `object, userId, name,
  authToken, lastMessageTimestamp`. If your model reordered them, the
  reference clients will refuse to parse.
- **Token storage.** `sqlite3 db.sqlite 'SELECT auth_token_hash FROM
  principals LIMIT 1;'` returns raw bytes, not the token you saw at
  register time.
- **Longpoll wake-up.** Open two shells. In one, `curl -sv -X POST
  .../longpoll ...` (up to 25 s wait). In the other, `curl -X POST
  .../postgroup ...` for a group the first shell is subscribed to.
  The first shell should return within a few ms of the second, not
  after the full 25 s.

## FAQ

**Why isn't there a native-Rust / Go / Elixir / etc. version?**
The blog post's point was that the *prompt* is the artifact worth
maintaining, not the specific server. Both stacks below are meant to
be reference points; adapting either to a language of your choice is
a small edit — change the Stack, Storage-usage-notes, and Deliverables
sections, and the rest of the prompt survives untouched.

**Which prompt produces a better server?**
In our (small) sample of tests, both produce a working server on the
first try when handed to a top-tier model. The differences show up in
what fails on the second run:
- Python variant fails most often on Pydantic key-order emission —
  fixed by pinning `model_dump(mode='json', by_alias=False)` and
  declaring fields in canonical order.
- Node.js variant fails most often on Fastify's `fast-json-stringify`
  optimisation reordering keys — fixed by the note in prompt B about
  bypassing Fastify's serializer or matching schema key order.
The prompts already contain those mitigations; the failure modes are
listed here so you know what to look for if a model deviates.

**Can I use this to run a public Secure Chats server?**
Yes, but the prompts produce a **minimum-viable** server. Before
exposing it: put it behind TLS, cap upload sizes, add persistent rate-
limit storage (the in-memory bucket resets on restart), and audit the
logging redact list against your actual logging config.

## Licence

Prompts are released under [CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) —
copy, fork, edit without attribution.
