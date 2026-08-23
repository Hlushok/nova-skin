# Nova Skin Premium Probe Pilot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish a byte-for-byte Nova Skin mirror plus one manually installable Lampac bridge that loads Nova for every user, forces Nova background probing to external mode for every user, and leaves Premium authorization entirely with Lampac group 3.

**Architecture:** The public fork Hlushok/nova-skin mirrors only upstream nova_skin.js and deploys only that payload through a pinned GitHub Pages workflow. The separate Hlushok/lampa-plugin repository hosts a tiny nova_skin_lampac.js adapter. The adapter installs window.nova_skin_probe_mode before loading the mirror and contains no account logic. The existing Lampac server guard remains unchanged and authoritative.

**Tech Stack:** JavaScript ES5-compatible IIFE, Lampa.Utils.putScriptAsync, Node.js syntax checks and node:test/vm harness, Git, GitHub CLI and REST API, GitHub Actions, GitHub Pages, PowerShell, Git Bash, SHA-256.

**Spec:** docs/superpowers/specs/2026-08-23-nova-skin-premium-probe-pilot-design.md

## Global Constraints

- This plan supersedes docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md. Do not execute the older plan separately or use its stale six-file allowlist.
- nova_skin.js must be byte-for-byte equal to the file at the recorded immutable upstream commit. Never append the bridge, minify the payload, or patch the accepted hook.
- Preserve upstream line endings exactly. Run whitespace checks only on downstream-owned files; validate nova_skin.js with its recorded SHA-256 and node --check.
- The mirror repository may contain exactly eight reviewed blobs after construction: the workflow, static marker, README, two specs, two plans, and nova_skin.js.
- The Pages artifact from the mirror contains only nova_skin.js and .nojekyll.
- The bridge repository changes only README.md and nova_skin_lampac.js. Do not add a package manifest, committed test fixture, preset, playlist, or another plugin.
- The bridge returns external for everyone. It must not inspect a user, group, email, token, URL parameter, Premium flag, Lampac host, or payment state.
- The bridge dynamically loads exactly one script: https://hlushok.github.io/nova-skin/nova_skin.js with an hourly query value. It has no author-URL fallback, alternate mirror, eval, retry loop, timer loop, or navigation mutation.
- Do not change Siaivo autoload.json, Lampac lampainit, D:/opt/lampac/init.conf, /opt/lampac/init.conf, current.conf, the VPS checkout, or any service.
- Do not configure a blacklist. It is neither needed for the personal pilot nor a Premium security boundary.
- Preserve unrelated user changes. Stop on a dirty worktree, repository-name collision, unexpected merge conflict, non-fast-forward remote, failed contract gate, or secret scan result.
- All pushes, GitHub repository creation, Pages configuration, and workflow dispatches happen only during execution of this approved implementation plan. Never force-push.
- The personal pilot ends at a manual user checkpoint. Global installation and the lower-group rollout test require separate authorization.

## File Map

### D:/opt/nova-skin

- Create nova_skin.js as exact upstream bytes.
- Create README.md with attribution, stable URL, automation behavior, and limits.
- Create .nojekyll as an empty marker.
- Create .github/workflows/sync-upstream.yml for hourly immutable-source sync and workflow-based Pages deployment.
- Preserve docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md.
- Preserve docs/superpowers/specs/2026-08-23-nova-skin-premium-probe-pilot-design.md.
- Preserve docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md.
- Preserve this implementation plan.

### D:/opt/lampac/lampa-plugin

- Create nova_skin_lampac.js.
- Modify README.md with the personal pilot URL and one-sentence behavior description.

### Read-only verification surfaces

- Inspect D:/opt/lampac-engine-private on custom/prod.
- Inspect D:/opt/lampac/init.conf.
- Inspect /opt/lampac and its active service through the configured lampaua SSH alias.
- Do not edit any of those surfaces.

---

### Task 1: Freeze the current upstream and repository baselines

**Files:**

- Verify: D:/opt/nova-skin
- Verify: D:/opt/lampac/lampa-plugin
- Verify: upstream nova_skin.js

**Interfaces:**

- Consumes: authenticated gh session for Hlushok, both clean local main branches, upstream GitHub API.
- Produces: immutable upstream file SHA, SHA-256, hook-contract result, and proof that Hlushok/nova-skin is either absent or already exactly compatible.

- [ ] **Step 1: Verify both local repositories are clean and correctly addressed**

~~~powershell
$mirrorRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$bridgeRoot = (Resolve-Path -LiteralPath 'D:/opt/lampac/lampa-plugin').Path

foreach ($repoRoot in @($mirrorRoot, $bridgeRoot)) {
  if ((git -C $repoRoot branch --show-current).Trim() -ne 'main') {
    throw "Expected main branch in $repoRoot."
  }
  if (git -C $repoRoot status --porcelain=v1) {
    throw "Dirty worktree in $repoRoot."
  }
}

$bridgeOrigin = (git -C $bridgeRoot remote get-url origin).Trim()
if ($bridgeOrigin -ne 'https://github.com/Hlushok/lampa-plugin.git') {
  throw "Unexpected bridge origin: $bridgeOrigin"
}

gh auth status
if ($LASTEXITCODE -ne 0) { throw 'GitHub CLI authentication failed.' }
$login = (gh api user --jq '.login').Trim()
if ($login -ne 'Hlushok') { throw "Expected Hlushok, got $login." }
~~~

Expected: both worktrees are clean on main, the bridge origin is exact, and gh reports Hlushok.

- [ ] **Step 2: Resolve and download only the latest upstream nova_skin.js**

~~~powershell
$upstreamCommit = (
  gh api --method GET repos/amikdn/amikdn.github.io/commits -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha'
).Trim()
if ($upstreamCommit -notmatch '^[0-9a-f]{40}$') {
  throw "Invalid upstream file commit: $upstreamCommit"
}

$probePath = Join-Path ([IO.Path]::GetTempPath()) ('nova-upstream-' + [guid]::NewGuid().ToString('N') + '.js')
try {
  curl.exe --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js" --output $probePath
  if ($LASTEXITCODE -ne 0 -or -not (Test-Path -LiteralPath $probePath) -or (Get-Item -LiteralPath $probePath).Length -eq 0) {
    throw 'Upstream payload download failed or is empty.'
  }
  node --check $probePath
  if ($LASTEXITCODE -ne 0) { throw 'Upstream payload is not valid JavaScript.' }
  $upstreamDigest = (Get-FileHash -LiteralPath $probePath -Algorithm SHA256).Hash.ToLowerInvariant()
  [pscustomobject]@{ Commit = $upstreamCommit; SHA256 = $upstreamDigest; Bytes = (Get-Item $probePath).Length } | Format-List
}
finally {
  if (Test-Path -LiteralPath $probePath) { Remove-Item -LiteralPath $probePath -Force }
}
~~~

Expected: an immutable 40-character SHA, a non-empty file, successful node --check, and a 64-character SHA-256.

- [ ] **Step 3: Run the explicit upstream hook compatibility gate**

