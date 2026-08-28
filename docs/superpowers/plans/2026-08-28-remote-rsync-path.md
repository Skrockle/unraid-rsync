# Remote rsync Path Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a safe optional per-connection remote rsync executable and produce an installable Unraid plugin package.

**Architecture:** Extend the backward-compatible connection shape and validate the new path centrally in `Credentials`. Pass the value through the existing materialized SSH transport to `Rsync::buildArgv()`, and reuse the SSH probe to run the chosen remote binary's `--version` command.

**Tech Stack:** PHP 8.4, PHPUnit, Unraid webGui PHP, Slackware `makepkg`, shell linting, XML manifest validation.

## Global Constraints

- Empty `remoteRsyncPath` means the existing PATH-based remote `rsync` behaviour.
- Only an absolute, single, shell-safe Unix path is accepted; no additional arguments.
- Both SSH PUSH and PULL use `--rsync-path=<path>` when configured.
- Do not modify or replace QNAP `/usr/bin/rsync`.
- Do not add a free-form rsync argument field or perform unrelated refactors.

---

### Task 1: Connection model, save path, and validation

**Files:**
- Modify: `tests/CredentialsTest.php`
- Modify: `tests/HandlerTest.php`
- Modify: `source/include/Credentials.php`
- Modify: `source/include/handler.php`

**Interfaces:**
- Produces: connection field `remoteRsyncPath: string`; `Credentials::isSafeRemoteRsyncPath(string): bool`.

- [ ] Add tests asserting the default/merged value is empty, whitespace is trimmed, `/opt/bin/rsync` is accepted, and relative paths, whitespace, semicolons, quotes, substitutions, and appended arguments are rejected.
- [ ] Add a handler normalisation test proving the submitted field is retained.
- [ ] Run only those tests and confirm failures are caused by the missing field/validation.
- [ ] Add `remoteRsyncPath` to the default, merge, save-normalisation, and validation paths with a strict absolute-path regular expression.
- [ ] Re-run the focused tests and confirm they pass.

### Task 2: SSH rsync argv for PUSH and PULL

**Files:**
- Modify: `tests/RsyncTest.php`
- Modify: `tests/RunnerTest.php`
- Modify: `source/include/Rsync.php`
- Modify: `source/include/Runner.php`

**Interfaces:**
- Consumes: `remoteRsyncPath: string` from the merged/materialized connection.
- Produces: SSH transport key `remoteRsyncPath`; argv token `--rsync-path=<path>`.

- [ ] Add a failing rsync-builder test for one exact `--rsync-path=/opt/bin/rsync` token before `--`, plus an empty-value compatibility test.
- [ ] Add failing runner tests that capture argv for SSH PUSH and PULL and verify the remote option, remote operand placement, `--mkpath`, and normal logging's `--info=progress2,stats2`.
- [ ] Run the focused tests and confirm the expected failures.
- [ ] Pass the path through `Runner` and emit it from `Rsync::buildArgv()` as one argv token.
- [ ] Re-run the focused tests and confirm they pass.

### Task 3: Connection UI and remote version probe

**Files:**
- Modify: `tests/SshTest.php`
- Modify: `tests/OptionsFormHelpTest.php` if the page-source test requires it
- Modify: `source/include/Ssh.php`
- Modify: `source/pages/connections.php`

**Interfaces:**
- Consumes: safe `remoteRsyncPath` from the merged connection.
- Produces: connection-test message containing the first version line and executable path/name.

- [ ] Add failing tests proving the probe ends in `/opt/bin/rsync --version`, blank configuration uses `rsync --version`, and a successful probe reports the version line and path.
- [ ] Extend the probe seam to return stdout without changing existing failure classification.
- [ ] Render an optional `Remote rsync path` field with `/opt/bin/rsync` help text.
- [ ] Re-run the focused tests and confirm they pass.

### Task 4: Full verification and installable package

**Files:**
- Modify during build: `unraid.rsync.plg` (generated version and MD5 only)
- Create during build: `archive/unraid.rsync-<version>.txz`
- Copy deliverables to: `outputs/`

**Interfaces:**
- Produces: matching `.plg` and `.txz` installation artifacts.

- [ ] Run `vendor/bin/phpunit` and confirm zero failures/warnings/risky tests.
- [ ] Run PHP syntax checks for every PHP and `.page` body, `xmllint` for the manifest, and shell syntax/lint checks for the build script.
- [ ] Build using the repository's Slackware container command and a local CalVer that safely hands future official updates back to the release channel.
- [ ] Verify the built `.txz` contains the changed runtime files and that the `.plg` MD5 equals the artifact MD5.
- [ ] Copy both artifacts to the user-facing outputs directory and provide the Unraid installation steps.
