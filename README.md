# search-contract

The [search contract](docs/glossary/search-contract.md) is one search request and one answer that every searchable service implements, so a front end can query
several services at once and merge their results without translating them, and without copying their
data into one index.

- [SPEC.md](SPEC.md): the contract, version 1 (draft).
- [model.json](model.json): the shared embedding model and word handling, the one place they are defined.

Status: draft. Planned: a JSON Schema for request and answer, and a conformance test a provider runs
against itself.
