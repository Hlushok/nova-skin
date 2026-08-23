# Nova Skin manual pilot and Premium background probe design

**Status:** approved by the user on 2026-08-23; ready for implementation

## Decision summary

Nova Skin remains available as a complete UI plugin regardless of Lampac user
group. Only background source availability probing is Premium. The integration
uses upstream Nova's accepted `window.nova_skin_probe_mode()` hook and never
patches the mirrored `nova_skin.js`.

The first rollout is deliberately personal and manual:

- publish one small loader at
  `https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js`;
- the user installs that URL manually in one Lampa profile;
- do not add it to Siaivo `autoload.json`, Lampac `lampainit`, the VPS, or any
  global plugin list;
- the loader sets Nova's mode to `external` before loading the byte-for-byte
  mirror from `https://hlushok.github.io/nova-skin/nova_skin.js`;
- Lampac continues to authorize background checks from the authenticated
  server-side user group, with group 3 as the current Premium boundary.

An ordinary user may use all Nova Skin presentation and navigation features,
see the source list, choose sources manually, and retain failure-driven source
switching. Without Premium authorization, the user receives no server-provided
background availability result and Nova does not fall back to its own client
fan-out.

## Verified baseline

The design depends on three existing and separately owned behaviors.

1. Upstream Nova Skin currently exposes
   `window.nova_skin_probe_mode()` and accepts `legacy`, `external`, and
   `disabled`. In explicit `external` mode, `probeAllowed()` is false,
   `probeRun()` returns before its client probe body, and existing `ghost`
   states remain displayable. Unknown, absent, or throwing hooks fall back to
   `legacy`.
2. The clean mirror design keeps `nova_skin.js` byte-for-byte identical to a
   recorded upstream revision and publishes it from `Hlushok/nova-skin` without
   Lampac-specific code.
3. Lampac's existing private overlay authorizes `checkOnlineSearch` and
   `/lifeevents` using the actual authenticated `requestInfo.user.group`.
   `online.checkOnlineSearchGroup` is 3. Lower groups retain normal source
   listing and manual playback while background probe initiation and result
   polling remain denied.

The upstream hook was introduced and refined in commits
`8a1594e438be7ceaff0c5d27fb22df88dc59f944` and
`13cec1bd5d3a8bcb0f321be2138d9016e04284a7`. The last source revision verified
while writing this design was
`6bac54770d1b44b42aab988fe2b6dd8fc61de080`. Implementation must resolve and
revalidate the current upstream revision instead of assuming that SHA remains
current.

## Goals

- Give the user one manually installable pilot URL.
- Load unmodified Nova Skin through the user's clean mirror.
- Register explicit `external` mode before Nova initializes.
- Keep all Nova UI and manual source behavior available to every group.
- Make Lampac group 3 the only authority that can start and receive background
  source availability checks.
- Prevent Nova from generating a second per-source client probe for both
  Premium and non-Premium users.
- Keep the pilot reversible by removing one installed plugin URL.
- Collect enough browser and server evidence to decide whether a later global
  rollout is safe.

## Non-goals

- Making the complete Nova Skin plugin Premium-only.
- Adding a Premium badge, payment action, or subscription UI in Nova during the
  pilot.
- Adding Nova or the loader to Siaivo `autoload.json` or Lampac `lampainit`.
- Changing `/opt/lampac/init.conf`, the live VPS, or any production service.
- Adding the upstream Nova URL to Lampa's plugin blacklist.
- Preventing a user from independently installing a public copy of Nova.
- Moving group authorization into JavaScript or trusting a client-supplied
  group value.
- Modifying, minifying, bundling, or appending code to mirrored
  `nova_skin.js`.
- Automatically promoting every mirror update into a globally installed
  Lampac plugin.

## Repository and publication boundaries

### Clean mirror

Repository: `Hlushok/nova-skin`

Public payload:

`https://hlushok.github.io/nova-skin/nova_skin.js`

This repository continues to follow
`docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md`. The loader is not
added to its source tree or Pages artifact. Its `nova_skin.js` remains the exact
upstream payload.

The older mirror implementation plan contains a literal six-file repository
allowlist written before this design existed. The follow-on implementation plan
must supersede that allowlist so it preserves this reviewed specification and
its own approved plan documents. The Pages artifact remains unchanged: only
`nova_skin.js` and `.nojekyll` are published. Do not execute the older plan's
exact tree assertions without this revision.

