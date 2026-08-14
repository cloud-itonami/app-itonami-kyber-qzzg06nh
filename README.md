# kyber-qzzg06nh

**This repository is the extracted home of the "Kyber" business-portal application from the
etzhayyim monorepo.** The name says nothing on its own — `kyber` was the application's
codename inside `etzhayyim/root`, and `qzzg06nh` is the extraction suffix that kept it
distinct from the other Kyber extraction. It holds one thing that builds and runs on its own
(`kotoba/`), and one thing that does not (`appview/`, which still expects the monorepo
around it).

## What is actually in here

| Path | What it is | State |
|---|---|---|
| `kotoba/` | Knowledge-graph actor: entities + events + reports on AT Protocol PDS records | **Self-contained and working.** Typechecks; 3 tests pass |
| `appview/*/kotodama.jsonld` | 18 governance descriptors — roles, RACI, compliance frameworks, agent system prompts | Data only. **Nothing in this repository reads these files** |
| `appview/{hr,inbox}/src/app.ts` | 2 real Cloudflare Worker actors (677 and 192 lines) on `@etzhayyim/kotodama-host-sdk` — XRPC commands, heartbeat, firehose handler | **Source is real; not buildable here.** See below |
| `appview/*/svelte/` | 12 Svelte packages | **Not installable standalone.** 11 of 12 depend on `workspace:*` packages that exist only in the monorepo |
| `PROJECT.jsonld` | Provenance record of the pre-extraction project, with a milestone list | Historical — see "What is not true today" |
| `README.edn`, `migration.edn` | The extraction's canonical files (`migration.edn` names them as such) | Authoritative for provenance |
| `OWNERS`, `NOTICE` | Approver list; Apache-2.0 + etzhayyim Charter Compliance Rider v3.1 | Current |

## `kotoba/` — the part that runs

A knowledge graph of three record kinds stored as AT PDS records, with referential integrity
enforced in the actor rather than the store:

- **entity** — `defineEntity` / `getEntity` / `listEntities` / `archiveEntity`
- **event** — `recordEvent` / `listEvents`. Every event must name an existing entity;
  a missing one returns `entityNotFound`, not a dangling row.
- **report** — `submitReport` / `publishReport` / `listReports`. The entity reference is
  optional; publishing twice is rejected rather than idempotent.
- **`coverage`** — counts by kind and by status across all three.

Two invariants are worth knowing before you write to it:

1. **These records are public by design and must not carry PII.** `types.ts` states this as
   an axis note (ADR-2605172400): generic name/kind/category/status only, no settlement, no
   liability. The type system does not stop you; the convention does.
2. **Identity is derived, not stored.** Every record's DID is built from
   `did:web:kyber-qzzg06nh.etzhayyim.com:` plus its kind and id.

To run it, see **[`docs/operator-quickstart.md`](docs/operator-quickstart.md)**.

## `appview/` — what survived the extraction and what did not

Eighteen directories, one per sub-application of the portal. Every one carries a
`kotodama.jsonld` governance descriptor: the DID, an accent colour, an agent system prompt,
a RACI role, and compliance frameworks (ITIL 4, SOC 2, and others). These are the most
complete artefacts here, and nothing consumes them — treat them as the record of what each
sub-application was *meant* to be accountable for.

Two directories carry working source beyond that:

- **`hr`** — 677 lines implementing 15 XRPC commands (employee lifecycle with probation
  arithmetic, positions, skill-gap analysis with a readiness percentage, weighted performance
  evaluation, training enrolment), plus a heartbeat and a firehose commit handler.
- **`inbox`** — 192 lines, same shape.

Three things stop either from running as extracted:

1. **`@etzhayyim/kotodama-host-sdk` is imported but not declared** — no `package.json`
   anywhere in this repository lists it.
2. **Their reads are already stubbed upstream**, independent of the extraction. `hr` carries
   `TODO(kyber-hr): vertex_hr_employee not in @etzhayyim/graph-schema — reads return empty`
   on the list commands, and `cmdCoverageStats` is marked `SQL deprecated 2026-04-12 — return
   zero counts`. Writes go to Kysely tables; reads return empty arrays. Do not read a passing
   command as a populated graph.
3. **`wrangler.jsonc` points `main` at `svelte/.svelte-kit/cloudflare/_worker.js`** — a
   SvelteKit build output — and the Svelte packages cannot be installed here (below).

The other 10 `svelte/` directories are entry scaffolds: `App.svelte` renders the directory
name and the words "Vite entry scaffold after SvelteKit cleanup" (the `tech` one says "UI
entry scaffold for kyber tech component"; `inbox` has SvelteKit routes instead of an
`App.svelte`). The `package.json` files are the pre-cleanup ones, still listing SvelteKit,
Tailwind, ConnectRPC and `@etzhayyim/design-system`.

## What is not true today (measured 2026-08-14)

`PROJECT.jsonld` describes the application as it stood inside the monorepo. It is kept as a
provenance record, not as a status page.

- **`kyber.etzhayyim.com` does not resolve** (DNS failure, not a 404), though
  `PROJECT.jsonld` says the portal is "Accessible at" it. The apex `etzhayyim.com` does
  answer (200).
- **No `*.etzhayyim.com` host referenced in this repository resolves** — not
  `kyber-qzzg06nh.etzhayyim.com` (so the `did:web` identity `kotoba/src/types.ts` derives
  record DIDs from has no published DID document), and not the `wrangler.jsonc` route hosts
  `mdtpqjc8.etzhayyim.com` or `inb0x4k2.etzhayyim.com`.
- **The milestone list — 34 ConnectRPC actors, Temporal workflows, Tilt orchestration, 244
  BDD e2e routes — describes the monorepo, not this repository.** Two of those actors came
  across, in the state described above.

## Provenance

Extracted from `etzhayyim/root` at `60-apps/etzhayyim-project-kyber-qzzg06nh`, source
revision `3034f3ac513c6ec6a62cd8df3d0e8f13a8b36ab0` (`migration.edn`). Originally migrated
out of `etzhayyimcojp` on 2026-05-21 in the ALIGN batch (`NOTICE`).

## Nearest repositories — do not confuse these

- **`kotoba-lang/kyber`** is a *separate* extraction of the same codename: its `README.edn`
  names it `com-etzhayyim-app-kyber` and it carries the langgraph (`lg/`) side. This
  repository carries the kotoba actor and the appview directories.
- **`cloud-itonami/app-open-kyber`** is unrelated despite the shared word: an Apache-2.0
  APQC-aligned open-source ERP built on the kotoba Datom log.

## Licence

Apache-2.0 with the etzhayyim Charter Compliance Rider v3.1. Using, modifying, or
redistributing this software accepts both — see `NOTICE`.
