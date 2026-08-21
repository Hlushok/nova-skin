# Nova Skin Mirror Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `Hlushok/nova-skin` as a minimal GitHub-native fork that mirrors upstream `nova_skin.js` hourly and publishes the validated bytes at a stable GitHub Pages URL.

**Architecture:** Keep GitHub fork provenance while replacing the downstream branch tree with one mirrored plugin, attribution, reviewed docs, and a downstream-owned workflow. The workflow resolves an immutable upstream file commit, parses but never executes the downloaded JavaScript, commits changed bytes, packages only the plugin for Pages, and deploys through GitHub's custom Pages workflow path so `GITHUB_TOKEN` commits cannot leave the public URL stale.

**Tech Stack:** Git, GitHub CLI and REST API, GitHub Actions on `ubuntu-latest`, Bash, Node.js `--check`, SHA-256, official GitHub Pages Actions, and PowerShell for safe Windows orchestration.

**Spec:** `docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md`

## Global Constraints

- The GitHub repository must be a public native fork named `Hlushok/nova-skin` whose parent is `amikdn/amikdn.github.io`.
- Mirror only `nova_skin.js`; never merge upstream plugins, configuration, or workflows after initial fork creation.
- Keep `nova_skin.js` byte-for-byte identical to the file at the recorded immutable upstream commit.
- Apply only transport checks, a non-empty-file check, and `node --check`; never execute downloaded JavaScript.
- Publish only `nova_skin.js` and `.nojekyll` in the Pages artifact.
- The stable plugin URL is `https://hlushok.github.io/nova-skin/nova_skin.js`.
- Run synchronization at minute 17 of every hour and through `workflow_dispatch`.
- Use no secrets or PAT. The sync job gets `contents: write` and `pages: read`; the deploy job gets `pages: write` and `id-token: write`.
- Pin every reusable Action to an immutable 40-character commit SHA.
- Never force-push, alter Lampac, change Siaivo `autoload.json`, access the VPS, or add a license not present upstream.
- Preserve unrelated local user changes; stop on an unexpected merge conflict, remote mismatch, or repository-name collision.

## File Map

- Create `nova_skin.js`: exact upstream plugin payload with no downstream edits.
- Create `README.md`: Ukrainian usage, provenance, automation, risk, and GitHub schedule caveat.
- Create `.nojekyll`: static Pages marker copied into the deployment artifact.
- Create `.github/workflows/sync-upstream.yml`: immutable-source sync, no-change handling, commit, clean artifact creation, and Pages deployment.
- Preserve `docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md`.
- Preserve `docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md`.
- Add no package manifest, generated site tree, license, or additional plugin file.

---

### Task 1: Create the GitHub-native fork and connect the local repository

**Files:**
- Preserve: `docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md`
- Preserve: `docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md`

**Interfaces:**
- Consumes: authenticated GitHub CLI account `Hlushok` and clean local repository `D:/opt/nova-skin` on branch `main`.
- Produces: public fork `Hlushok/nova-skin` and local remote `origin` pointing to `https://github.com/Hlushok/nova-skin.git`, with `origin/main` fetched.

- [ ] **Step 1: Run the non-mutating preflight**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path

gh auth status
if ($LASTEXITCODE -ne 0) { throw 'GitHub CLI authentication is not valid.' }

$login = (gh api user --jq '.login').Trim()
if ($LASTEXITCODE -ne 0 -or $login -ne 'Hlushok') {
  throw "Expected GitHub account Hlushok, got '$login'."
}

$upstream = gh api repos/amikdn/amikdn.github.io | ConvertFrom-Json
if ($LASTEXITCODE -ne 0 -or $upstream.default_branch -ne 'main' -or -not $upstream.forkable) {
  throw 'Upstream is unavailable, does not use main, or is not forkable.'
}

$targetProbe = gh api repos/Hlushok/nova-skin 2>&1
if ($LASTEXITCODE -eq 0) {
  throw 'Hlushok/nova-skin already exists; stop before mutating it.'
}
if ($targetProbe -notmatch 'HTTP 404') {
  throw "Unexpected target-repository probe failure: $targetProbe"
}

if ((git -C $repoRoot branch --show-current).Trim() -ne 'main') {
  throw 'Local nova-skin repository is not on main.'
}
if (git -C $repoRoot status --porcelain=v1) {
  throw 'Local nova-skin repository is not clean.'
}

git -C $repoRoot log --oneline --decorate -3
```

Expected: authentication reports `Hlushok`, upstream reports `main` and forkable, target probe reports only HTTP 404, and local status is empty.

- [ ] **Step 2: Create the custom-named fork**

```powershell
gh api --method POST repos/amikdn/amikdn.github.io/forks `
  -f name='nova-skin' `
  -F default_branch_only=true `
  --jq '{full_name,private,fork,default_branch}'
if ($LASTEXITCODE -ne 0) { throw 'Fork creation request failed.' }
```

Expected: the API returns `full_name: Hlushok/nova-skin`, `private: false`, `fork: true`, and `default_branch: main`.

- [ ] **Step 3: Wait for fork materialization in one bounded pass**

