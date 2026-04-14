# Threads API Contract

This is the lightweight contract between the Dart server and the Flutter client
for the threads feature, introduced at `v0.4`. The file is intentionally
TypeScript-interface-styled. OpenAPI is not introduced in this series.

Language policy: the series body is Korean; contract artifacts like this one
are English so they double as a handoff document for tooling.

## Scope

- Authenticated users can create, list (paginated), read, update, and delete
  threads.
- Each thread has a title, body, and author (the user id from the JWT `sub`).
- All timestamps are ISO-8601 UTC strings. Field casing is **camelCase** on
  the wire (JSON). The Postgres column casing is `snake_case` — transformation
  happens in `apps/dart_server/lib/models/thread.dart`.

## Types

```ts
interface Thread {
  id: number;          // bigint, DB-assigned
  authorId: number;    // FK → users.id
  title: string;       // non-empty, ≤ 200 chars
  body: string;        // ≤ 20_000 chars
  createdAt: string;   // ISO-8601 UTC
  updatedAt: string;   // ISO-8601 UTC
}

interface ThreadCreateInput {
  title: string;
  body: string;
}

interface ThreadUpdateInput {
  title?: string;
  body?: string;
}

interface ThreadListPage {
  items: Thread[];
  nextCursor: string | null;   // opaque, server-issued
}
```

## Endpoints

All protected routes require `Authorization: Bearer <jwt>`. `401` is returned
if the header is missing or the token is expired. `403` is returned for
`PUT`/`DELETE` attempts on threads owned by a different user.

### `POST /threads`

- Body: `ThreadCreateInput`
- Response: `201` + `Thread`

### `GET /threads?cursor=<opaque>&limit=<int>`

- Query: `cursor` optional, `limit` 1–50 (default 20)
- Response: `200` + `ThreadListPage`
- Ordering: `created_at DESC, id DESC`

### `GET /threads/:id`

- Response: `200` + `Thread`, or `404` if not found

### `PUT /threads/:id`

- Body: `ThreadUpdateInput` (at least one field)
- Response: `200` + `Thread`
- Only the author may edit

### `DELETE /threads/:id`

- Response: `204`
- Only the author may delete

## Decisions worth noting

- **camelCase on the wire**: chosen because the Flutter side generates models
  for JSON consumption. The server is the only place that needs to translate,
  and it does so in one file.
- **Opaque cursor**: the string is a base64-encoded `{createdAt, id}` tuple.
  Clients must not parse it. This keeps pagination stable under inserts.
- **No soft delete** in v0.4. Hard delete is fine for MVP scope. Restoration
  is out of scope.
- **Validation boundaries**: the server is authoritative. Clients may enforce
  UX constraints (title non-empty, body ≤ 20k) but must handle `422` from the
  server gracefully.
