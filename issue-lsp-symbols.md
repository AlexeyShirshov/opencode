# [FEATURE]: Symbol-based operations for the `lsp` tool

> Template: 🚀 Feature Request. Checkbox "Feature hasn't been suggested before" — verified.

## Problem

The `lsp` tool only works by position: `filePath` plus 1-based `line` and `character`. An agent usually knows a symbol's *name* (e.g. `SqlBuilder.ToCommand`, `Greet`, `pkg.Handler`), not its exact line and column, so it has to grep first. Text search lands in comments and strings, and it cannot answer semantic questions such as "who calls this" or "what implements this" at all.

## Proposal

Add a symbol-name family of operations to the existing `lsp` tool, next to the position-based ones:

- `symbols` — resolve a name to its definition location(s)
- `members` — list the members of a type by name
- `refs`, `callers`, `implementations` — semantic queries by name
- `rename` — `textDocument/rename` by name; dry-run by default, writes only with `apply=true`
- `types` — find types by name substring

These are standard LSP requests, so they work for any language with a configured server, not just one. Pass `filePath` to choose which server answers (any file in the target language); without it, the project's dominant extension is used. When a server is chosen, only servers for that file type are queried, so a polyglot workspace cannot answer a query from an unrelated server.

## Notes

- Reuses the already-running server for the file type (touch one file to start it), so no extra process or memory is added.
- `callers` needs `callHierarchy` and `rename` needs `textDocument/rename`; a server that does not implement them returns nothing for that operation.
- A cold server can return an empty `workspace/symbol` while it loads, so the first query waits (bounded), then later queries are warm.

## Verification

Working prototype: `symbols`, `members`, `types` and `rename` exercised by name on a TypeScript project and on a .NET solution; `tsgo --noEmit` clean.

## Ask

Is this in scope? Happy to trim the surface (e.g. read-only operations only) before opening a PR.