```powershell
$fork = $null
for ($attempt = 1; $attempt -le 12; $attempt++) {
  $json = gh api repos/Hlushok/nova-skin 2>$null
  if ($LASTEXITCODE -eq 0) {
    $fork = $json | ConvertFrom-Json
    break
  }
  Start-Sleep -Seconds 5
}

if ($null -eq $fork) {
  throw 'Fork did not materialize within 60 seconds; report progress and run this bounded pass again.'
}
if (-not $fork.fork -or $fork.parent.full_name -ne 'amikdn/amikdn.github.io') {
  throw 'Created repository does not have the required fork parent.'
}
if ($fork.private -or $fork.default_branch -ne 'main') {
  throw 'Created fork is not public on main.'
}

$fork | Select-Object full_name, fork, private, default_branch,
  @{Name='parent';Expression={$_.parent.full_name}} | Format-List
```

Expected: `fork=True`, `private=False`, `parent=amikdn/amikdn.github.io`, and `default_branch=main`.

- [ ] **Step 4: Attach and fetch the fork without resetting local history**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$expectedOrigin = 'https://github.com/Hlushok/nova-skin.git'
$origin = git -C $repoRoot remote get-url origin 2>$null

if ($LASTEXITCODE -eq 0 -and $origin.Trim() -ne $expectedOrigin) {
  throw "Existing origin points to '$origin', not '$expectedOrigin'."
}
if ($LASTEXITCODE -ne 0) {
  git -C $repoRoot remote add origin $expectedOrigin
  if ($LASTEXITCODE -ne 0) { throw 'Failed to add origin.' }
}

git -C $repoRoot fetch --no-tags origin main
if ($LASTEXITCODE -ne 0) { throw 'Failed to fetch origin/main.' }

git -C $repoRoot remote -v
git -C $repoRoot show -s --format='%H %s' origin/main
```

Expected: both origin URLs equal the custom fork URL and `origin/main` resolves to an upstream-derived commit.

---

### Task 2: Construct and commit the minimal downstream branch

**Files:**
- Create: `nova_skin.js`
- Create: `README.md`
- Create: `.nojekyll`
- Preserve: both reviewed documents under `docs/superpowers/`

**Interfaces:**
- Consumes: fetched `origin/main` from Task 1 and the GitHub commits API filtered to `nova_skin.js`.
- Produces: a local merge commit descending from `origin/main`; its tree contains only the initial payload, attribution/static marker, and reviewed docs, and its body records `Upstream-Commit` and `SHA256`.

- [ ] **Step 1: Run the acceptance check and confirm it fails before construction**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$required = @('.nojekyll', 'README.md', 'nova_skin.js')
$missing = @($required | Where-Object {
  -not (Test-Path -LiteralPath (Join-Path $repoRoot $_))
})
if ($missing.Count -eq 0) {
  throw 'Foundation files already exist unexpectedly.'
}
"Expected pre-construction failure: missing $($missing -join ', ')"
```

Expected: the command reports all three foundation files as missing.

- [ ] **Step 2: Begin a merge that preserves native fork ancestry**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot merge --allow-unrelated-histories --no-commit --no-ff origin/main
if ($LASTEXITCODE -ne 0) {
  git -C $repoRoot status --short
  throw 'Unexpected merge conflict; stop without guessing or pushing.'
}
```

Expected: Git reports that the automatic merge succeeded and stopped before committing.

- [ ] **Step 3: Remove inherited files through a validated PowerShell allowlist**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$repoPrefix = $repoRoot.TrimEnd([IO.Path]::DirectorySeparatorChar) +
  [IO.Path]::DirectorySeparatorChar
$keep = [Collections.Generic.HashSet[string]]::new([StringComparer]::Ordinal)
@(
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'nova_skin.js'
) | ForEach-Object { [void]$keep.Add($_) }

$tracked = @(git -C $repoRoot ls-files)
if ($LASTEXITCODE -ne 0) { throw 'Unable to enumerate merge-tree files.' }
$remove = @($tracked | Where-Object { -not $keep.Contains($_) })

foreach ($relative in $remove) {
  if ([IO.Path]::IsPathRooted($relative) -or ($relative -split '/') -contains '..') {
    throw "Unsafe tracked path: $relative"
  }
  $nativeRelative = $relative.Replace(
    [IO.Path]::AltDirectorySeparatorChar,
    [IO.Path]::DirectorySeparatorChar
  )
  $resolved = [IO.Path]::GetFullPath((Join-Path $repoRoot $nativeRelative))
  if (-not $resolved.StartsWith($repoPrefix, [StringComparison]::OrdinalIgnoreCase)) {
    throw "Tracked path resolves outside repository: $relative"
  }
  git -C $repoRoot rm -f -- $relative
  if ($LASTEXITCODE -ne 0) { throw "git rm failed for $relative" }
}

"Removed $($remove.Count) unrelated inherited files."
git -C $repoRoot status --short
```

Expected: every inherited file except `nova_skin.js` is staged for deletion; both reviewed docs remain.

- [ ] **Step 4: Create the attribution README and static marker using `apply_patch`**

Create `README.md` with exactly:

```markdown
# Nova Skin mirror

Автоматичне дзеркало плагіна **Nova Skin** для Lampa.

## Підключення

`https://hlushok.github.io/nova-skin/nova_skin.js`

## Походження