### Pilot loader

Repository: `Hlushok/lampa-plugin`

Source file: `nova_skin_lampac.js`

Public URL:

`https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js`

`Hlushok/lampa-plugin` already publishes from `main` through GitHub Pages. The
pilot adds only the loader file and a concise README entry. It does not change
any existing plugin, playlist, preset, or Pages setting.

### Lampac server

The pilot does not deploy Lampac code or configuration. It consumes the current
server contract only. Before the manual test, current local and live state must
be rechecked independently: active configuration, group threshold, guarded
probe initiation, guarded `/lifeevents`, and the authenticated test account's
actual group.

## Loader contract

The loader is an integration adapter, not a Nova fork. It has five
responsibilities.

1. **Idempotence.** A dedicated global marker prevents the loader from starting
   more than once in the same page. Nova's own `window.nova_skin` marker remains
   the authority for whether the upstream plugin already initialized.
2. **Hook ordering.** Before requesting Nova, the loader assigns
   `window.nova_skin_probe_mode` to a function that always returns
   `external`. It performs no account, group, payment, or URL-parameter check.
3. **Single allowed dependency.** The only dynamically loaded script is the
   stable clean-mirror URL. The loader contains no fallback to the author's URL,
   no alternate mirror list, and no dynamic code evaluation.
4. **Bounded freshness.** The mirror request receives an hourly cache-busting
   query value. This aligns with the planned hourly mirror synchronization while
   allowing a browser cache within the same hour.
5. **Failure isolation.** If Lampa APIs are unavailable or Nova fails to load,
   the loader records a concise console error and exits without changing Lampa
   navigation, storage, source lists, or Lampac configuration. It does not retry
   forever and does not load a legacy fallback.

The loader must not include a user identifier, email, access code, Premium
flag, Lampac host, secret, or server group. The `external` mode is intentionally
the same for everyone; the server remains the authorization boundary.

## Runtime data flow

### Common startup

1. The user manually installs the pilot loader URL through Lampa's Extensions
   screen.
2. Lampa executes `nova_skin_lampac.js`.
3. The loader registers explicit `external` probe mode.
4. The loader requests the current clean-mirror `nova_skin.js`.
5. Nova initializes normally and exposes its complete UI.

### Premium group 3

1. The normal Lampac Online flow starts the server-side background availability
   check for the authenticated group-3 user.
2. The browser may initiate the existing orchestration and poll endpoints, but
   Nova does not issue its own per-source `checksearch=true` fan-out.
3. Lampac returns availability through the existing source result shape,
   including `ghost` state where supported.
4. Nova displays the supplied state and known quality without doing a second
   pass.

### Other groups

1. The same Nova UI loads and the same `external` hook remains active.
2. Lampac denies background check initiation and `/lifeevents` according to the
   existing server policy.
3. Nova does not fall back to client probing when `ghost` is absent.
4. Sources remain visible and manually selectable. Failure-driven auto-switch
   remains independent of background probing.

## Why `external` is global

Returning `external` only for group 3 would be the wrong boundary. A lower
group would then fall back to `legacy` and Nova could generate the exact client
fan-out that the integration is intended to suppress. A browser-provided group
value would also be forgeable.

Therefore the loader returns `external` for every user who installs it. Lampac
alone decides whether a background check is admitted and whether any result is
available. This separates presentation from authorization and preserves the
server-side enforcement already deployed.

## Blacklist decision

Lampa's URL blacklist is not part of this pilot. Nova is intentionally
available to all groups, and a public upstream file cannot be made exclusive by
blocking one URL. A blacklist may later prevent duplicate or known-incompatible
Nova URLs, but it cannot be treated as a Premium security boundary and requires
a separate decision.

## Failure modes and safe behavior

- **Loader URL unavailable:** Nova does not load; the rest of Lampa continues.
- **Mirror URL unavailable:** the hook remains registered but Nova does not
  initialize; the rest of Lampa continues.
- **Loader installed twice:** the integration marker makes the second execution
  a no-op.
- **Nova already loaded first:** the loader must not reload Nova. It still
  registers `external`, but the pilot is considered invalid because ordering
  was not proven; remove the duplicate/original Nova URL and restart before
  testing.