~~~powershell
$upstreamCommit = (
  gh api --method GET repos/amikdn/amikdn.github.io/commits -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha'
).Trim()
$probePath = Join-Path ([IO.Path]::GetTempPath()) ('nova-contract-' + [guid]::NewGuid().ToString('N') + '.js')
try {
  curl.exe --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js" --output $probePath
  if ($LASTEXITCODE -ne 0) { throw 'Contract payload download failed.' }
  $source = Get-Content -LiteralPath $probePath -Raw -Encoding utf8
  $required = @(
    "typeof window.nova_skin_probe_mode === 'function'",
    "mode === 'external' || mode === 'disabled' || mode === 'legacy'",
    "return probeMode() === 'legacy' && probeOn()",
    "if (!probeAllowed() || !probe_url || !movie) return"
  )
  foreach ($literal in $required) {
    if (-not $source.Contains($literal)) { throw "Nova hook contract missing: $literal" }
  }
  'Nova external hook contract is compatible.'
}
finally {
  if (Test-Path -LiteralPath $probePath) { Remove-Item -LiteralPath $probePath -Force }
}
~~~

Expected: all four compatibility literals are present. If any fails, stop the pilot; do not patch the mirror or bridge around it.

- [ ] **Step 4: Branch safely for absent or already-created mirror repository**

~~~powershell
$targetProbe = gh api repos/Hlushok/nova-skin 2>&1
$targetExit = $LASTEXITCODE

if ($targetExit -eq 0) {
  $target = $targetProbe | ConvertFrom-Json
  if (-not $target.fork -or $target.parent.full_name -ne 'amikdn/amikdn.github.io' -or $target.private -or $target.default_branch -ne 'main') {
    throw 'Existing Hlushok/nova-skin is not the required public native fork.'
  }
  'Compatible target repository already exists; continue only with fast-forward reconciliation.'
}
elseif ($targetProbe -match 'HTTP 404') {
  'Target repository is absent and may be created in Task 2.'
}
else {
  throw "Unexpected target probe result: $targetProbe"
}
~~~

Expected: either a clean 404 or exact public-fork metadata. Never delete or replace an unexpected repository.

- [ ] **Step 5: Commit no files**

This is a read-only task. Confirm both local worktrees remain clean.

---

### Task 2: Create and construct the minimal native mirror

**Files:**

- Create: D:/opt/nova-skin/nova_skin.js
- Create: D:/opt/nova-skin/README.md
- Create: D:/opt/nova-skin/.nojekyll
- Preserve: all four reviewed documents listed in the File Map

**Interfaces:**

- Consumes: Task 1 baseline and Hlushok GitHub account.
- Produces: a local main branch descending from the native fork baseline with an exact seven-blob foundation tree; Task 3 adds the eighth blob.

- [ ] **Step 1: Create the custom-named fork only when Task 1 proved it absent**

~~~powershell
$targetProbe = gh api repos/Hlushok/nova-skin 2>&1
if ($LASTEXITCODE -ne 0 -and $targetProbe -match 'HTTP 404') {
  gh api --method POST repos/amikdn/amikdn.github.io/forks -f name='nova-skin' -F default_branch_only=true --jq '{full_name,private,fork,default_branch}'
  if ($LASTEXITCODE -ne 0) { throw 'Fork creation failed.' }
}

$fork = $null
for ($attempt = 1; $attempt -le 12; $attempt++) {
  $json = gh api repos/Hlushok/nova-skin 2>$null
  if ($LASTEXITCODE -eq 0) { $fork = $json | ConvertFrom-Json; break }
  Start-Sleep -Seconds 5
}
if ($null -eq $fork) { throw 'Fork did not materialize within 60 seconds.' }
if (-not $fork.fork -or $fork.parent.full_name -ne 'amikdn/amikdn.github.io' -or $fork.private) {
  throw 'Fork metadata is incompatible.'
}
~~~

Expected: Hlushok/nova-skin is a public native fork of amikdn/amikdn.github.io.

- [ ] **Step 2: Attach origin and merge native fork ancestry without resetting local history**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$expectedOrigin = 'https://github.com/Hlushok/nova-skin.git'
$origin = git -C $repoRoot remote get-url origin 2>$null
if ($LASTEXITCODE -eq 0 -and $origin.Trim() -ne $expectedOrigin) {
  throw "Unexpected mirror origin: $origin"
}
if ($LASTEXITCODE -ne 0) {
  git -C $repoRoot remote add origin $expectedOrigin
  if ($LASTEXITCODE -ne 0) { throw 'Failed to add mirror origin.' }
}

git -C $repoRoot fetch --no-tags origin main
if ($LASTEXITCODE -ne 0) { throw 'Failed to fetch mirror origin/main.' }

git -C $repoRoot merge-base --is-ancestor origin/main HEAD
if ($LASTEXITCODE -ne 0) {
  git -C $repoRoot merge --allow-unrelated-histories --no-commit --no-ff origin/main
  if ($LASTEXITCODE -ne 0) {
    git -C $repoRoot status --short
    throw 'Unexpected native-fork merge conflict.'
  }
}
~~~

Expected: local work is either already descended from origin/main or a clean uncommitted merge is in progress.

- [ ] **Step 3: Remove inherited upstream files through an exact path allowlist**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$keep = [Collections.Generic.HashSet[string]]::new([StringComparer]::Ordinal)
@(
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/plans/2026-08-23-nova-skin-premium-probe-pilot-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'docs/superpowers/specs/2026-08-23-nova-skin-premium-probe-pilot-design.md',
  'nova_skin.js'
) | ForEach-Object { [void]$keep.Add($_) }

$repoPrefix = $repoRoot.TrimEnd([IO.Path]::DirectorySeparatorChar) + [IO.Path]::DirectorySeparatorChar
$tracked = @(git -C $repoRoot ls-files)
foreach ($relative in @($tracked | Where-Object { -not $keep.Contains($_) })) {
  if ([IO.Path]::IsPathRooted($relative) -or ($relative -split '/') -contains '..') {
    throw "Unsafe tracked path: $relative"
  }
  $resolved = [IO.Path]::GetFullPath((Join-Path $repoRoot $relative.Replace('/', [IO.Path]::DirectorySeparatorChar)))
  if (-not $resolved.StartsWith($repoPrefix, [StringComparison]::OrdinalIgnoreCase)) {
    throw "Path escapes repository: $relative"
  }
  git -C $repoRoot rm -f -- $relative
  if ($LASTEXITCODE -ne 0) { throw "git rm failed: $relative" }
}
~~~

Expected: only the upstream nova_skin.js and four reviewed documents remain tracked before README and .nojekyll are added.

- [ ] **Step 4: Create README.md and the empty .nojekyll marker with apply_patch**

README.md must contain:

~~~markdown
# Nova Skin mirror

Автоматичне byte-for-byte дзеркало плагіна Nova Skin для Lampa.

## Підключення

https://hlushok.github.io/nova-skin/nova_skin.js

## Походження

- Авторське джерело: https://github.com/amikdn/amikdn.github.io
- Оригінальний файл: https://github.com/amikdn/amikdn.github.io/blob/main/nova_skin.js
- Це GitHub-форк і дзеркало, а не переписаний плагін.

У перевіреному upstream-репозиторії не знайдено файла ліцензії. Цей форк не
додає власної ліцензії та не заявляє авторство на upstream-код.

## Автоматичне оновлення

Workflow перевіряє upstream щогодини на 17-й хвилині UTC та підтримує ручний
запуск. Він завантажує лише nova_skin.js з зафіксованого commit SHA, перевіряє
доставку і JavaScript-синтаксис, але не виконує та не аудитить поведінку коду.

GitHub може вимкнути cron у неактивному публічному репозиторії. У такому разі
workflow треба знову ввімкнути в Actions.

## Межі

