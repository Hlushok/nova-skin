# Nova Skin upstream mirror design

**Status:** approved for implementation planning on 2026-08-21

## Context

The upstream repository `amikdn/amikdn.github.io` is a public, flat collection of
many independent Lampa plugins. `nova_skin.js` is the complete Nova Skin entry
file: it contains its own JavaScript, CSS, translations, icons, settings, and
initialization. It does not need `test.js` or another `main.js`.

The downstream project must provide a stable URL owned by `Hlushok` and accept
future upstream changes to `nova_skin.js` automatically. It must not silently
import or run unrelated upstream plugins or upstream GitHub Actions workflows.

## Goals

- Create a real GitHub fork named `Hlushok/nova-skin` from
  `amikdn/amikdn.github.io`, preserving GitHub's parent/fork provenance.
- Publish an exact, unmodified copy of upstream `nova_skin.js` from downstream
  GitHub Pages.
- Check upstream hourly and also support a manual synchronization run.
- Accept every upstream change to this file without editorial modification,
  subject only to transport and JavaScript syntax checks.
- Keep the published URL stable:
  `https://hlushok.github.io/nova-skin/nova_skin.js`.

## Non-goals

- Mirroring other files from the upstream repository, including `test.js`.
- Combining Nova Skin with another plugin or changing Nova Skin behavior.
- Performing a security review of each upstream release before publication.
- Adding the URL to Siaivo `autoload.json`, Lampac configuration, or any
  production/VPS installation in this task.
- Claiming authorship or applying a new license to the upstream code. No license
  was found in the inspected upstream repository, so the README will retain
  attribution and link to the source without inventing license terms.

## Repository shape

The downstream `main` branch will intentionally contain only:

- `nova_skin.js` — byte-for-byte upstream payload;
- `README.md` — purpose, upstream attribution, public URL, and sync behavior;
- `.nojekyll` — direct static publication through GitHub Pages;
- `.github/workflows/sync-upstream.yml` — downstream-owned sync automation;
- `docs/superpowers/` — the reviewed design and implementation plan.

Unrelated files inherited by the initial fork will be removed from downstream
`main`. The repository remains in the GitHub fork network even though its branch
is deliberately reduced to the Nova Skin distribution surface.

## Synchronization flow

1. GitHub Actions starts at minute 17 of every hour and through
   `workflow_dispatch`. A concurrency group permits only one sync at a time.
2. The workflow queries the public GitHub API for the newest commit on upstream
   `main` that affects `nova_skin.js`.
3. It downloads `nova_skin.js` from that immutable commit SHA into a temporary
   path. This ties the bytes and provenance to one exact upstream revision and
   avoids a branch-update race.
4. Transport checks require a successful download and a non-empty regular file.
   `node --check` verifies JavaScript syntax without executing the downloaded
   plugin. There are no behavioral, content, or style gates.
5. If the bytes match downstream, the run exits successfully without a commit.
6. If they differ, the temporary file atomically replaces `nova_skin.js`. The
   workflow records the upstream commit SHA and SHA-256 digest in its commit
   message, commits through a dedicated automation identity, and pushes to
   downstream `main`.
7. GitHub Pages rebuilds from the root of `main`, preserving the stable URL.

The workflow fetches only `nova_skin.js`; it never merges the complete upstream
branch. Consequently, a new upstream plugin, configuration file, or workflow
cannot enter the downstream repository through synchronization.

Because GitHub disables scheduled workflows by default when a public repository
is forked, implementation explicitly enables this downstream workflow before
the first manual run. GitHub also documents that a public repository's scheduled
workflows can be disabled after 60 days without repository activity. Normal
Nova Skin sync commits provide activity while upstream remains active; the
minimal mirror will not manufacture heartbeat commits. If upstream has no file
changes for that long, the owner must re-enable the workflow manually. This
platform limitation will be stated in the README and can later be covered by a
separately approved external watchdog if needed. See GitHub's
[workflow enablement documentation](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/disable-and-enable-workflows).

## Failure behavior

- API, download, syntax, or push failure makes the Action fail and leaves the
  last successfully published plugin unchanged.
- A syntactically broken upstream revision is not published. This is the only
  content-related safety stop and protects users from receiving a file that a
  JavaScript engine cannot parse.
- A no-change run creates no commit.
- A push race is handled by re-running against the current downstream `main`;
  synchronization never force-pushes.
- Action failures remain visible in the repository's Actions history. No
  external alerting is introduced in this task.

## Security and permissions

- The repository and Pages site remain public.
- Workflow default permissions are read-only; the sync job receives only
  `contents: write` so it can commit the mirrored file.
- No repository secrets are required.
- Downloaded JavaScript is parsed with `node --check` but never loaded, sourced,
  or executed by the workflow.
- The workflow itself stays downstream-owned and is never replaced from
  upstream.
- Reusable actions are pinned to immutable commit SHAs during implementation.
- No production server or Lampac configuration is accessed or changed.

## Initial publication

The initial downstream `nova_skin.js` will be imported from a recorded upstream
commit using the same download and validation rules as later synchronizations.
GitHub Pages will be configured to publish from `main` at the repository root.
The README will clearly identify `amikdn/amikdn.github.io` as the upstream source
and describe this repository as an automatic mirror/fork, not an independent
rewrite.

## Verification and acceptance

Implementation is accepted only when fresh checks demonstrate all of the
following:

1. GitHub reports `Hlushok/nova-skin` as a public fork whose parent is
   `amikdn/amikdn.github.io`.
2. The downstream branch contains only the agreed distribution and operational
   files; unrelated upstream plugins and workflows are absent.
3. Local and workflow syntax checks pass for `nova_skin.js` and the workflow
   configuration.
4. The downstream file SHA-256 equals the file downloaded from the recorded
   upstream commit.
5. A manual sync run succeeds, and a second no-change run does not create a new
   commit.
6. GitHub Pages reports a successful build; the public URL returns HTTP 200 and
   its downloaded bytes have the same SHA-256 as downstream `main`.
7. The hourly schedule and manual dispatch are present, and GitHub reports the
   workflow as enabled after the fork-default disabled state is cleared.
8. No Siaivo autoload, Lampac workspace, VPS, or production configuration was
   modified.

## Later, separately authorized work

After the mirror is proven, its Pages URL can be considered for Siaivo's public
plugin list through `autoload.json`. That is a separate deployment decision and
is intentionally outside this design.