- Авторське джерело: [amikdn/amikdn.github.io](https://github.com/amikdn/amikdn.github.io)
- Оригінальний файл: [nova_skin.js](https://github.com/amikdn/amikdn.github.io/blob/main/nova_skin.js)
- Це GitHub-форк і byte-for-byte дзеркало, а не окремий переписаний плагін.

У перевіреному upstream-репозиторії не знайдено файла ліцензії. Цей форк не
додає власної ліцензії та не заявляє авторство на upstream-код.

## Автоматичне оновлення

[![Sync Nova Skin](https://github.com/Hlushok/nova-skin/actions/workflows/sync-upstream.yml/badge.svg)](https://github.com/Hlushok/nova-skin/actions/workflows/sync-upstream.yml)

Workflow перевіряє upstream щогодини на 17-й хвилині UTC та підтримує ручний
запуск. Він завантажує лише `nova_skin.js` з зафіксованого commit SHA, перевіряє
доставку й JavaScript-синтаксис, але не проводить ручний аудит поведінки.

GitHub автоматично вимикає cron у публічному репозиторії після 60 днів без
активності. У такому разі workflow потрібно знову ввімкнути в Actions.

## Межі

Інші файли та плагіни upstream сюди автоматично не переносяться. Підключення до
Siaivo/Lampac і розгортання на сервері не входять до цього репозиторію.
```

Create `.nojekyll` as a tracked empty marker file. Do not create `LICENSE`, `index.html`, a package manifest, or another plugin file.

- [ ] **Step 5: Download the immutable payload, validate it, and create the merge commit**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$upstreamCommit = (
  gh api --method GET repos/amikdn/amikdn.github.io/commits `
    -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha'
).Trim()
if ($LASTEXITCODE -ne 0 -or $upstreamCommit -notmatch '^[0-9a-f]{40}$') {
  throw "Invalid upstream file commit '$upstreamCommit'."
}

$downloadPath = Join-Path ([IO.Path]::GetTempPath()) (
  'nova-skin-' + [guid]::NewGuid().ToString('N') + '.js'
)
try {
  $rawUrl = "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js"
  curl.exe --fail --silent --show-error --location --retry 3 `
    $rawUrl --output $downloadPath
  if ($LASTEXITCODE -ne 0) { throw 'Upstream download failed.' }
  if (-not (Test-Path -LiteralPath $downloadPath) -or
      (Get-Item -LiteralPath $downloadPath).Length -eq 0) {
    throw 'Downloaded plugin is missing or empty.'
  }

  node --check $downloadPath
  if ($LASTEXITCODE -ne 0) { throw 'Downloaded plugin failed node --check.' }

  $digest = (Get-FileHash -LiteralPath $downloadPath -Algorithm SHA256).Hash.ToLowerInvariant()
  Move-Item -LiteralPath $downloadPath `
    -Destination (Join-Path $repoRoot 'nova_skin.js') -Force
  $downloadPath = $null

  git -C $repoRoot add -- .nojekyll README.md nova_skin.js `
    docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md `
    docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md
  if ($LASTEXITCODE -ne 0) { throw 'Failed to stage the minimal mirror.' }

  git -C $repoRoot diff --cached --check
  if ($LASTEXITCODE -ne 0) { throw 'Staged tree failed git diff --check.' }

  git -C $repoRoot commit `
    -m 'feat: create minimal Nova Skin mirror' `
    -m "Upstream-Commit: $upstreamCommit" `
    -m "SHA256: $digest"
  if ($LASTEXITCODE -ne 0) { throw 'Minimal mirror commit failed.' }
}
finally {
  if ($null -ne $downloadPath -and (Test-Path -LiteralPath $downloadPath)) {
    Remove-Item -LiteralPath $downloadPath -Force
  }
}
```

Expected: `node --check` exits 0 and Git creates one merge commit containing the recorded upstream SHA and SHA-256.

- [ ] **Step 6: Verify ancestry, exact bytes, and the foundation allowlist**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot merge-base --is-ancestor origin/main HEAD
if ($LASTEXITCODE -ne 0) { throw 'HEAD does not descend from fork main.' }

$expected = @(
  '.nojekyll',
  'README.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'nova_skin.js'
) | Sort-Object
$actual = @(git -C $repoRoot ls-tree -r --name-only HEAD) | Sort-Object
$shapeDiff = Compare-Object $expected $actual
if ($shapeDiff) {
  $shapeDiff | Format-Table
  throw 'Foundation tree differs from the allowlist.'
}

$message = git -C $repoRoot log -1 --format='%B'
$upstreamCommit = (
  $message | Select-String -Pattern 'Upstream-Commit: ([0-9a-f]{40})'
).Matches.Groups[1].Value
$expectedDigest = (
  $message | Select-String -Pattern 'SHA256: ([0-9a-f]{64})'
).Matches.Groups[1].Value
if ($upstreamCommit -notmatch '^[0-9a-f]{40}$' -or
    $expectedDigest -notmatch '^[0-9a-f]{64}$') {
  throw 'Commit provenance trailers are missing.'
}

$actualDigest = (
  Get-FileHash -LiteralPath (Join-Path $repoRoot 'nova_skin.js') -Algorithm SHA256
).Hash.ToLowerInvariant()
if ($actualDigest -ne $expectedDigest) {
  throw "Payload digest $actualDigest does not match recorded $expectedDigest."
}

node --check (Join-Path $repoRoot 'nova_skin.js')
if ($LASTEXITCODE -ne 0) { throw 'Committed payload failed node --check.' }
git -C $repoRoot status --short
```

Expected: ancestry and allowlist checks pass, hashes match, JavaScript parses, and status is empty.

---

### Task 3: Add the trusted sync and Pages deployment workflow

**Files:**
- Create: `.github/workflows/sync-upstream.yml`
- Verify: `nova_skin.js`
- Verify: `README.md`

**Interfaces:**
- Consumes: local `main` from Task 2, the public GitHub API, raw immutable upstream URL, and repository `GITHUB_TOKEN`.
- Produces: workflow output `sync.changed` as string `true` or `false`; a Pages artifact containing only `_site/nova_skin.js` and `_site/.nojekyll`; the `github-pages` deployment URL.

- [ ] **Step 1: Run the workflow-presence check and confirm it fails**

```powershell
$workflow = 'D:/opt/nova-skin/.github/workflows/sync-upstream.yml'
if (Test-Path -LiteralPath $workflow) {
  throw 'Workflow exists before its implementation task.'
}
'Expected pre-implementation failure: sync-upstream.yml is absent.'
```

Expected: the command reports that the workflow is absent.

- [ ] **Step 2: Create `.github/workflows/sync-upstream.yml` using `apply_patch`**

Use this exact workflow:

```yaml
name: Sync Nova Skin

on:
  workflow_dispatch:
  schedule:
    - cron: '17 * * * *'

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
    outputs:
      changed: ${{ steps.publish.outputs.changed }}
    steps:
      - name: Checkout downstream
        uses: actions/checkout@d23441a48e516b6c34aea4fa41551a30e30af803 # v6
        with:
          fetch-depth: 0

      - name: Resolve and validate immutable upstream file
        id: upstream
        shell: bash
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          set -euo pipefail

          commit_sha="$(
            gh api --method GET repos/amikdn/amikdn.github.io/commits \
              -f sha=main \
              -f path=nova_skin.js \
              -F per_page=1 \
              --jq '.[0].sha'
          )"
          if [[ ! "$commit_sha" =~ ^[0-9a-f]{40}$ ]]; then
            echo "Invalid upstream commit: $commit_sha" >&2
            exit 1
          fi

          download="$RUNNER_TEMP/nova_skin-${commit_sha}.js"
          curl --fail --silent --show-error --location --retry 3 \
            "https://raw.githubusercontent.com/amikdn/amikdn.github.io/${commit_sha}/nova_skin.js" \
            --output "$download"
          test -s "$download"
          node --check "$download"

          digest="$(sha256sum "$download" | cut -d' ' -f1)"
          printf 'commit_sha=%s\n' "$commit_sha" >> "$GITHUB_OUTPUT"
          printf 'sha256=%s\n' "$digest" >> "$GITHUB_OUTPUT"
          printf 'download_path=%s\n' "$download" >> "$GITHUB_OUTPUT"

      - name: Commit changed upstream bytes
        id: publish
        shell: bash
        env:
          DOWNLOAD_PATH: ${{ steps.upstream.outputs.download_path }}
          UPSTREAM_COMMIT: ${{ steps.upstream.outputs.commit_sha }}
          UPSTREAM_SHA256: ${{ steps.upstream.outputs.sha256 }}
        run: |
          set -euo pipefail

          if cmp -s "$DOWNLOAD_PATH" nova_skin.js; then
            echo "nova_skin.js already matches upstream $UPSTREAM_COMMIT"
            echo 'changed=false' >> "$GITHUB_OUTPUT"
            exit 0
          fi

          install -m 0644 "$DOWNLOAD_PATH" nova_skin.js.next
          mv -f nova_skin.js.next nova_skin.js
          git diff --check -- nova_skin.js
          git config user.name 'github-actions[bot]'
          git config user.email '41898282+github-actions[bot]@users.noreply.github.com'
          git add -- nova_skin.js
          git diff --cached --check

          short_sha="${UPSTREAM_COMMIT:0:12}"
          git commit \
            -m "chore: sync nova_skin.js from upstream $short_sha" \
            -m "Upstream-Commit: $UPSTREAM_COMMIT" \
            -m "SHA256: $UPSTREAM_SHA256"
          git push origin HEAD:main
          echo 'changed=true' >> "$GITHUB_OUTPUT"

      - name: Configure Pages
        uses: actions/configure-pages@983d7736d9b0ae728b81ab479565c72886d7745b # v5

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
        uses: actions/upload-pages-artifact@7b1f4a764d45c48632c6b24a0339c27f5614fb0b # v4
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
        uses: actions/deploy-pages@d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e # v4
```

- [ ] **Step 3: Parse/format-check the workflow and verify immutable Action pins**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
$workflowPath = Join-Path $repoRoot '.github/workflows/sync-upstream.yml'

npx.cmd --yes prettier@3.6.2 --check $workflowPath
if ($LASTEXITCODE -ne 0) {
  throw 'Prettier could not parse or format-check the workflow.'
}

$workflowText = Get-Content -LiteralPath $workflowPath -Encoding utf8 -Raw
$pins = [regex]::Matches(
  $workflowText,
  'uses:\s+([A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+)@([0-9a-f]{40})'
)
if ($pins.Count -ne 4) {
  throw "Expected four immutable Action pins, found $($pins.Count)."
}
if ($workflowText -match 'uses:\s+\S+@(v\d+|main|master)(\s|$)') {
  throw 'A mutable Action reference remains.'
}
if ($workflowText -notmatch "cron:\s+'17 \* \* \* \*'" -or
    $workflowText -notmatch 'workflow_dispatch:') {
  throw 'Required schedule or manual trigger is missing.'
}

node --check (Join-Path $repoRoot 'nova_skin.js')
if ($LASTEXITCODE -ne 0) { throw 'Current plugin failed node --check.' }
git -C $repoRoot diff --check
```

Expected: Prettier parses the YAML, four SHA pins are found, both triggers are present, and syntax/diff checks pass.

- [ ] **Step 4: Exercise the immutable resolver without modifying the repository**

```powershell
$upstreamCommit = (
  gh api --method GET repos/amikdn/amikdn.github.io/commits `
    -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha'
).Trim()
if ($LASTEXITCODE -ne 0 -or $upstreamCommit -notmatch '^[0-9a-f]{40}$') {
  throw 'Resolver did not return a 40-character commit SHA.'
}

$probePath = Join-Path ([IO.Path]::GetTempPath()) (
  'nova-skin-resolver-' + [guid]::NewGuid().ToString('N') + '.js'
)
try {
  curl.exe --fail --silent --show-error --location --retry 3 `
    "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js" `
    --output $probePath
  if ($LASTEXITCODE -ne 0 -or
      (Get-Item -LiteralPath $probePath).Length -eq 0) {
    throw 'Resolver probe download failed or is empty.'
  }
  node --check $probePath
  if ($LASTEXITCODE -ne 0) { throw 'Resolver probe is not valid JavaScript.' }
  Get-FileHash -LiteralPath $probePath -Algorithm SHA256
}
finally {
  if (Test-Path -LiteralPath $probePath) {
    Remove-Item -LiteralPath $probePath -Force
  }
}
```

Expected: the resolver returns an immutable commit, downloads a non-empty file, and `node --check` exits 0.

- [ ] **Step 5: Commit the downstream-owned workflow**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot add -- .github/workflows/sync-upstream.yml
git -C $repoRoot diff --cached --check
if ($LASTEXITCODE -ne 0) { throw 'Workflow staging failed validation.' }

git -C $repoRoot commit -m 'ci: mirror Nova Skin and deploy Pages'
if ($LASTEXITCODE -ne 0) { throw 'Workflow commit failed.' }

git -C $repoRoot status --short
git -C $repoRoot log --oneline --decorate -4
```

Expected: one workflow commit is created and local status is empty.

---

### Task 4: Review, push, enable automation, and perform the first deployment

**Files:**
- Publish: the six allowed blobs from Tasks 2-3

**Interfaces:**
- Consumes: clean local `main` and remote `origin/main` still at the new fork baseline.
- Produces: fast-forwarded public `main`, enabled workflow `sync-upstream.yml`, Pages configured with `build_type: workflow`, and one successful manual run/deployment.

- [ ] **Step 1: Run the pre-push review and secret scan**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
if (git -C $repoRoot status --porcelain=v1) {
  throw 'Local repository is not clean before push.'
}

$remoteLine = git -C $repoRoot ls-remote origin refs/heads/main
if ($LASTEXITCODE -ne 0 -or -not $remoteLine) {
  throw 'Unable to resolve remote main.'
}
$remoteMain = $remoteLine.Split()[0]
if ($remoteMain -notmatch '^[0-9a-f]{40}$') {
  throw "Invalid remote main SHA '$remoteMain'."
}
git -C $repoRoot merge-base --is-ancestor $remoteMain HEAD
if ($LASTEXITCODE -ne 0) {
  throw 'Push would not be a fast-forward from current remote main.'
}

git -C $repoRoot diff --check origin/main..HEAD
if ($LASTEXITCODE -ne 0) { throw 'Pre-push diff check failed.' }
git -C $repoRoot diff --stat origin/main..HEAD
git -C $repoRoot log --oneline --decorate origin/main..HEAD

$secretMatches = rg -n --hidden -g '!.git/**' `
  '(ghp_[A-Za-z0-9]+|github_pat_[A-Za-z0-9_]+|AKIA[0-9A-Z]{16}|BEGIN (RSA |OPENSSH |EC )?PRIVATE KEY|Authorization:\s*Bearer\s+\S+)' `
  $repoRoot
if ($LASTEXITCODE -eq 0) {
  $secretMatches
  throw 'Secret-like material found; do not push.'
}
if ($LASTEXITCODE -gt 1) { throw 'Secret scan failed to run.' }
```

Expected: push is fast-forward, diff check passes, the reviewed commits are shown, and the secret scan has no matches.

- [ ] **Step 2: Push the minimal branch without force**

```powershell
$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot push --set-upstream origin main
if ($LASTEXITCODE -ne 0) { throw 'Fast-forward push failed.' }

$localHead = (git -C $repoRoot rev-parse HEAD).Trim()
$remoteHead = (gh api repos/Hlushok/nova-skin/commits/main --jq '.sha').Trim()
if ($localHead -ne $remoteHead) {
  throw "Remote head $remoteHead does not match local head $localHead."
}
```

Expected: Git reports a normal fast-forward push and both head SHAs match.

- [ ] **Step 3: Verify the public tree before enabling writes**

```powershell
$expected = @(
  '.github/workflows/sync-upstream.yml',
  '.nojekyll',
  'README.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'nova_skin.js'
) | Sort-Object
$actual = @(
  gh api 'repos/Hlushok/nova-skin/git/trees/main?recursive=1' `
    --jq '.tree[] | select(.type == "blob") | .path'
) | Sort-Object
$treeDiff = Compare-Object $expected $actual
if ($treeDiff) {
  $treeDiff | Format-Table
  throw 'Public repository tree differs from the six-file allowlist.'
}

gh api repos/Hlushok/nova-skin `
  --jq '{full_name,private,fork,parent:.parent.full_name,default_branch}'
```

Expected: exactly six blobs are present and metadata confirms the required public fork parent.

- [ ] **Step 4: Enable the fork workflow and configure workflow-based Pages**

```powershell
gh workflow enable sync-upstream.yml --repo Hlushok/nova-skin
if ($LASTEXITCODE -ne 0) { throw 'Failed to enable sync workflow.' }

$workflowState = (
  gh api repos/Hlushok/nova-skin/actions/workflows/sync-upstream.yml --jq '.state'
).Trim()
if ($workflowState -ne 'active') {
  throw "Workflow state is '$workflowState', not active."
}

$pagesProbe = gh api repos/Hlushok/nova-skin/pages 2>&1
$pagesExit = $LASTEXITCODE
if ($pagesExit -eq 0) {
  gh api --method PUT repos/Hlushok/nova-skin/pages -f build_type=workflow
  if ($LASTEXITCODE -ne 0) { throw 'Failed to update Pages build type.' }
}
elseif ($pagesProbe -match 'HTTP 404') {
  gh api --method POST repos/Hlushok/nova-skin/pages -f build_type=workflow
  if ($LASTEXITCODE -ne 0) { throw 'Failed to create workflow-based Pages site.' }
}
else {
  throw "Unexpected Pages probe failure: $pagesProbe"
}

$buildType = (gh api repos/Hlushok/nova-skin/pages --jq '.build_type').Trim()
if ($buildType -ne 'workflow') {
  throw "Pages build type is '$buildType', not workflow."
}
```

Expected: workflow state is `active` and Pages reports `build_type: workflow`.

- [ ] **Step 5: Dispatch the first manual run and monitor it with bounded tool polling**

```powershell
gh workflow run sync-upstream.yml --repo Hlushok/nova-skin --ref main
if ($LASTEXITCODE -ne 0) { throw 'Manual workflow dispatch failed.' }

Start-Sleep -Seconds 3
$runId = (
  gh run list --repo Hlushok/nova-skin `
    --workflow sync-upstream.yml `
    --event workflow_dispatch `
    --limit 1 `
    --json databaseId `
    --jq '.[0].databaseId'
).Trim()
if ($runId -notmatch '^\d+$') { throw "Invalid run id '$runId'." }
"Run ID: $runId"
```

Start `gh run watch $runId --repo Hlushok/nova-skin --exit-status --interval 5` through the command tool with a short initial yield. Poll its running session in bounded calls and send a concise progress update at least once per minute. Do not launch a second run while this run is active.

Expected: `sync` succeeds and `deploy` succeeds. Every successful run redeploys the validated artifact, even when `sync.changed` is `false`, so a transient Pages failure is retried by the next schedule.

- [ ] **Step 6: Inspect the exact jobs and deployment**

```powershell
$runId = (
  gh run list --repo Hlushok/nova-skin `
    --workflow sync-upstream.yml `
    --event workflow_dispatch `
    --limit 1 `
    --json databaseId `
    --jq '.[0].databaseId'
).Trim()
if ($runId -notmatch '^\d+$') { throw "Invalid latest manual run id '$runId'." }

gh run view $runId --repo Hlushok/nova-skin `
  --json conclusion,event,headSha,jobs,url
if ($LASTEXITCODE -ne 0) { throw 'Unable to inspect the first run.' }

$conclusion = (
  gh run view $runId --repo Hlushok/nova-skin --json conclusion --jq '.conclusion'
).Trim()
if ($conclusion -ne 'success') {
  gh run view $runId --repo Hlushok/nova-skin --log-failed
  throw "First run concluded '$conclusion'."
}

$pages = gh api repos/Hlushok/nova-skin/pages | ConvertFrom-Json
$pages | Select-Object status, build_type, html_url, https_enforced | Format-List
if ($pages.build_type -ne 'workflow' -or -not $pages.html_url) {
  throw 'Pages configuration is incomplete after deployment.'
}
```

Expected: run conclusion is `success`, both jobs expose successful steps, and Pages exposes the repository site URL.

---

### Task 5: Prove no-op behavior and public byte-for-byte delivery

**Files:**
- Verify only: all six public repository blobs
- Verify only: `https://hlushok.github.io/nova-skin/nova_skin.js`

**Interfaces:**
- Consumes: successful first run from Task 4.
- Produces: fork metadata, tree allowlist, workflow/run URLs, stable-main no-op proof, Pages configuration, HTTP result, public SHA-256, downstream SHA-256, and immutable upstream SHA.

- [ ] **Step 1: Dispatch a controlled no-change run**

```powershell
$priorRun = (
  gh run list --repo Hlushok/nova-skin --workflow sync-upstream.yml `
    --limit 1 --json databaseId --jq '.[0].databaseId'
).Trim()

gh workflow run sync-upstream.yml --repo Hlushok/nova-skin --ref main
if ($LASTEXITCODE -ne 0) { throw 'No-change dispatch failed.' }

Start-Sleep -Seconds 3
$noOpRun = (
  gh run list --repo Hlushok/nova-skin --workflow sync-upstream.yml `
    --event workflow_dispatch --limit 1 --json databaseId `
    --jq '.[0].databaseId'
).Trim()
if ($noOpRun -notmatch '^\d+$' -or $noOpRun -eq $priorRun) {
  throw 'Could not identify the new no-change run.'
}
"No-op candidate run ID: $noOpRun"
```

Monitor `$noOpRun` with the same bounded `gh run watch` procedure from Task 4.

Expected: the run succeeds, its sync log says `nova_skin.js already matches upstream`, and deploy succeeds without creating a Git commit.

- [ ] **Step 2: Prove the run did not create a commit while upstream stayed stable**

```powershell
$noOpRun = (
  gh run list --repo Hlushok/nova-skin --workflow sync-upstream.yml `
    --event workflow_dispatch --limit 1 --json databaseId `
    --jq '.[0].databaseId'
).Trim()
if ($noOpRun -notmatch '^\d+$') { throw "Invalid no-op run id '$noOpRun'." }

$run = gh run view $noOpRun --repo Hlushok/nova-skin `
  --json conclusion,headSha,url | ConvertFrom-Json
if ($run.conclusion -ne 'success') {
  throw "No-op candidate concluded '$($run.conclusion)'."
}

$noOpLog = gh run view $noOpRun --repo Hlushok/nova-skin --log
if ($LASTEXITCODE -ne 0) { throw 'Unable to read the no-op run log.' }
$marker = [regex]::Match(
  $noOpLog,
  'nova_skin\.js already matches upstream ([0-9a-f]{40})'
)
if (-not $marker.Success) { throw 'No-change log marker was not found.' }
$runUpstream = $marker.Groups[1].Value

$currentUpstream = (
  gh api --method GET repos/amikdn/amikdn.github.io/commits `
    -f sha=main -f path=nova_skin.js -F per_page=1 --jq '.[0].sha'
).Trim()
$mainAfter = (gh api repos/Hlushok/nova-skin/commits/main --jq '.sha').Trim()

if ($currentUpstream -ne $runUpstream) {
  throw 'Upstream changed during the no-op test; finish this run, then repeat Task 5 Steps 1-2 with a fresh baseline.'
}
if ($mainAfter -ne $run.headSha) {
  throw "Downstream main $mainAfter differs from the no-op run head $($run.headSha)."
}
```

Expected: current upstream matches the run marker and downstream `main` still equals the workflow's triggering `headSha`.

- [ ] **Step 3: Compare upstream, downstream, and public payload bytes**

```powershell
$mirrorCommit = gh api --method GET repos/Hlushok/nova-skin/commits `
  -f sha=main -f path=nova_skin.js -F per_page=1 | ConvertFrom-Json
if ($LASTEXITCODE -ne 0 -or $mirrorCommit.Count -lt 1) {
  throw 'Unable to resolve the latest downstream payload commit.'
}
$recorded = [regex]::Match(
  $mirrorCommit[0].commit.message,
  'Upstream-Commit: ([0-9a-f]{40})'
)
if (-not $recorded.Success) {
  throw 'Latest downstream payload commit lacks Upstream-Commit provenance.'
}
$upstreamCommit = $recorded.Groups[1].Value
if ($upstreamCommit -notmatch '^[0-9a-f]{40}$') {
  throw 'Invalid upstream SHA during public verification.'
}

$tempRoot = Join-Path ([IO.Path]::GetTempPath()) (
  'nova-skin-verify-' + [guid]::NewGuid().ToString('N')
)
New-Item -ItemType Directory -LiteralPath $tempRoot | Out-Null
$upstreamPath = Join-Path $tempRoot 'upstream.js'
$downstreamPath = Join-Path $tempRoot 'downstream.js'
$publicPath = Join-Path $tempRoot 'public.js'

try {
  curl.exe --fail --silent --show-error --location --retry 3 `
    "https://raw.githubusercontent.com/amikdn/amikdn.github.io/$upstreamCommit/nova_skin.js" `
    --output $upstreamPath
  if ($LASTEXITCODE -ne 0) { throw 'Pinned upstream download failed.' }

  $downstreamHead = (gh api repos/Hlushok/nova-skin/commits/main --jq '.sha').Trim()
  curl.exe --fail --silent --show-error --location --retry 3 `
    "https://raw.githubusercontent.com/Hlushok/nova-skin/$downstreamHead/nova_skin.js" `
    --output $downstreamPath
  if ($LASTEXITCODE -ne 0) { throw 'Pinned downstream download failed.' }

  $cacheBuster = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
  $response = Invoke-WebRequest `
    -Uri "https://hlushok.github.io/nova-skin/nova_skin.js?rev=$upstreamCommit&ts=$cacheBuster" `
    -Headers @{ 'Cache-Control' = 'no-cache' } `
    -OutFile $publicPath `
    -PassThru
  if ($response.StatusCode -ne 200) {
    throw "Public plugin returned HTTP $($response.StatusCode)."
  }

  $upstreamHash = (
    Get-FileHash -LiteralPath $upstreamPath -Algorithm SHA256
  ).Hash.ToLowerInvariant()
  $downstreamHash = (
    Get-FileHash -LiteralPath $downstreamPath -Algorithm SHA256
  ).Hash.ToLowerInvariant()
  $publicHash = (
    Get-FileHash -LiteralPath $publicPath -Algorithm SHA256
  ).Hash.ToLowerInvariant()

  [pscustomobject]@{
    UpstreamCommit = $upstreamCommit
    DownstreamHead = $downstreamHead
    UpstreamSHA256 = $upstreamHash
    DownstreamSHA256 = $downstreamHash
    PublicSHA256 = $publicHash
    HTTP = $response.StatusCode
  } | Format-List

  if ($upstreamHash -ne $downstreamHash -or $upstreamHash -ne $publicHash) {
    throw 'Upstream, downstream, and public SHA-256 values differ.'
  }
  node --check $publicPath
  if ($LASTEXITCODE -ne 0) { throw 'Public payload failed node --check.' }
}
finally {
  $tempBase = [IO.Path]::GetFullPath([IO.Path]::GetTempPath())
  $resolvedTemp = [IO.Path]::GetFullPath($tempRoot)
  if ((Test-Path -LiteralPath $resolvedTemp) -and
      $resolvedTemp.StartsWith($tempBase, [StringComparison]::OrdinalIgnoreCase)) {
    Remove-Item -LiteralPath $resolvedTemp -Recurse -Force
  }
}
```

Expected: HTTP 200, all three SHA-256 values are identical, and the public plugin parses. If Pages is still propagating, rerun only this step after 30 seconds, reporting progress at least once per minute, for no more than 20 attempts.

- [ ] **Step 4: Run final metadata, allowlist, workflow, and local checks**

```powershell
$repo = gh api repos/Hlushok/nova-skin | ConvertFrom-Json
if (-not $repo.fork -or $repo.private -or
    $repo.parent.full_name -ne 'amikdn/amikdn.github.io' -or
    $repo.default_branch -ne 'main') {
  throw 'Final fork metadata check failed.'
}

$expected = @(
  '.github/workflows/sync-upstream.yml',
  '.nojekyll',
  'README.md',
  'docs/superpowers/plans/2026-08-21-nova-skin-mirror-implementation.md',
  'docs/superpowers/specs/2026-08-21-nova-skin-mirror-design.md',
  'nova_skin.js'
) | Sort-Object
$actual = @(
  gh api 'repos/Hlushok/nova-skin/git/trees/main?recursive=1' `
    --jq '.tree[] | select(.type == "blob") | .path'
) | Sort-Object
if (Compare-Object $expected $actual) {
  throw 'Final public tree allowlist check failed.'
}

