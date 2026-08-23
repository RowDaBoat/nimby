# Changelog

## 0.2.0

First release since 0.1.27 (2026-05-24). Workspaces changed enough to justify
the minor bump.

### Breaking

#### Workspaces are explicit

Nimby used to create a `nim.cfg` workspace wherever you happened to be
standing. It no longer does that inside a Git checkout or a Nimble package. It
stops and makes you choose:

```
No Nimby workspace found.
Refusing to create one inside package or Git checkout: /home/me/myproject
Run `nimby create` in the directory you want as the workspace.
```

Run `nimby create` in the directory you want as the workspace root. Nimby walks
up from wherever you are to find it, so you can still work from any
subdirectory.

The marker line in `nim.cfg` is now `# Managed by Nimby`. Workspaces marked
`# Created by Nimby` are still recognized, so there is nothing to migrate.

**Migrating CI.** If your workflow checks out a repository and then runs
`nimby install` in the checkout root, it will now fail. That pattern was
quietly wrong before: Nimby installs packages as siblings of the workspace
root, so it was writing package directories into your repository. Create the
workspace outside the checkout instead:

```yaml
- uses: actions/checkout@v5
- uses: treeform/setup-nim-action@v6

- name: Create a Nimby workspace
  shell: bash
  run: |
    mkdir -p "${{ runner.temp }}/workspace"
    cd "${{ runner.temp }}/workspace"
    nimby create

- name: Install dependencies
  shell: bash
  working-directory: ${{ runner.temp }}/workspace
  run: nimby install mypackage
```

Locally the same rule applies: run `nimby create` in the directory you want as
the workspace root, with your project as one of its children.

#### `-g` no longer touches your local `nim.cfg`

Global installs could previously create or modify a `nim.cfg` in the current
directory, including inside a Git checkout. They now leave the local workspace
alone.

### Added

#### `nimby create`

Creates a workspace in the current directory. Idempotent, and prepends the
marker to an existing `nim.cfg` rather than overwriting it.

#### Compiler subcommands: `c`, `cpp`, `js`, `e`, `doc`, `check`

Each verifies `nimby.lock` before handing off to nim. Every locked dependency
must be present, free of uncommitted changes, and sitting on the locked commit:

```
Dependency `vmath` not found in workspace.
Dependency `pixie` has uncommitted changes.
Dependency `chroma` is not at the locked commit.
```

Extra arguments forward straight through, so `nimby c -d:release src/app.nim`
does what you expect. A verified reproducible build is now one command instead
of a lock check plus a compile.

#### Install several packages at once

`nimby install a b c` and `nimby install a,b,c` both work.

### Fixed

#### Intermittent `Can't add package's tree to config`

A package required by URL in one `.nimble` and by bare name in another was
queued twice under two different keys, so two workers cloned into the same
directory. When the second arrived mid-clone it found the directory present but
the `.nimble` file not yet checked out, and Nimby exited 1. Jobs are now
identified by package name, so this cannot happen. In practice it also means
packages like `bitty` and `bumpy` are no longer installed twice on every run.

#### Interrupted installs no longer poison a workspace

Clones are staged in a temporary directory and moved into place only once
complete. Killing an install mid-clone used to leave a half-populated directory
that made every later install in that workspace fail until it was deleted by
hand.

#### URLs containing `~`, `=`, `<`, `>` or `^`

`requires "file://C:\Users\RUNNER~1\...\pkgs/dep"` parsed as the name
`file://C:\Users\RUNNER` with version operator `~`. Version operator characters
inside a URL spec are now part of the URL. Bare names like `bitty ~= 0.1.4`
parse as before.

### Internal

- The functional suite runs on Linux X64, Linux ARM64, macOS ARM64 and Windows
  X64. It was Linux-only, and four Windows-specific test harness bugs were
  hiding behind that.
- `name` and `spec` are used consistently throughout: a spec is a requirement as
  written (url, url#branch, lock line, `.nimble` path, bare name), a name is the
  normalized package name and the directory it installs into.
- Added `docs/new_nimby_process.md`, the release runbook this release follows.