Інші upstream-файли не переносяться. Lampac bridge зберігається окремо в
Hlushok/lampa-plugin. Автоматичне встановлення у Lampa або розгортання на VPS
не входить до цього репозиторію.
~~~

Create .nojekyll as a tracked empty or newline-only marker. Do not create
LICENSE, index.html, a package manifest, or another plugin.

- [ ] **Step 5: Replace nova_skin.js with immutable upstream bytes and commit the foundation**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$upstreamCommit = (
  gh api --method GET repos/amikdn/amikdn.github.io/commits -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha'
).Trim()
if ($upstreamCommit -notmatch '^[0-9a-f]{40}$') { throw 'Invalid upstream SHA.' }

$downloadPath = Join-Path ([IO.Path]::GetTempPath()) ('nova-mirror-' + [guid]::NewGuid().ToString('N') + '.js')
try {
  curl.exe --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js" --output $downloadPath
  if ($LASTEXITCODE -ne 0 -or (Get-Item -LiteralPath $downloadPath).Length -eq 0) { throw 'Mirror download failed.' }
  node --check $downloadPath
  if ($LASTEXITCODE -ne 0) { throw 'Mirror payload syntax failed.' }
  $digest = (Get-FileHash -LiteralPath $downloadPath -Algorithm SHA256).Hash.ToLowerInvariant()
  Move-Item -LiteralPath $downloadPath -Destination (Join-Path $repoRoot 'nova_skin.js') -Force
  $downloadPath = $null

  git -C $repoRoot add -- .nojekyll README.md nova_skin.js docs/superpowers
  git -C $repoRoot diff --cached --check -- . ':(exclude)nova_skin.js'
  if ($LASTEXITCODE -ne 0) { throw 'Foundation diff check failed.' }
  git -C $repoRoot commit -m 'feat: create minimal Nova Skin mirror' -m "Upstream-Commit: $upstreamCommit" -m "SHA256: $digest"
  if ($LASTEXITCODE -ne 0) { throw 'Foundation commit failed.' }
}
finally {
  if ($null -ne $downloadPath -and (Test-Path -LiteralPath $downloadPath)) {
    Remove-Item -LiteralPath $downloadPath -Force
  }
}
~~~

Expected: one commit records the upstream SHA and SHA-256, and the worktree is clean.

- [ ] **Step 6: Prove ancestry and the seven-blob foundation allowlist**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot merge-base --is-ancestor origin/main HEAD
if ($LASTEXITCODE -ne 0) { throw 'Mirror HEAD does not descend from native fork main.' }

$expected = @(
  '.nojekyll',
  'README.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/plans/2026-08-23-nova-skin-premium-probe-pilot-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'docs/superpowers/specs/2026-08-23-nova-skin-premium-probe-pilot-design.md',
  'nova_skin.js'
) | Sort-Object
$actual = @(git -C $repoRoot ls-tree -r --name-only HEAD) | Sort-Object
if (Compare-Object $expected $actual) { throw 'Foundation tree differs from seven-blob allowlist.' }
node --check (Join-Path $repoRoot 'nova_skin.js')
if ($LASTEXITCODE -ne 0) { throw 'Committed Nova payload syntax failed.' }
if (git -C $repoRoot status --porcelain=v1) { throw 'Mirror worktree is dirty.' }
~~~

Expected: exact seven-blob tree, valid JavaScript, correct ancestry, and clean status.

---

### Task 3: Add the trusted hourly sync and mirror Pages workflow

**Files:**

- Create: D:/opt/nova-skin/.github/workflows/sync-upstream.yml
- Verify: D:/opt/nova-skin/nova_skin.js

**Interfaces:**

- Consumes: public GitHub API, immutable raw upstream URLs, repository GITHUB_TOKEN.
- Produces: changed=true/false output, fast-forward bot commits for changed bytes, and a Pages artifact with exactly two files.

- [ ] **Step 1: Confirm the workflow is absent**

~~~powershell
$workflowPath = 'D:/opt/nova-skin/.github/workflows/sync-upstream.yml'
if (Test-Path -LiteralPath $workflowPath) {
  throw 'Workflow already exists unexpectedly; inspect before continuing.'
}
'Expected red check: workflow is absent.'
~~~

- [ ] **Step 2: Create sync-upstream.yml with apply_patch**

Use this exact workflow:

~~~yaml
name: Sync Nova Skin

on:
  workflow_dispatch:
  schedule:
    - cron: "17 * * * *"

permissions:
  contents: read

concurrency:
  group: nova-skin-upstream-sync
  cancel-in-progress: false

jobs:
  sync:
    name: Sync and package
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: write
      pages: read
    steps:
      - name: Checkout downstream
        uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803
        with:
          fetch-depth: 0

      - name: Resolve and validate immutable upstream file
        id: upstream
        shell: bash
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          set -euo pipefail
          commit_sha="$(gh api --method GET repos/amikdn/amikdn.github.io/commits -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha')"
          [[ "$commit_sha" =~ ^[0-9a-f]{40}$ ]]
          download="$RUNNER_TEMP/nova_skin-$commit_sha.js"
          curl --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$commit_sha/nova_skin.js" --output "$download"
          test -s "$download"
          node --check "$download"
          digest="$(sha256sum "$download" | cut -d' ' -f1)"
          printf 'commit_sha=%s\n' "$commit_sha" >> "$GITHUB_OUTPUT"
          printf 'sha256=%s\n' "$digest" >> "$GITHUB_OUTPUT"
          printf 'download_path=%s\n' "$download" >> "$GITHUB_OUTPUT"

      - name: Commit changed upstream bytes
        shell: bash
        env:
          DOWNLOAD_PATH: ${{ steps.upstream.outputs.download_path }}
          UPSTREAM_COMMIT: ${{ steps.upstream.outputs.commit_sha }}
          UPSTREAM_SHA256: ${{ steps.upstream.outputs.sha256 }}
        run: |
          set -euo pipefail
          if cmp -s "$DOWNLOAD_PATH" nova_skin.js; then
            echo "nova_skin.js already matches upstream $UPSTREAM_COMMIT"
            exit 0
          fi
          install -m 0644 "$DOWNLOAD_PATH" nova_skin.js.next
          mv -f nova_skin.js.next nova_skin.js
          git config user.name 'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
          git add -- nova_skin.js
          short_sha="${UPSTREAM_COMMIT:0:12}"
          git commit -m "chore: sync nova_skin.js from upstream $short_sha" -m "Upstream-Commit: $UPSTREAM_COMMIT" -m "SHA256: $UPSTREAM_SHA256"
          git push origin HEAD:main

      - name: Configure Pages
        uses: actions/configure-pages@983d7736d9b0ae728b81ab479565c72886d7745b

      - name: Prepare clean Pages artifact
        shell: bash
        run: |
          set -euo pipefail
          test ! -e _site
          install -d -m 0755 _site
          install -m 0644 nova_skin.js _site/nova_skin.js
          install -m 0644 .nojekyll _site/.nojekyll
          test "$(find _site -type f | wc -l)" -eq 2

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@7b1f4a764d45c48632c6b24a0339c27f5614fb0b
        with:
          path: _site

  deploy:
    name: Deploy Pages
    needs: sync
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy Pages artifact
        id: deployment
        uses: actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e
~~~

