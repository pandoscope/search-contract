# Search contract, version 1 (draft)

One request and one answer that every searchable service implements, so a front end can query several
services at once and merge their results without translating, and without copying their data into one
index.

## Roles

- A **provider** holds its own data and index, applies its own redaction, and answers search requests.
- A **front end** sends one request to every provider at once, merges the answers and shows one list,
  each result labelled with its provider. A front end may also be a provider of its own data.

## What providers share

1. **This interface.**
2. **Word handling:** BM25 over text tokenized with a German stemmer and ASCII folding. Reference
   setting (ParadeDB `pg_search`): `pdb.simple('stemmer=german', 'ascii_folding=true')`.
3. **The embedding model of record:** `Qwen3-Embedding-0.6B`, L2-normalised vectors of dimension 1024.
   Documents are embedded as they are; queries with the prefix
   `Instruct: <task>\nQuery:<query>`, where `<task>` is fixed in `model.json` of this repository.
   Text is cut into chunks of about 2,000 characters before embedding.

A change to 2 or 3 is a new contract version.

## Why meaning scores merge and word scores don't

With one model, a cosine similarity is absolute: it does not depend on what else an index holds, so
meaning results from several providers are merged by score. A BM25 score depends on each index's own
statistics (document frequencies, document lengths), so identical text scores differently in two
indexes even with the same engine. Sharing those statistics per query (as distributed engines do) costs
a round trip and code on both sides; instead, word results are merged by rank with reciprocal rank
fusion (RRF, k = 60).

## Request

`POST <base>/search/v1`, `Content-Type: application/json`, `Authorization: Bearer <token>`.

| Field | Required | Meaning |
| --- | --- | --- |
| `q` | yes | the query as typed: words, `"a phrase"`, `-excluded` |
| `vector` | no | the query embedding, made once by the front end with the model of record |
| `model` | with `vector` | the model id the vector was made with |
| `modes` | no | `["words", "meaning"]` (default both) or one of them |
| `time` | no | `{"from": ISO 8601, "to": ISO 8601}` |
| `kinds` | no | kinds to include, from `GET <base>/search/v1/kinds` |
| `limit` | no | results per mode, default 50 |

Because the front end sends the vector, every provider ranks by the identical vector, and no provider
needs a model loaded at query time. A provider without vectors of the model of record answers `words`
only.

## Answer

```json
{
  "contract": 1,
  "service": "example-provider",
  "model": "Qwen3-Embedding-0.6B",
  "took_ms": 84,
  "partial": false,
  "words":   [ { "...": "result" } ],
  "meaning": [ { "...": "result" } ]
}
```

A result:

| Field | Meaning |
| --- | --- |
| `id` | stable within the provider, so a front end can hide or pin a result |
| `kind` | one of the provider's kinds |
| `title` | short, plain text |
| `snippet` | plain text with matched words between `[[` and `]]`; never HTML |
| `time` | ISO 8601, the item's own time |
| `url` | absolute link to the item in the provider's own interface |
| `ref` | optional: an identifier the owner uses for the item elsewhere |
| `score` | meaning: the cosine similarity; words: the provider's own score, for its order only |

`words` is in the provider's order, `meaning` by cosine, highest first. `partial: true` when the provider
cut its own work short.

`GET <base>/search/v1/kinds` returns `{"contract": 1, "kinds": [{"name", "label"}]}`.

## Rules for providers

- Answer within 1 s. A front end waits about 2 s, then shows what it has and names the providers missing.
- Results leave the provider already redacted; raw data never does.
- Every caller has a token of its own, read-only and for these endpoints only.
- Everything runs locally or self-hosted: neither queries nor results go to a third-party service.
- Logs carry counts and timings, never a query or a result. A query history, if any, belongs to the
  front end.
- A provider on another contract version or model is still usable: the front end ranks its results by
  rank only.

## Merging, in a front end

1. Meaning: every provider's `meaning` list whose `model` equals the front end's, merged by `score`.
2. Words: every `words` list (and meaning lists from other models), fused by rank.
3. The two fused by rank into one list.

## Open

- Shared filter names beyond `time` and `kinds`.
- Paging past `limit`.
- Whether a provider without its own vectors may embed documents at query time.
