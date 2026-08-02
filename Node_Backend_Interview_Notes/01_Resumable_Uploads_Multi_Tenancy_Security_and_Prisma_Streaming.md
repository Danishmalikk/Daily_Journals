# Node/Backend Notes — Resumable Large Uploads, Multi-Tenant Security (BOLA), Prisma Streaming at Scale

> Beginner-friendly + interview-ready. Read top to bottom.

---

## 1. Uploading a 50 GB file when the internet disconnects midway

*Definition:* You solve this with a **resumable / chunked upload**: the file is split into small pieces on the client, each piece is uploaded (and tracked) independently, and if the connection drops, the client only re-sends the pieces that never arrived — it never restarts the whole 50 GB from byte zero.

Real life: Think of moving house by carrying boxes one at a time instead of trying to carry the entire house in one trip. If you get interrupted, you just check which boxes are already in the new house and carry the rest — you don't unpack and start over.

> If your upload logic re-sends the entire file after any network blip, that's the bug. The interviewer isn't testing "do you know HTTP file upload" — they're testing "do you know how to make a long-running transfer *resumable* and *idempotent*."

### The architecture

```
Client                                   Server / Object Storage
------                                   -----------------------
1. POST /uploads/init  (filename, size) ---->  create uploadId, session record
                                          <---- uploadId, chunkSize

2. Split file into N chunks (e.g. 8 MB each)

3. for each chunk:
     PUT /uploads/{uploadId}/chunk/{index}  (chunk bytes + checksum)
                                          ---->  store chunk, mark index "received"
                                          <----  200 OK

   [ network drops here! Client only lost the CURRENT chunk, not the file ]

4. Client reconnects:
     GET /uploads/{uploadId}/status
                                          <----  { receivedIndexes: [0,1,2,...,4021] }

5. Client resumes from the first MISSING index — not from 0

6. POST /uploads/{uploadId}/complete
                                          ---->  merge chunks (or S3 CompleteMultipartUpload)
                                          <----  final file URL + checksum
```

### Minimal client-side chunking logic

```js
// ❌ Bad: uploads the whole 50GB file as one request body
async function badUpload(file) {
  await fetch('/upload', { method: 'POST', body: file }); // any drop = start over
}

// ✅ Good: chunked, resumable, checks server state first
const CHUNK_SIZE = 8 * 1024 * 1024; // 8MB

async function resumableUpload(file, uploadId) {
  const totalChunks = Math.ceil(file.size / CHUNK_SIZE);

  // Ask the server what it already has (works even after a page refresh)
  const { receivedIndexes } = await fetch(`/uploads/${uploadId}/status`).then(r => r.json());
  const done = new Set(receivedIndexes);

  for (let i = 0; i < totalChunks; i++) {
    if (done.has(i)) continue; // skip chunks already on the server

    const chunk = file.slice(i * CHUNK_SIZE, (i + 1) * CHUNK_SIZE);
    await fetch(`/uploads/${uploadId}/chunk/${i}`, { method: 'PUT', body: chunk });
  }

  await fetch(`/uploads/${uploadId}/complete`, { method: 'POST' });
}
```

### Server side: track chunk state, don't trust one long stream

```js
// Chunk uploads are idempotent — re-uploading index 42 just overwrites index 42.
app.put('/uploads/:id/chunk/:index', async (req, res) => {
  const { id, index } = req.params;
  await storage.saveChunk(id, Number(index), req.body); // e.g. write to disk/S3 part
  await redis.sadd(`upload:${id}:received`, index);      // track progress
  res.sendStatus(200);
});

app.get('/uploads/:id/status', async (req, res) => {
  const receivedIndexes = await redis.smembers(`upload:${req.params.id}:received`);
  res.json({ receivedIndexes: receivedIndexes.map(Number) });
});
```

### Real-world building blocks (don't reinvent this)

| Piece | What to actually use |
|-------|----------------------|
| Protocol/spec | **tus protocol** (open resumable-upload standard) |
| Client library | `tus-js-client`, `Uppy`, `resumable.js` |
| Cloud storage | **AWS S3 Multipart Upload** (parts uploaded independently, `CompleteMultipartUpload` stitches them — S3 does the merge for you) |
| Google Cloud | **Resumable Upload API** (session URI + `Content-Range` header) |
| Progress tracking | Redis (fast, TTL-friendly) or a `uploads` DB table |