- [ ] **Step 3: Parse the workflow and enforce immutable pins and triggers**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$workflowPath = Join-Path $repoRoot '.github/workflows/sync-upstream.yml'
npx.cmd --yes prettier@3.6.2 --check $workflowPath
if ($LASTEXITCODE -ne 0) { throw 'Workflow YAML parse/format check failed.' }

$workflowText = Get-Content -LiteralPath $workflowPath -Raw -Encoding utf8
$pins = [regex]::Matches($workflowText, 'uses:\s+([A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+)@([0-9a-f]{40})')
if ($pins.Count -ne 4) { throw "Expected four immutable Action pins, found $($pins.Count)." }
if ($workflowText -match 'uses:\s+\S+@(v\d+|main|master)(\s|$)') { throw 'Mutable Action reference remains.' }
if ($workflowText -notmatch 'cron:\s+"17 \* \* \* \*"' -or $workflowText -notmatch 'workflow_dispatch:') {
  throw 'Required trigger is missing.'
}
git -C $repoRoot diff --check
node --check (Join-Path $repoRoot 'nova_skin.js')
~~~

Expected: YAML parses, all four actions are pinned, both triggers exist, and local checks pass.

- [ ] **Step 4: Commit the workflow and verify the final eight-blob local tree**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot add -- .github/workflows/sync-upstream.yml
git -C $repoRoot diff --cached --check
if ($LASTEXITCODE -ne 0) { throw 'Workflow staging failed.' }
git -C $repoRoot commit -m 'ci: mirror Nova Skin and deploy Pages'
if ($LASTEXITCODE -ne 0) { throw 'Workflow commit failed.' }

$expected = @(
  '.github/workflows/sync-upstream.yml',
  '.nojekyll',
  'README.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/plans/2026-08-23-nova-skin-premium-probe-pilot-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'docs/superpowers/specs/2026-08-23-nova-skin-premium-probe-pilot-design.md',
  'nova_skin.js'
) | Sort-Object
$actual = @(git -C $repoRoot ls-tree -r --name-only HEAD) | Sort-Object
if (Compare-Object $expected $actual) { throw 'Final local tree differs from eight-blob allowlist.' }
if (git -C $repoRoot status --porcelain=v1) { throw 'Mirror worktree is dirty.' }
~~~

Expected: the workflow commit exists and the local repository contains exactly eight blobs.

---

### Task 4: Publish and prove the byte-for-byte mirror

**Files:**

- Publish: the exact eight-blob mirror repository tree
- Publish as Pages artifact: nova_skin.js and .nojekyll only

**Interfaces:**

- Consumes: clean local mirror main and unchanged remote baseline.
- Produces: public native fork, active workflow, HTTPS Pages URL, successful deployment, no-op proof, and three matching SHA-256 values.

- [ ] **Step 1: Run pre-push fast-forward, diff, and secret checks**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
if (git -C $repoRoot status --porcelain=v1) { throw 'Mirror is dirty before push.' }
$remoteMain = (git -C $repoRoot ls-remote origin refs/heads/main).Split()[0]
if ($remoteMain -notmatch '^[0-9a-f]{40}$') { throw 'Could not resolve remote main.' }
git -C $repoRoot merge-base --is-ancestor $remoteMain HEAD
if ($LASTEXITCODE -ne 0) { throw 'Mirror push is not fast-forward.' }
git -C $repoRoot diff --check origin/main..HEAD -- . ':(exclude)nova_skin.js'
if ($LASTEXITCODE -ne 0) { throw 'Mirror pre-push diff check failed.' }

$secretMatches = rg -n --hidden -g '!.git/**' '(ghp_[A-Za-z0-9]+|github_pat_[A-Za-z0-9_]+|AKIA[0-9A-Z]{16}|BEGIN (RSA |OPENSSH |EC )?PRIVATE KEY|Authorization:\s*Bearer\s+\S+)' $repoRoot
if ($LASTEXITCODE -eq 0) { $secretMatches; throw 'Secret-like material found.' }
if ($LASTEXITCODE -gt 1) { throw 'Secret scan failed.' }
~~~

Expected: a fast-forward push, clean diff, and zero secret-like matches.

- [ ] **Step 2: Push without force and verify exact public tree**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot push --set-upstream origin main
if ($LASTEXITCODE -ne 0) { throw 'Mirror push failed.' }

$expected = @(
  '.github/workflows/sync-upstream.yml',
  '.nojekyll',
  'README.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/plans/2026-08-23-nova-skin-premium-probe-pilot-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'docs/superpowers/specs/2026-08-23-nova-skin-premium-probe-pilot-design.md',
  'nova_skin.js'
) | Sort-Object
$actual = @(gh api 'repos/Hlushok/nova-skin/git/trees/main?recursive=1' --jq '.tree[] | select(.type == "blob") | .path') | Sort-Object
if (Compare-Object $expected $actual) { throw 'Public mirror tree differs from eight-blob allowlist.' }
~~~

Expected: remote main equals local HEAD and exactly eight blobs are public.

- [ ] **Step 3: Enable the workflow and configure workflow-based Pages**

~~~powershell
gh workflow enable sync-upstream.yml --repo Hlushok/nova-skin
if ($LASTEXITCODE -ne 0) { throw 'Could not enable mirror workflow.' }

$pagesProbe = gh api repos/Hlushok/nova-skin/pages 2>&1
if ($LASTEXITCODE -eq 0) {
  gh api --method PUT repos/Hlushok/nova-skin/pages -f build_type=workflow
}
elseif ($pagesProbe -match 'HTTP 404') {
  gh api --method POST repos/Hlushok/nova-skin/pages -f build_type=workflow
}
else {
  throw "Unexpected Pages probe: $pagesProbe"
}
if ($LASTEXITCODE -ne 0) { throw 'Pages configuration failed.' }

$pages = gh api repos/Hlushok/nova-skin/pages | ConvertFrom-Json
if ($pages.build_type -ne 'workflow') { throw 'Pages is not workflow-based.' }
~~~

Expected: workflow state active and Pages build_type workflow.

- [ ] **Step 4: Dispatch and monitor the first deployment**

~~~powershell
gh workflow run sync-upstream.yml --repo Hlushok/nova-skin --ref main
if ($LASTEXITCODE -ne 0) { throw 'Mirror dispatch failed.' }
Start-Sleep -Seconds 3
$runId = (gh run list --repo Hlushok/nova-skin --workflow sync-upstream.yml --event workflow_dispatch --limit 1 --json databaseId --jq '.[0].databaseId').Trim()
if ($runId -notmatch '^\d+$') { throw "Invalid workflow run id: $runId" }
gh run watch $runId --repo Hlushok/nova-skin --exit-status --interval 5
if ($LASTEXITCODE -ne 0) {
  gh run view $runId --repo Hlushok/nova-skin --log-failed
  throw 'First mirror workflow failed.'
}
~~~

Monitor through bounded command polling and report progress at least once per minute. Expected: both sync and deploy jobs succeed.

- [ ] **Step 5: Dispatch one controlled no-change run**

