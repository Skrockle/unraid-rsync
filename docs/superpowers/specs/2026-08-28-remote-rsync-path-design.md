# Remote rsync Path Design

## Goal

Allow each SSH connection to select an optional remote rsync executable, such as
`/opt/bin/rsync` on QNAP, without changing the local Unraid rsync binary or the
default behaviour of existing connections and jobs.

## Data model and compatibility

Connections gain a `remoteRsyncPath` string. Its default is the empty string.
`Credentials::mergeConnection()` fills the field for old records, so no schema
migration is needed and existing `credentials.json` files continue to load.
The Connections form saves the value alongside the existing host and SSH fields.

## Validation and execution

A non-empty value must be an absolute Unix path and may contain only safe path
characters. Whitespace, quotes, backticks, shell metacharacters, control
characters, and extra arguments are rejected. The value remains one argv token;
free-form rsync options are not introduced.

For SSH jobs, the runner copies the validated connection value into the existing
SSH transport structure. `Rsync::buildArgv()` emits exactly one
`--rsync-path=<path>` argument before the operand terminator. Because the same
builder handles both operand directions, PUSH and PULL use the configured remote
binary. An empty value emits nothing and preserves current behaviour.

## Connection test

The connection test runs `<configured-path> --version` remotely. With an empty
field it runs `rsync --version`, preserving PATH-based lookup while adding useful
diagnostics. A successful response includes the first non-empty version line and
the executable name/path. Authentication, host-key, and reachability errors keep
their existing classification.

## Tests and packaging

Tests cover default/merge compatibility, accepted and rejected paths, save
normalisation, argv generation with and without the option, PUSH and PULL
operands, modern remote flags (`--info=progress2,stats2` and `--mkpath`), and the
version-aware connection probe. Tests must be observed failing before production
code changes, then pass after the minimal implementation. The existing PHPUnit,
PHP syntax, manifest, shell, and package-build checks remain authoritative. The
existing Slackware build produces the installable `.txz` and matching rewritten
`.plg`; QNAP `/usr/bin/rsync` is never modified.
