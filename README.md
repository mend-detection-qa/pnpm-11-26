# catalogs-workspace-deploy-v11_26

**Pattern:** `catalogs-workspace-deploy-v11_26`
**PM version under test:** pnpm 11.26
**Schema version:** 1.2 (version-variation probe with `pm_version_under_test`)
**Generated:** 2026-09-07

## Feature exercised

Three pnpm 11.26 features in one monorepo workspace probe:

1. **Catalogs + `workspace:` protocol** (`tree_structure`) — The
   `pnpm-workspace.yaml` defines a catalog entry whose value is a
   `workspace:*` specifier, pointing at the `@probe/shared` workspace
   package. In the lockfile this resolves to a `link:` path, not a
   registry fetch. This tests whether Mend follows the catalog
   indirection through `workspace:` and reports `source: "local"`.

2. **Trust flags on `pnpm remove`/`update`** (`install_command`) —
   `esbuild@0.21.5` has platform-specific postinstall scripts (native
   binary selection). `.npmrc` sets `trust-public-scripts=true`,
   which is the config-file equivalent of the `--trust-public-scripts`
   flag introduced for `pnpm remove` and `pnpm update` in pnpm 11.26.
   The flag is **behavioral** (controls whether scripts run at install
   time) and does NOT change the resolved dependency set or lockfile
   content. Mend must detect `esbuild@0.21.5` regardless of trust-flag
   state.

3. **Deploy + `injectWorkspacePackages` + peer resolution**
   (`tree_structure`, `version_constraints`) — `apps/consumer` declares
   `dependenciesMeta: { "@probe/shared": { "injected": true } }`.
   Under pnpm 11.26, when `injectWorkspacePackages` is active, the
   injected copy carries its own peer-dep graph. The injected
   `@probe/shared` peers on `react@^18.3.1`; the expected tree must
   show `react@18.3.1` resolved correctly inside the injected package's
   subtree (not lost due to the copy-not-symlink deployment).

## Workspace layout

```
probe-root/
├── package.json                 workspace root (no direct deps)
├── pnpm-workspace.yaml          catalog with workspace: + registry entries
├── pnpm-lock.yaml               v9 single-document lockfile
├── .npmrc                       trust-public-scripts=true, auto-install-peers
├── .whitesource                 Bucket A: pins pnpm 11.26.0 + node 22.14.0
├── packages/
│   └── shared/
│       └── package.json         @probe/shared@1.0.0 (react + zod deps)
└── apps/
    └── consumer/
        └── package.json         @probe/consumer@1.0.0 (injected @probe/shared + esbuild)
```

## Expected dependency tree (summary)

Workspace packages (importers):

| Importer | Direct dependencies |
|---|---|
| `packages/shared` | `react@18.3.1` (registry), `zod@3.23.8` (registry) |
| `apps/consumer` | `@probe/shared@link:../../packages/shared` (local/injected), `esbuild@0.21.5` (registry) |

Transitive chain under `react@18.3.1`:
- `loose-envify@1.4.0` → `js-tokens@4.0.0`

Transitive chain under `esbuild@0.21.5`:
- `@esbuild/linux-x64@0.21.5` (optional, platform-specific)

The catalog entry `"@probe/shared": "workspace:*"` resolves to
`link:../../packages/shared` in the lockfile. Mend must report this
as `source: "local"`, NOT as a registry package.

## Mend resolver behavior

**Resolver:** `PnpmLockCollector` (pnpm v9 path via `PnpmParserV9Impl`).

Key resolution steps:
1. Reads `importers` section for workspace dep sets.
2. Reads `snapshots` section for transitive resolution facts.
3. Catalog entries in manifests use `catalog:` specifier; the lockfile
   records the resolved version directly — Mend reads from the lockfile,
   so catalog indirection is transparent UNLESS Mend tries to validate
   the manifest specifier against the lockfile value.
4. The `link:` resolution for `@probe/shared` must be reported as
   `source: "local"` per the resolver's `link:` handling.

**Known Mend limitation to probe:** If `PnpmParserV9Impl` reads the
catalog section of the lockfile and misinterprets the `workspace:*`
catalog specifier as an unresolvable version, it may drop `@probe/shared`
from the consumer's tree. The expected tree encodes the CORRECT output;
discrepancy flags the bug.

## Mend config

**Bucket A — default-emit.** `js-pnpm` has no dynamic version detection
from the manifest. This probe ships `.whitesource` with:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "pnpm": "11.26.0",
      "node": "22.14.0"
    }
  }
}
```

`configMode` is `"AUTO"` because no `whitesource.config` ships with
this probe. The `pnpm` and `node` keys are from the `install-tool`
supported list for `js-pnpm`. Both are pinned to exact versions to
prevent transitive-set drift across scans.

**Additional dimension — resolver-driven pre-step:** This probe does
NOT ship a `whitesource.config`; Mend will use lockfile-driven
detection (default). The trust-flag behavior (`trust-public-scripts`)
is a pnpm CLI concern, not a UA pre-step concern — it affects what
happens when `npm.runPreStep=true` is set externally, but the probe
itself does not require the pre-step to be on.

## Probe categories

- `tree_structure` — catalogs resolving via `workspace:` protocol;
  injected workspace package in tree.
- `install_command` — `trust-public-scripts` flag on `pnpm remove` /
  `pnpm update` documented.
- `version_constraints` — peer dep of injected package resolved
  correctly under pnpm 11.26 deploy+inject behavior.

## Added from

pnpm 11.26 release:
- Catalogs now resolve workspace dependencies via `workspace:` protocol.
- `pnpm remove` and `pnpm update` accept `--trust-public-scripts` and
  related trust flags.
- Deploy behavior changed regarding `injectWorkspacePackages` and
  peer resolution — injected copies carry their own peer graphs.

## Status

Generated (first run). `pm_version_under_test = "11.26.0"`.