~~~powershell
$headBefore = (gh api repos/Hlushok/nova-skin/commits/main --jq '.sha').Trim()
gh workflow run sync-upstream.yml --repo Hlushok/nova-skin --ref main
if ($LASTEXITCODE -ne 0) { throw 'No-op dispatch failed.' }
Start-Sleep -Seconds 3
$noOpRun = (gh run list --repo Hlushok/nova-skin --workflow sync-upstream.yml --event workflow_dispatch --limit 1 --json databaseId --jq '.[0].databaseId').Trim()
gh run watch $noOpRun --repo Hlushok/nova-skin --exit-status --interval 5
if ($LASTEXITCODE -ne 0) { throw 'No-op workflow failed.' }
$headAfter = (gh api repos/Hlushok/nova-skin/commits/main --jq '.sha').Trim()
$log = gh run view $noOpRun --repo Hlushok/nova-skin --log
if ($log -notmatch 'nova_skin\.js already matches upstream [0-9a-f]{40}') {
  throw 'No-op marker missing; upstream may have changed. Rebaseline and repeat.'
}
if ($headBefore -ne $headAfter) {
  throw 'Downstream main changed during the intended no-op run.'
}
~~~

Expected: the second run deploys successfully without creating a commit.

- [ ] **Step 6: Compare immutable upstream, remote main, and Pages bytes**

~~~powershell
$payloadCommit = gh api --method GET repos/Hlushok/nova-skin/commits -f sha=main -f path=nova_skin.js -F per_page=1 | ConvertFrom-Json
$recorded = [regex]::Match($payloadCommit[0].commit.message, 'Upstream-Commit: ([0-9a-f]{40})')
if (-not $recorded.Success) { throw 'Latest payload commit lacks upstream provenance.' }
$upstreamCommit = $recorded.Groups[1].Value
$downstreamHead = (gh api repos/Hlushok/nova-skin/commits/main --jq '.sha').Trim()