$workflow = gh api repos/Hlushok/nova-skin/actions/workflows/sync-upstream.yml |
  ConvertFrom-Json
if ($workflow.state -ne 'active') { throw 'Final workflow state is not active.' }

$workflowYaml = gh api `
  repos/Hlushok/nova-skin/contents/.github/workflows/sync-upstream.yml `
  -H 'Accept: application/vnd.github.raw+json'
if ($workflowYaml -notmatch "cron:\s+'17 \* \* \* \*'" -or
    $workflowYaml -notmatch 'workflow_dispatch:') {
  throw 'Final public workflow trigger check failed.'
}

$pages = gh api repos/Hlushok/nova-skin/pages | ConvertFrom-Json
if ($pages.build_type -ne 'workflow' -or -not $pages.https_enforced) {
  throw 'Final Pages configuration check failed.'
}

$repoRoot = (Resolve-Path -LiteralPath 'D:/opt/nova-skin').Path
git -C $repoRoot fetch origin main
git -C $repoRoot merge --ff-only origin/main
if ($LASTEXITCODE -ne 0) {
  throw 'Local main could not fast-forward to the workflow-created remote commit.'
}
$localHead = (git -C $repoRoot rev-parse HEAD).Trim()
$fetchedHead = (git -C $repoRoot rev-parse origin/main).Trim()
if ($localHead -ne $fetchedHead) {
  throw "Local head $localHead differs from origin/main $fetchedHead."
}
if (git -C $repoRoot status --porcelain=v1) {
  throw 'Local nova-skin worktree is not clean at handoff.'
}
node --check (Join-Path $repoRoot 'nova_skin.js')
if ($LASTEXITCODE -ne 0) { throw 'Final local JavaScript check failed.' }
```

Expected: metadata/configuration checks pass, the public tree contains exactly six blobs, workflow is active, Pages uses custom workflow publishing with HTTPS, local and remote heads match, and the local worktree is clean.

- [ ] **Step 5: Record the handoff evidence**

Report all of these values without claiming Lampa installation:

- repository URL and stable public plugin URL;
- downstream `main` SHA and immutable upstream file commit SHA;
- matching upstream/downstream/public SHA-256;
- first manual run URL and no-op manual run URL;
- Pages `build_type`, HTTPS state, and HTTP 200 result;
- exact six-file public allowlist;
- explicit statement that Lampac, Siaivo `autoload.json`, and the VPS were not changed;
- GitHub's 60-day public-repository schedule caveat and manual re-enable path.

## Deferred Premium proposal

This mirror plan deliberately does not modify the plugin. A separate reviewed design should make only **background source probing** Premium for authenticated Lampac group 3:

- keep ordinary source listing, manual source selection, and failure-driven auto-switch available to lower groups;
- authorize against server-side `requestInfo.user.group`, never a client-supplied group or a hidden UI toggle;
- reuse Lampac's existing guarded `checkOnlineSearch` / `/lifeevents` contract instead of Nova's current client-side fan-out whenever the result mapping is compatible;
- keep the setting visible for non-Premium users with a Premium badge and activation action, but enforce denial on the server;
- preserve Nova's current conservative limits during rollout: 12 sources maximum, two parallel requests, 20-second total budget, six-hour positive cache, and 30-minute empty cache;
- start Premium opt-in with the current default `false`; consider default-on only after measured bandwidth, timeout, and upstream-blocking behavior is acceptable;
- prefer an upstream extension hook plus a separate Lampac bridge. If upstream does not accept a hook, keep a separately named Premium build and patch pipeline rather than contaminating the byte-for-byte mirror.

This proposal requires its own brainstorming spec before any Nova or Lampac code, autoload, VPS, or production change.