- **Upstream removes or changes the hook:** compatibility checks fail and the
  pilot URL is not handed off as verified. The clean mirror may continue to
  mirror upstream because it has a separate contract, but a global rollout is
  blocked.
- **User is unauthenticated or not group 3:** Nova still loads; Lampac rejects
  Premium probe endpoints and Nova performs no fallback fan-out.
- **Stale browser cache:** the hourly query changes the mirror URL. Manual
  retesting may use a new hour bucket or clear only the pilot plugin cache.
- **Plugin removed after testing:** Nova does not load on the next full app
  restart. Nova-specific preferences may remain dormant in local storage; the
  pilot does not erase unrelated user settings.

## Verification strategy

### Static and publication checks

- `node --check` passes for both the mirror payload and loader.
- The loader source contains exactly one remote script origin and that URL is
  the clean mirror.
- The hook assignment appears before the Nova load call.
- The loader has an idempotence marker and no `eval`, `new Function`, inline
  group, account, token, password, or access-code logic.
- The current mirror still contains `window.nova_skin_probe_mode`, accepts all
  three modes, restricts `probeAllowed()` to `legacy`, and guards `probeRun()`.
- `Hlushok/lampa-plugin` Pages returns the loader with HTTP 200, and downloaded
  bytes match repository `main`.
- `Hlushok/nova-skin` Pages returns Nova with HTTP 200, and downloaded bytes
  match the recorded upstream revision.

### Loader behavior checks

- A stubbed Lampa harness proves the loader registers `external` before asking
  to load Nova.
- Running the loader twice makes only one Nova load request.
- A simulated load failure produces one bounded error and no retry loop.
- A pre-existing `window.nova_skin` prevents a duplicate Nova load.

### Manual Premium test

- Record the test account identifier only in private operational evidence, not
  in repository files or chat output, and verify its live server-side group is
  3.
- Install only the pilot loader URL and fully restart Lampa.
- Confirm Nova UI, source listing, manual source selection, and normal playback.
- Confirm `window.nova_skin_probe_mode()` returns `external`.
- Open a card with several online sources and capture the browser network flow.
- Confirm no Nova-generated per-source `checksearch=true` fan-out occurs.
- Confirm the normal Lampac background flow completes and Nova displays the
  returned `ghost`/quality state where the provider supplies it.

### Manual non-Premium test

This test is required before any global rollout, but it is not required before
the user begins the personal group-3 pilot.

- Use a separate authenticated lower-group account; never alter the user's
  Premium account merely to test denial.
- Confirm Nova UI and sources remain available.
- Confirm manual source selection and playback remain available.
- Confirm no Nova-generated per-source client fan-out occurs.
- Confirm background initiation/results remain denied by Lampac and no
  Premium-only availability state is exposed.

## Pilot acceptance criteria

The personal pilot link may be handed to the user only when all of the following
are true:

1. The clean mirror has been implemented and its public payload is
   byte-for-byte equal to the recorded current upstream file.
2. The upstream file still satisfies the explicit `external` hook contract.
3. The loader passes syntax, static contract, idempotence, ordering, and failure
   tests.
4. The public loader and mirror URLs both return HTTP 200 with expected bytes.
5. Neither repository worktree contains unrelated changes or secret-like
   material.
6. No Siaivo `autoload.json`, Lampac loader/configuration, VPS file, or service
   has been changed.

The pilot itself succeeds when the user confirms Nova works normally and fresh
evidence shows group-3 background state without Nova's client fan-out.

## Rollback

The immediate pilot rollback is user-local: remove
`https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js` from Extensions and
fully restart Lampa. No server restart or configuration rollback is needed
because the pilot does not modify Lampac.

If the published loader is defective, revert only its commit in
`Hlushok/lampa-plugin` after preserving evidence. Do not alter the clean mirror
to compensate for a bridge problem.

## Later global rollout

Global installation is a separate authorization. It requires:

- a successful personal group-3 pilot;
- the separate lower-group denial/manual-playback test;
- fresh verification of the live Lampac group guard;
- a compatibility check against the then-current mirrored Nova;
- an explicit choice of distribution path and rollback procedure;
- direct user approval before changing `autoload.json`, `lampainit`, the VPS, or
  any globally consumed plugin list.

Until that approval, the public loader may exist but is neither advertised nor
automatically installed.