$tempRoot = Join-Path ([IO.Path]::GetTempPath()) ('nova-publish-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -LiteralPath $tempRoot | Out-Null
$upstreamPath = Join-Path $tempRoot 'upstream.js'
$downstreamPath = Join-Path $tempRoot 'downstream.js'
$publicPath = Join-Path $tempRoot 'public.js'
try {
  curl.exe --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js" --output $upstreamPath
  curl.exe --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/Hlushok/nova-skin/$downstreamHead/nova_skin.js" --output $downstreamPath
  $response = Invoke-WebRequest -Uri "https://hlushok.github.io/nova-skin/nova_skin.js?rev=$upstreamCommit&ts=$([DateTimeOffset]::UtcNow.ToUnixTimeSeconds())" -Headers @{ 'Cache-Control' = 'no-cache' } -OutFile $publicPath -PassThru
  if ($response.StatusCode -ne 200) { throw "Mirror Pages HTTP $($response.StatusCode)." }

  $hashes = @(
    (Get-FileHash -LiteralPath $upstreamPath -Algorithm SHA256).Hash.ToLowerInvariant(),
    (Get-FileHash -LiteralPath $downstreamPath -Algorithm SHA256).Hash.ToLowerInvariant(),
    (Get-FileHash -LiteralPath $publicPath -Algorithm SHA256).Hash.ToLowerInvariant()
  )
  if (($hashes | Select-Object -Unique).Count -ne 1) { throw 'Upstream, downstream, and Pages bytes differ.' }
  node --check $publicPath
  if ($LASTEXITCODE -ne 0) { throw 'Public mirror syntax failed.' }
  [pscustomobject]@{ UpstreamCommit=$upstreamCommit; DownstreamHead=$downstreamHead; SHA256=$hashes[0]; HTTP=$response.StatusCode } | Format-List
}
finally {
  $tempBase = [IO.Path]::GetFullPath([IO.Path]::GetTempPath())
  $resolvedTemp = [IO.Path]::GetFullPath($tempRoot)
  if ((Test-Path -LiteralPath $resolvedTemp) -and $resolvedTemp.StartsWith($tempBase, [StringComparison]::OrdinalIgnoreCase)) {
    Remove-Item -LiteralPath $resolvedTemp -Recurse -Force
  }
}
~~~

Expected: HTTP 200, one SHA-256 across all three files, and valid public JavaScript. If Pages propagation lags, retry only this step in bounded passes.

---

### Task 5: Build the bridge test-first without adding test files

**Files:**

- Create: D:/opt/lampac/lampa-plugin/nova_skin_lampac.js
- Modify: D:/opt/lampac/lampa-plugin/README.md

**Interfaces:**

- Consumes: window.Lampa.Utils.putScriptAsync(items, complete, error, success, show_logs).
- Produces: window.nova_skin_lampac_loader, window.nova_skin_probe_mode returning external, and at most one mirror load request per page.

- [ ] **Step 1: Run the inline behavioral harness and confirm the missing-file failure**

Run this exact PowerShell block before creating the bridge:

~~~powershell
$bridgePath = 'D:/opt/lampac/lampa-plugin/nova_skin_lampac.js'
$env:NOVA_BRIDGE_PATH = $bridgePath
$harness = @'
const assert = require('node:assert/strict');
const fs = require('node:fs');
const test = require('node:test');
const vm = require('node:vm');

const sourcePath = process.env.NOVA_BRIDGE_PATH;
assert.ok(fs.existsSync(sourcePath), 'nova_skin_lampac.js must exist');
const source = fs.readFileSync(sourcePath, 'utf8');

function harness(options = {}) {
  const calls = [];
  const errors = [];
  const window = {};
  const consoleStub = {
    error(...args) {
      errors.push(args);
    }
  };

  if (options.preloaded) window.nova_skin = true;
  if (!options.missingApi) {
    window.Lampa = {
      Utils: {
        putScriptAsync(items, complete, error, success, showLogs) {
          calls.push({
            items,
            complete,
            error,
            success,
            showLogs,
            modeAtLoad: window.nova_skin_probe_mode()
          });
          if (options.throwSync) throw new Error('synthetic synchronous failure');
          if (options.failLoad) error(items[0]);
        }
      }
    };
  }

  window.console = consoleStub;
  const context = vm.createContext({
    window,
    console: consoleStub,
    Date: { now: () => 500 * 3600000 + 1234 }
  });

  return {
    calls,
    errors,
    window,
    execute() {
      vm.runInContext(source, context, { filename: sourcePath });
    }
  };
}

test('registers external before one exact hourly mirror load', () => {
  const h = harness();
  h.execute();
  assert.equal(h.window.nova_skin_probe_mode(), 'external');
  assert.equal(h.calls.length, 1);
  assert.equal(h.calls[0].modeAtLoad, 'external');
  assert.deepEqual(
    Array.from(h.calls[0].items),
    ['https://hlushok.github.io/nova-skin/nova_skin.js?v=500']
  );
  assert.equal(h.calls[0].complete, false);
  assert.equal(typeof h.calls[0].error, 'function');
  assert.equal(h.calls[0].success, false);
  assert.equal(h.calls[0].showLogs, false);
});

test('second execution is a no-op', () => {
  const h = harness();
  h.execute();
  h.execute();
  assert.equal(h.calls.length, 1);
  assert.equal(h.errors.length, 0);
});

test('preloaded Nova is not requested again but external hook is installed', () => {
  const h = harness({ preloaded: true });
  h.execute();
  assert.equal(h.window.nova_skin_probe_mode(), 'external');
  assert.equal(h.calls.length, 0);
});

test('missing Lampa API logs once and exits', () => {
  const h = harness({ missingApi: true });
  h.execute();
  assert.equal(h.calls.length, 0);
  assert.equal(h.errors.length, 1);
});

test('load failure logs once and does not retry', () => {
  const h = harness({ failLoad: true });
  h.execute();
  assert.equal(h.calls.length, 1);
  assert.equal(h.errors.length, 1);
});

test('synchronous loader failure logs once and does not escape', () => {
  const h = harness({ throwSync: true });
  assert.doesNotThrow(() => h.execute());
  assert.equal(h.calls.length, 1);
  assert.equal(h.errors.length, 1);
});
'@

$harness | node -
$exit = $LASTEXITCODE
Remove-Item Env:NOVA_BRIDGE_PATH
if ($exit -eq 0) { throw 'Harness unexpectedly passed before bridge creation.' }
'Expected red check: nova_skin_lampac.js must exist.'
~~~

Expected: Node exits nonzero only because nova_skin_lampac.js does not yet exist.

- [ ] **Step 2: Create nova_skin_lampac.js with apply_patch**

Use exactly:

~~~javascript
(function () {
  'use strict';

  var marker = 'nova_skin_lampac_loader';
  var mirror = 'https://hlushok.github.io/nova-skin/nova_skin.js';

  function report(message, error) {
    try {
      if (window.console && typeof window.console.error === 'function') {
        if (error) window.console.error(message, error);
        else window.console.error(message);
      }
    } catch (e) {}
  }

  if (window[marker]) return;
  window[marker] = true;

  window.nova_skin_probe_mode = function () {
    return 'external';
  };

  if (window.nova_skin) return;

  if (
    !window.Lampa ||
    !window.Lampa.Utils ||
    typeof window.Lampa.Utils.putScriptAsync !== 'function'
  ) {
    report('[Nova Skin Lampac] Lampa script loader is unavailable.');
    return;
  }

  var hour = Math.floor(Date.now() / 3600000);
  var url = mirror + '?v=' + hour;

  try {
    window.Lampa.Utils.putScriptAsync(
      [url],
      false,
      function () {
        report('[Nova Skin Lampac] Nova Skin failed to load.');
      },
      false,
      false
    );
  } catch (error) {
    report('[Nova Skin Lampac] Nova Skin request failed.', error);
  }
})();
~~~

Do not add a success notification, retry, fallback origin, storage write, or account check.

- [ ] **Step 3: Re-run the same inline harness and require six passing tests**

Repeat Task 5 Step 1 without its final expected-failure wrapper:

~~~powershell
$env:NOVA_BRIDGE_PATH = 'D:/opt/lampac/lampa-plugin/nova_skin_lampac.js'
# Set $harness to the exact here-string from Task 5 Step 1.
$harness | node -
$exit = $LASTEXITCODE
Remove-Item Env:NOVA_BRIDGE_PATH
if ($exit -ne 0) { throw 'Bridge behavioral harness failed.' }
~~~

Expected: six tests pass, zero fail.

- [ ] **Step 4: Run syntax and static contract gates**

~~~powershell
$bridgePath = 'D:/opt/lampac/lampa-plugin/nova_skin_lampac.js'
node --check $bridgePath
if ($LASTEXITCODE -ne 0) { throw 'Bridge JavaScript syntax failed.' }

$source = Get-Content -LiteralPath $bridgePath -Raw -Encoding utf8
$mirrorLiteral = 'https://hlushok.github.io/nova-skin/nova_skin.js'
if (($source.Split($mirrorLiteral).Count - 1) -ne 1) { throw 'Mirror URL must occur exactly once.' }
if ([regex]::Matches($source, 'https://').Count -ne 1) { throw 'Bridge must contain exactly one HTTPS origin.' }

$hookIndex = $source.IndexOf('window.nova_skin_probe_mode')
$loadIndex = $source.IndexOf('window.Lampa.Utils.putScriptAsync')
if ($hookIndex -lt 0 -or $loadIndex -lt 0 -or $hookIndex -gt $loadIndex) {
  throw 'External hook is not assigned before the load call.'
}

$forbidden = @(
  'eval(',
  'new Function',
  'setTimeout',
  'setInterval',
  'XMLHttpRequest',
  'fetch(',
  'window.location',
  'location.search',
  'requestInfo',
  'user_group',
  'access_code',
  'account_email',
  'password',
  'token'
)
foreach ($literal in $forbidden) {
  if ($source.IndexOf($literal, [StringComparison]::OrdinalIgnoreCase) -ge 0) {
    throw "Forbidden bridge behavior found: $literal"
  }
}
if ($source -notmatch "return\s+'external'") { throw 'Bridge does not return external.' }
if ($source -notmatch "Math\.floor\(Date\.now\(\)\s*/\s*3600000\)") { throw 'Hourly cache bucket missing.' }
~~~

Expected: syntax passes, one remote URL exists, hook ordering is correct, and no forbidden behavior is present.

- [ ] **Step 5: Add the concise README entry with apply_patch**

Keep the existing title and description, then append:

~~~markdown
## Nova Skin + Lampac — персональний пілот

URL для ручного тесту:

https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js

Loader вмикає upstream-режим external до завантаження незміненого Nova Skin.
Він не визначає Premium-групу: фонову перевірку дозволяє або відхиляє Lampac.
~~~

Do not advertise global installation and do not add the loader to presets.js.

- [ ] **Step 6: Review and commit exactly the two bridge files**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/lampac/lampa-plugin').Path
$changed = @(git -C $repoRoot status --short)
$paths = @($changed | ForEach-Object { $_.Substring(3) } | Sort-Object)
$expected = @('README.md', 'nova_skin_lampac.js') | Sort-Object
if (Compare-Object $expected $paths) {
  $changed
  throw 'Bridge worktree contains files outside the two-file scope.'
}

git -C $repoRoot diff --check
if ($LASTEXITCODE -ne 0) { throw 'Bridge diff check failed.' }
git -C $repoRoot diff -- README.md nova_skin_lampac.js

git -C $repoRoot add -- README.md nova_skin_lampac.js
git -C $repoRoot diff --cached --check
if ($LASTEXITCODE -ne 0) { throw 'Bridge staged diff check failed.' }
git -C $repoRoot commit -m 'feat: add Nova Skin Lampac pilot loader'
if ($LASTEXITCODE -ne 0) { throw 'Bridge commit failed.' }
if (git -C $repoRoot status --porcelain=v1) { throw 'Bridge worktree is dirty after commit.' }
~~~

Expected: one local commit changes only README.md and nova_skin_lampac.js.

---

### Task 6: Publish and prove the bridge Pages payload

**Files:**

- Publish: D:/opt/lampac/lampa-plugin/README.md
- Publish: D:/opt/lampac/lampa-plugin/nova_skin_lampac.js
- Preserve: every other bridge repository file and existing Pages setting

**Interfaces:**

- Consumes: clean bridge main and existing legacy Pages source main/root.
- Produces: public HTTP 200 loader whose bytes equal remote main and the local commit.

- [ ] **Step 1: Verify fast-forward safety, two-file diff, and no secrets**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/lampac/lampa-plugin').Path
git -C $repoRoot fetch --no-tags origin main
if ($LASTEXITCODE -ne 0) { throw 'Bridge fetch failed.' }
git -C $repoRoot merge-base --is-ancestor origin/main HEAD
if ($LASTEXITCODE -ne 0) { throw 'Bridge push would not be fast-forward.' }

$changed = @(git -C $repoRoot diff --name-only origin/main..HEAD) | Sort-Object
$expected = @('README.md', 'nova_skin_lampac.js') | Sort-Object
if (Compare-Object $expected $changed) { throw 'Bridge commit scope differs from two files.' }
git -C $repoRoot diff --check origin/main..HEAD
if ($LASTEXITCODE -ne 0) { throw 'Bridge pre-push diff check failed.' }

$secretMatches = rg -n --hidden -g '!.git/**' '(ghp_[A-Za-z0-9]+|github_pat_[A-Za-z0-9_]+|AKIA[0-9A-Z]{16}|BEGIN (RSA |OPENSSH |EC )?PRIVATE KEY|Authorization:\s*Bearer\s+\S+)' $repoRoot
if ($LASTEXITCODE -eq 0) { $secretMatches; throw 'Secret-like material found in bridge repository.' }
if ($LASTEXITCODE -gt 1) { throw 'Bridge secret scan failed.' }
~~~

Expected: fast-forward relationship, exactly two changed files, clean diff, and no secret-like material.

- [ ] **Step 2: Push the bridge without changing Pages configuration**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/lampac/lampa-plugin').Path
git -C $repoRoot push origin main
if ($LASTEXITCODE -ne 0) { throw 'Bridge push failed.' }

$localHead = (git -C $repoRoot rev-parse HEAD).Trim()
$remoteHead = (gh api repos/Hlushok/lampa-plugin/commits/main --jq '.sha').Trim()
if ($localHead -ne $remoteHead) { throw 'Bridge local and remote heads differ.' }

$pages = gh api repos/Hlushok/lampa-plugin/pages | ConvertFrom-Json
if ($pages.build_type -ne 'legacy' -or $pages.source.branch -ne 'main' -or $pages.source.path -ne '/' -or -not $pages.https_enforced) {
  throw 'Existing lampa-plugin Pages contract changed unexpectedly.'
}
~~~

Expected: normal push, matching heads, and unchanged legacy Pages source main/root with HTTPS.

- [ ] **Step 3: Compare local, immutable remote, and Pages loader bytes**

~~~powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/lampac/lampa-plugin').Path
$remoteHead = (gh api repos/Hlushok/lampa-plugin/commits/main --jq '.sha').Trim()
$tempRoot = Join-Path ([IO.Path]::GetTempPath()) ('nova-bridge-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -LiteralPath $tempRoot | Out-Null
$remotePath = Join-Path $tempRoot 'remote.js'
$publicPath = Join-Path $tempRoot 'public.js'
try {
  curl.exe --fail --silent --show-error --location --retry 3 "https://raw.githubusercontent.com/Hlushok/lampa-plugin/$remoteHead/nova_skin_lampac.js" --output $remotePath
  $response = Invoke-WebRequest -Uri "https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js?rev=$remoteHead&ts=$([DateTimeOffset]::UtcNow.ToUnixTimeSeconds())" -Headers @{ 'Cache-Control' = 'no-cache' } -OutFile $publicPath -PassThru
  if ($response.StatusCode -ne 200) { throw "Bridge Pages HTTP $($response.StatusCode)." }

  $localHash = (Get-FileHash -LiteralPath (Join-Path $repoRoot 'nova_skin_lampac.js') -Algorithm SHA256).Hash.ToLowerInvariant()
  $remoteHash = (Get-FileHash -LiteralPath $remotePath -Algorithm SHA256).Hash.ToLowerInvariant()
  $publicHash = (Get-FileHash -LiteralPath $publicPath -Algorithm SHA256).Hash.ToLowerInvariant()
  if ($localHash -ne $remoteHash -or $localHash -ne $publicHash) {
    throw 'Bridge local, remote, and Pages bytes differ.'
  }
  node --check $publicPath
  if ($LASTEXITCODE -ne 0) { throw 'Public bridge syntax failed.' }
  [pscustomobject]@{ Commit=$remoteHead; SHA256=$publicHash; HTTP=$response.StatusCode } | Format-List
}
finally {
  $tempBase = [IO.Path]::GetFullPath([IO.Path]::GetTempPath())
  $resolvedTemp = [IO.Path]::GetFullPath($tempRoot)
  if ((Test-Path -LiteralPath $resolvedTemp) -and $resolvedTemp.StartsWith($tempBase, [StringComparison]::OrdinalIgnoreCase)) {
    Remove-Item -LiteralPath $resolvedTemp -Recurse -Force
  }
}
~~~

Expected: HTTP 200, three identical SHA-256 values, and valid public JavaScript. Retry only the public fetch if legacy Pages is still propagating.

---

### Task 7: Revalidate the existing Premium boundary without changing Lampac

**Files:**

- Verify only: D:/opt/lampac-engine-private/Online/ModuleConf.cs
- Verify only: D:/opt/lampac-engine-private/Online/ModInit.cs
- Verify only: D:/opt/lampac-engine-private/Online/OnlineApi.cs
- Verify only: D:/opt/lampac-engine-private/ops/check-online-search-guard.sh
- Verify only: D:/opt/lampac/init.conf
- Verify only: /opt/lampac on the VPS

**Interfaces:**

- Consumes: current custom/prod source, production init.conf, active Lampac runtime, authenticated pilot session.
- Produces: evidence that group 3 is the active threshold and both initiation and lifeevents stay server-guarded.

- [ ] **Step 1: Verify the private source checkout and invariant script**

~~~powershell
$engineRoot = (Resolve-Path -LiteralPath 'D:/opt/lampac-engine-private').Path
if ((git -C $engineRoot branch --show-current).Trim() -ne 'custom/prod') {
  throw 'Private engine checkout is not on custom/prod.'
}
if (git -C $engineRoot status --porcelain=v1) {
  throw 'Private engine checkout is dirty; do not mix pilot work with it.'
}
git -C $engineRoot fetch --no-tags origin custom/prod
if ($LASTEXITCODE -ne 0) { throw 'Could not refresh custom/prod reference.' }
$localHead = (git -C $engineRoot rev-parse HEAD).Trim()
$remoteHead = (git -C $engineRoot rev-parse origin/custom/prod).Trim()
if ($localHead -ne $remoteHead) {
  throw "Private source is not current: local $localHead, remote $remoteHead."
}

& 'C:/Program Files/Git/bin/bash.exe' -lc 'cd /d/opt/lampac-engine-private && bash ops/check-online-search-guard.sh'
if ($LASTEXITCODE -ne 0) { throw 'Online search guard invariant failed.' }

$moduleConf = Get-Content -LiteralPath (Join-Path $engineRoot 'Online/ModuleConf.cs') -Raw -Encoding utf8
$modInit = Get-Content -LiteralPath (Join-Path $engineRoot 'Online/ModInit.cs') -Raw -Encoding utf8
$onlineApi = Get-Content -LiteralPath (Join-Path $engineRoot 'Online/OnlineApi.cs') -Raw -Encoding utf8
if ($moduleConf -notmatch 'checkOnlineSearchGroup\s*\{[^}]*\}\s*=\s*3') { throw 'Default group threshold is not 3.' }
if ($modInit -notmatch 'requestInfo\.user\.group\s*>=\s*conf\.checkOnlineSearchGroup') { throw 'Authenticated group check is missing.' }
if ([regex]::Matches($onlineApi, 'ModInit\.CanCheckOnlineSearch\(requestInfo\)').Count -ne 2) { throw 'Expected exactly two Online API guards.' }
~~~

Expected: current custom/prod, clean checkout, invariant script success, threshold 3, authenticated group comparison, and exactly two API guards.

- [ ] **Step 2: Verify local production-source configuration read-only**

~~~powershell
$conf = Get-Content -LiteralPath 'D:/opt/lampac/init.conf' -Raw -Encoding utf8 | ConvertFrom-Json
if ($conf.online.checkOnlineSearch -ne $true) { throw 'Local init.conf disables background search.' }
if ([int]$conf.online.checkOnlineSearchGroup -ne 3) { throw 'Local init.conf threshold is not 3.' }
[pscustomobject]@{
  checkOnlineSearch = $conf.online.checkOnlineSearch
  checkOnlineSearchGroup = $conf.online.checkOnlineSearchGroup
} | Format-List
~~~

Expected: true and 3. Do not write init.conf or current.conf.

- [ ] **Step 3: Verify the live checkout, guard, config lines, and service read-only**

~~~powershell
$live = & 'C:/Program Files/Git/bin/bash.exe' -lc 'ssh lampaua "set -eu; cd /opt/lampac; systemctl is-active lampac; git branch --show-current; git rev-parse HEAD; bash ops/check-online-search-guard.sh; grep -n checkOnlineSearch init.conf"'
if ($LASTEXITCODE -ne 0) { throw 'Live Lampac verification command failed.' }
$liveText = $live -join "\n"
$live
if ($liveText -notmatch '(?m)^active$') { throw 'Lampac service is not active.' }
if ($liveText -notmatch '(?m)^custom/prod$') { throw 'Live checkout is not custom/prod.' }
if ($liveText -notmatch 'Online search guard invariant is present') { throw 'Live guard invariant did not pass.' }
if ($liveText -notmatch '"checkOnlineSearch"\s*:\s*true') { throw 'Live checkOnlineSearch is not true.' }
if ($liveText -notmatch '"checkOnlineSearchGroup"\s*:\s*3') { throw 'Live threshold is not 3.' }
~~~

Expected: active service, custom/prod, guard invariant present, checkOnlineSearch true, threshold 3. This command must not restart or edit anything.

- [ ] **Step 4: Verify the pilot account group through its authenticated browser session**

In the same Lampa session that will receive the pilot:

1. Take one existing authenticated Lampac request from browser Network tools.
2. Preserve its authentication parameters privately, but change only the path to /lite/groupdeny and add name=nova-pilot plus required_group=3.
3. Inspect the JSON response locally.
4. Require user_group equal to 3 and required_group equal to 3.
5. Record only the boolean result and timestamp in handoff evidence. Do not copy the full URL, account identifier, user_name, token, or response into the repository or chat.

Expected: the real server response for the active test session reports group 3. If the group is absent or lower, stop; do not emulate Premium in JavaScript.

- [ ] **Step 5: Confirm no Lampac mutation occurred**

Re-run clean-status checks in D:/opt/lampac-engine-private and compare the live Git SHA and init.conf hashes before/after the read-only inspection. Expected: no changes and no service restart.

---

### Task 8: Hand off the personal URL and run the manual Premium pilot

**Files:**

- No repository changes
- No VPS changes
- User-local Lampa extension list changes only

**Interfaces:**

- Consumes: both verified public URLs and an authenticated group-3 Lampa session.
- Produces: user confirmation of normal Nova behavior plus network evidence that Nova performs no client fan-out.

- [ ] **Step 1: Run the automated acceptance gate before sharing the URL**

Require all of these to be green:

- mirror repository metadata is public native fork with exact parent;
- mirror public tree has exactly eight allowed blobs;
- mirror workflow and no-op runs succeeded;
- upstream, mirror main, and mirror Pages SHA-256 are identical;
- current upstream still satisfies the accepted external hook contract;
- bridge syntax, static gates, and all six behavior tests pass;
- bridge local, remote main, and Pages SHA-256 are identical;
- lampa-plugin Pages remains main/root with HTTPS;
- private and live Lampac guard checks pass with threshold 3;
- the authenticated pilot session reports group 3;
- both worktrees are clean;
- no autoload, Lampac config, VPS file, blacklist, or service changed.

If any item is not proven, do not describe the pilot as ready.

- [ ] **Step 2: Give the user only this install URL**

https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js

Also provide the mirror URL as provenance, not as a second extension to install:

https://hlushok.github.io/nova-skin/nova_skin.js

- [ ] **Step 3: Prepare one clean Lampa profile**

The user performs these actions:

1. Remove or disable any separately installed author Nova Skin URL.
2. Add only the bridge URL through Lampa Extensions.
3. Fully close and restart Lampa.
4. Do not add the raw mirror URL as a second plugin.

If Nova was already loaded before the bridge, remove the duplicate URL and restart again; do not treat that run as valid.

- [ ] **Step 4: Verify loader ordering and single-load behavior**

In browser developer tools:

- window.nova_skin_lampac_loader is true;
- window.nova_skin_probe_mode() returns external;
- window.nova_skin is true;
- Network shows one Nova request to hlushok.github.io/nova-skin/nova_skin.js with one hourly v query;
- no request falls back to amikdn.github.io or another mirror.

Expected: one bridge, one mirror load, external mode.

- [ ] **Step 5: Verify ordinary Nova behavior**

Open several cards and verify:

- Nova presentation and navigation load normally;
- the full source list remains visible;
- manual source selection works;
- playback starts normally;
- failure-driven source switching still works when naturally encountered;
- removing or disabling the background-check setting does not hide ordinary sources.

Expected: Nova is a complete UI plugin, not a Premium-only plugin.

- [ ] **Step 6: Verify group-3 background behavior and absence of client fan-out**

With browser Network recording enabled:

1. Open a card with several Lampac Online sources.
2. Confirm the normal Lampac orchestration and lifeevents flow completes.
3. Confirm Nova displays server-supplied ghost/quality state when the selected provider supplies it.
4. Search browser requests for checksearch=true.
5. Confirm there is no Nova-generated per-source browser fan-out.

Do not infer this from HTTP 200 alone. Record the relevant endpoint classes, request count, timestamps, and visible UI result without exposing account parameters.

- [ ] **Step 7: Stop at the user confirmation checkpoint**

Ask the user whether the personal pilot behaves correctly. Do not add the bridge to autoload, lampainit, presets.js, a VPS loader, or a blacklist.

- [ ] **Step 8: Keep the lower-group test and global rollout deferred**

Before any future global rollout, use a separate lower-group account and prove:

- Nova UI, sources, manual selection, and playback remain available;
- window.nova_skin_probe_mode() remains external;
- Lampac denies background initiation and lifeevents results;
- no client checksearch=true fan-out occurs.

This is a future gate, not part of the personal group-3 handoff.

## Rollback

For the personal pilot, remove https://hlushok.github.io/lampa-plugin/nova_skin_lampac.js from Extensions and fully restart Lampa. No server restart or configuration rollback is required.

If the bridge itself is defective, preserve evidence and revert only its commit in Hlushok/lampa-plugin. Never alter the byte-for-byte mirror to compensate for a bridge problem.

## Completion Evidence

Report:

- mirror repository URL, public payload URL, downstream main SHA, immutable upstream SHA, and matching SHA-256;
- first workflow run URL and no-op workflow run URL;
- bridge repository main SHA, public URL, HTTP status, and matching local/remote/public SHA-256;
- six loader tests and static contract result;
- read-only Lampac guard/config/service results and group-3 boolean without identifiers;
- manual Nova UI/playback/network result after the user tests it;
- explicit statement that autoload, Lampac config, VPS files, blacklist, services, and global plugin lists were unchanged;
- reminder that GitHub scheduled workflows may require manual re-enabling after prolonged repository inactivity.
