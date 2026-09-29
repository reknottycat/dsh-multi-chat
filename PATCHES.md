# PATCHES.md — reknottycat DSH 0.1.7-rc.2 adaptation

Upstream: [`dsh-multi-chat@1.0.3`](https://www.npmjs.com/package/dsh-multi-chat)
(author: [daetz-coder](https://github.com/daetz-coder/dsh-multi-chat), MIT)

This tree is upstream `1.0.3` plus the patches below. Nothing else was touched.
Target engine: DeepSeek Harness `0.1.7-rc.2` (the engine shipped in
`reknottycat/dsh-mobile-apk`, running on the user's vivo PA2573 tablet).

---

## P1 — `lib/client.js`: import `defineStore` from where 0.1.7 actually keeps it

**Symptom (fatal, engine refused to boot):**

```
Failed to load plugins: dsh-multi-chat: import failed:
client-modules: require("@deepseek-ai/dsh-client-runtime/client")
missed the module table
```

**Root cause.** DSH 0.1.7 moved `defineStore` out of `dsh-client-runtime` into
its own package `@deepseek-ai/dsh-client-store`, and `dsh-client-runtime` is no
longer a resolvable specifier in the browser module table.

Evidence gathered on the shipped engine snapshot:

1. `@deepseek-ai/dsh-client-store@0.1.7-rc.2` exports it directly —
   `lib/index.js:178` → `export { createSnapshotStore, defineStore,
   notifySubscribers, shallowEqual }`.
2. It is registered as a **platform seed word** by the web host, which is what
   makes `require()` resolve without a graph row —
   `@deepseek-ai/dsh-client-web/lib/index.js:9` `import * as ClientStore from
   "@deepseek-ai/dsh-client-store"`, `:216` `"@deepseek-ai/dsh-client-store":
   ClientStore` inside `getStaticModules()`, and the same name listed at `:565`.
3. Across the whole engine tree, **33** client packages `require
   ("@deepseek-ai/dsh-client-store")` and **0** require
   `"@deepseek-ai/dsh-client-runtime"` — the migration is already complete
   engine-side.
4. `dsh-client-runtime` is absent from the served combo script. The client
   module table only ever registers `dsh.client.inject` entries that are
   themselves graph rows (`arriveGraphRow`, `dsh-client-modules/lib/client.js`
   — `const dependency = this.graphRows.get(packageName); if (dependency !==
   void 0) …`), and a missing dependency is skipped *silently*, so the failure
   only surfaces later as a `require()` miss.

**Patch** (`lib/client.js`):

```diff
-const __deepseek_ai_dsh_client_runtime_client = __toESM(require("@deepseek-ai/dsh-client-runtime/client"));
+const __deepseek_ai_dsh_client_store = __toESM(require("@deepseek-ai/dsh-client-store"));
```

plus the one call site (`__deepseek_ai_dsh_client_runtime_client.defineStore` →
`__deepseek_ai_dsh_client_store.defineStore`). `defineStore` was the plugin's
only import from that module, so nothing else moves.

The TypeScript source is shipped in the tarball, so the equivalent source-level
change is one line in `src/client/store.ts`:

```diff
-import { defineStore, type EngineStoreHandle } from '@deepseek-ai/dsh-client-runtime/client'
+import { defineStore, type EngineStoreHandle } from '@deepseek-ai/dsh-client-store'
```

> The `import type { ClientContext } from '@deepseek-ai/dsh-client-runtime/client'`
> in `src/client/index.ts` is type-only and is erased at build time, so it needs
> no change.

## P2 — `package.json`: drop the dead `dsh.client.inject` entry

`@deepseek-ai/dsh-client-runtime` is removed from `dsh.client.inject`. It is not
a graph row, so the entry was a no-op that only masked the real requirement; and
`dsh-client-store` is a seed word, so it must *not* be listed there either.

## P3 — version

`1.0.3` → `1.0.3-mavis.1`.

---

## Verified on device

* `node --check lib/client.js` → exit 0.
* Engine `@deepseek-ai/dsh@0.1.7-rc.2`, Node `v24.18.0`, profile `web`.
* Install: `dsh plugin add github:reknottycat/dsh-multi-chat`.