> **Content-Range is your friend:** both S3 and tus use a byte-range header so the client can ask "resume from byte X" instead of "resume from chunk index." Same idea, byte-granularity instead of chunk-granularity.

**Key Points:**
- Never buffer the whole 50 GB in server memory — stream each chunk straight to disk/object storage.
- Make chunk upload **idempotent** — re-sending the same chunk index must not corrupt or duplicate data.
- Use **presigned URLs** so large chunks go directly client → S3, not client → your API server → S3 (saves your server's bandwidth/CPU).
- Verify a **checksum per chunk** (and one final checksum for the whole file) to catch silent corruption.
- Run a cleanup job for abandoned upload sessions (TTL on the Redis/DB record) so orphaned chunks don't pile up in storage.

**Easy rule of thumb:** If a single request can silently fail and force a full restart, it isn't resumable — resumability means "resume from where it broke," not "retry the whole thing."

**Interview one-liner:** Split the file into chunks, upload each independently with a checksum, track received chunk indexes server-side, and on reconnect only resend the missing chunks — using S3 Multipart Upload / the tus protocol rather than hand-rolling it.

---

## 2. Why reading `tenantId` from `req.body` is a critical security risk

*Definition:* In a multi-tenant app, **`tenantId`** is the boundary that decides *whose data* a request can touch. If the server trusts a `tenantId` value the client typed into the request instead of deriving it from the authenticated session, any logged-in user can simply put a *different* tenant's ID in their request and read or write that tenant's data. This is **Broken Object Level Authorization (BOLA)** — OWASP API Security's **#1** most common API vulnerability.

Real life: A hotel key card that lets you type in *any* room number yourself instead of the card being pre-coded to your room. The front desk (your server) shouldn't ask "which room do you want?" — it should already know, because you checked in (authenticated) as the guest of room 214.

```
❌ Client decides the boundary:
   Attacker (tenant A, logged in) sends:
   POST /invoices  { "tenantId": "tenant-B", "amount": 999999 }
                              │
                              ▼
                    Server trusts req.body.tenantId
                              │
                              ▼
                 Row written into TENANT B's data  ← breach

✅ Server decides the boundary:
   Attacker sends the same payload, but tenantId is IGNORED
                              │
                              ▼
        Server pulls tenantId from the verified JWT/session
                              │
                              ▼
                 Row can only ever land in the ATTACKER'S OWN tenant
```

### Vulnerable vs fixed code

```js
// ❌ Bad — tenantId comes from the attacker-controlled request body
app.post('/api/invoices', authenticate, async (req, res) => {
  const { tenantId, amount, customerId } = req.body;
  const invoice = await prisma.invoice.create({
    data: { tenantId, amount, customerId }, // attacker can write ANY tenantId
  });
  res.json(invoice);
});

// Read path is just as dangerous:
app.get('/api/invoices/:id', authenticate, async (req, res) => {
  const invoice = await prisma.invoice.findUnique({ where: { id: req.params.id } });
  res.json(invoice); // no tenant check at all -> any user can read any invoice by guessing IDs
});
```

```js
// ✅ Good — tenantId is derived server-side from the verified auth context
app.post('/api/invoices', authenticate, async (req, res) => {
  const tenantId = req.user.tenantId; // set by your auth middleware from a verified JWT/session
  const { amount, customerId } = req.body; // only non-sensitive fields come from the client

  const invoice = await prisma.invoice.create({
    data: { tenantId, amount, customerId },
  });
  res.json(invoice);
});

// Reads must be scoped by tenant too — not just creates
app.get('/api/invoices/:id', authenticate, async (req, res) => {
  const invoice = await prisma.invoice.findFirst({
    where: { id: req.params.id, tenantId: req.user.tenantId }, // scoped
  });
  if (!invoice) return res.sendStatus(404); // 404, not 403 — don't confirm the record exists
  res.json(invoice);
});
```

### Defense in depth — don't rely on "remembering to filter" everywhere

| Layer | How it protects you |
|-------|----------------------|
| **Auth middleware** | Decodes JWT/session once, attaches `req.user.tenantId` — the *only* trusted source |
| **DTO/validation layer** | Whitelist allowed input fields (e.g. Zod schema) so `tenantId` can never even be parsed from `req.body` |
| **ORM query layer** | Prisma middleware (`$use`) that auto-injects `where: { tenantId }` on every query, so a forgotten filter can't leak data |
| **Database layer** | Postgres **Row-Level Security (RLS)** policy keyed on `tenant_id` — even a raw SQL bug can't cross tenants |
| **Audit logging** | Log `tenantId` + `userId` on every write, so a breach is at least detectable |

> Golden rule: **anything that decides "which tenant" is a trust boundary, not a form field.** The client should never be the one asserting its own security scope.

Memory trick: **"The client picks WHAT, the server picks WHO/WHERE."** The client can say *what* amount or *what* customer — never *whose* data it's allowed to touch.

**Key Points:**
- This is **Broken Object Level Authorization (BOLA / IDOR)** — OWASP API Security Top 10, #1 (`API1:2023`).
- It affects **both reads and writes** — a missing tenant filter on a `GET` leaks data just as badly as trusting it on a `POST`.
- `tenantId` (like `userId`, `role`, `isAdmin`) belongs to the category of fields that must **never** be client-writable — treat it like you'd treat a password field, but in reverse (never *readable as input*, always *derived*).
- Add a DB-level backstop (RLS) so a single missed `where` clause in application code isn't a full breach.
- Return `404` (not `403`) for cross-tenant lookups, so attackers can't even confirm another tenant's record IDs exist.

**Interview one-liner:** Trusting a client-supplied `tenantId` lets any authenticated user impersonate any tenant just by editing the request body — it's Broken Object Level Authorization (OWASP API #1). The fix: derive `tenantId` from the verified auth token server-side, never from `req.body`, and enforce it again at the database layer (e.g. Postgres RLS) as a backstop.

---

## 3. Processing 1M rows in Prisma without page-based pagination or crashing Node's memory

*Definition:* Fetching all 1,000,000 rows in one `findMany()` call materializes **1 million JS objects at once** in Node's heap — that blows past the default heap limit (~1.5–4 GB) and crashes the process. The fix is to **never hold more than one batch in memory at a time**: fetch a small batch, process it fully, discard it, fetch the next — using **cursor-based pagination**, not offset/page-based pagination.

Real life: You don't drink the ocean in one gulp — you sip it in cups. Each cup (batch) is small enough to swallow, and you don't need to hold every cup you've already drunk.

```
❌ One giant query                         ✅ Cursor-based batching
────────────────────                       ─────────────────────────
findMany()  → 1,000,000 rows               loop:
     │                                        findMany({ take: 1000, cursor })
     ▼                                             │
 all 1M objects sit in                             ▼
   Node's heap at once                        process 1,000 rows, then DISCARD them
     │                                             │
     ▼                                             ▼
  💥 OOM crash                              move cursor forward, repeat
                                             (peak memory ≈ size of ONE batch)
```

### Why offset ("page") pagination is also the wrong tool here

`skip`/`take` (`OFFSET`/`LIMIT`) pagination works fine for a UI showing "page 4 of results" to a human, but it's a poor fit for *processing* a whole table:

```sql
-- Page 999: the DB must scan and discard the first 999,000 rows every time
SELECT * FROM users ORDER BY id OFFSET 999000 LIMIT 1000;
```

| | Offset (`skip`/`take`) pagination | Cursor (`cursor`/`take`) pagination |
|--|-----------------------------------|--------------------------------------|
| How it finds the next batch | `OFFSET N` — DB scans & discards N rows | `WHERE id > lastSeenId` — index seek |
| Speed at row 999,000 | Slow — gets worse every page (O(N)) | Fast — same speed as row 1 (O(log N)) |
| Safe if rows are inserted/deleted mid-scan | ❌ Can skip or repeat rows | ✅ Stable — always continues after the last real ID seen |
| Good for | "Page 3 of 10" UI pagination | Bulk export, migrations, ETL, background jobs |

### Cursor-based batch processing in Prisma

```js
// ✅ Streams through 1M rows using a bounded amount of memory
async function processAllUsers() {
  const BATCH_SIZE = 1000;
  let cursor; // undefined on the first iteration

  while (true) {
    const users = await prisma.user.findMany({
      take: BATCH_SIZE,
      skip: cursor ? 1 : 0,        // skip the cursor row itself, not a page offset
      cursor: cursor ? { id: cursor } : undefined,
      orderBy: { id: 'asc' },      // orderBy column MUST be unique + indexed
    });

    if (users.length === 0) break; // done

    await processBatch(users);     // do the work, then let this batch be garbage-collected
    cursor = users[users.length - 1].id;
  }
}

async function processBatch(users) {
  // e.g. send emails, transform + write elsewhere, etc — in small transactions if writing
  for (const user of users) {
    await sendWelcomeEmail(user);
  }
}
```

```js
// ✅ Same idea, expressed as an async generator — keeps the "streaming" intent explicit
async function* streamUsers(batchSize = 1000) {
  let cursor;
  while (true) {
    const batch = await prisma.user.findMany({
      take: batchSize,
      skip: cursor ? 1 : 0,
      cursor: cursor ? { id: cursor } : undefined,
      orderBy: { id: 'asc' },
    });
    if (batch.length === 0) return;
    yield batch;
    cursor = batch[batch.length - 1].id;
  }
}

for await (const batch of streamUsers()) {
  await processBatch(batch); // only one batch resident in memory at a time
}
```

> **Prisma doesn't give you a true DB result-set stream** — every `findMany()` call fully materializes its own result into memory. That's fine, *as long as each call's `take` is small* (hundreds to a few thousand rows). The "streaming" behavior comes from looping over small queries, not from one query streaming rows lazily.

### If you truly need a raw streaming cursor (very large exports/ETL)

For true server-side cursor streaming (row-by-row, not batch-by-batch), drop to raw SQL with a library like `pg-cursor`, or better — for pure export/migration jobs, use the database's own bulk tools (`COPY`, `pg_dump`) instead of routing gigabytes through the ORM at all. The interview answer that matters most is: **recognize when the job is no longer an "app query" but a "data pipeline job,"** and reach for the right tool.

**Key Points:**
- The crash isn't caused by "too much data in Postgres" — it's caused by **holding all of it as live JS objects in one array at the same time.**
- **Cursor pagination** (`cursor` + unique `orderBy` column) is O(1)-ish per page; **offset pagination** (`skip`) degrades the deeper you page.
- Keep batch size modest (500–5,000) — big enough to be efficient, small enough that one batch never threatens the heap.
- For **writes**, wrap each batch in its own small `$transaction`, never one transaction for all 1M rows (long transactions hold locks and bloat memory too).
- If you're just moving/exporting data rather than transforming it row-by-row in app code, prefer database-native bulk tools over the ORM.

**Easy rule of thumb:** If you can't say "at most N rows are ever in memory at once" for some fixed, small N — you haven't actually fixed the memory problem, you've just moved it.

**Interview one-liner:** Loading all 1M rows in one `findMany()` crashes Node because every row becomes a live JS object held in memory at once; the fix is cursor-based batching (`cursor` + `take` on a unique indexed column, not `skip`-based page offsets) so only one small batch is ever resident in memory, processed and discarded before the next batch is fetched.

---

## Quick Revision Cheat Sheet

- **Resumable uploads (50GB file):** chunk the file client-side, upload chunks independently with checksums, track received chunk indexes server-side (Redis/DB), and on reconnect resume only the missing chunks — use S3 Multipart Upload or the tus protocol rather than hand-rolling it.
- **`tenantId` from `req.body`** = Broken Object Level Authorization (OWASP API #1) — the client can impersonate any tenant by editing the payload. Always derive `tenantId` from the verified JWT/session server-side, never trust it as client input, and back it up with DB-level Row-Level Security.
- **1M-row Prisma processing:** never `findMany()` everything at once. Use **cursor-based** pagination (`cursor` + `take` on a unique indexed column) in a loop/async-generator so only one small batch is ever in memory — offset (`skip`) pagination gets slower with depth and isn't meant for bulk processing anyway.
