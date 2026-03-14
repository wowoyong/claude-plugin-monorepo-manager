# /affected Command

Analyze which packages are affected by recent changes and recommend actions.

## Usage

```
/affected [--base <branch>] [--head <ref>] [--action build|test|lint|publish] [--json]
```

## Arguments

- `--base` (optional): Base branch to compare against. Defaults to `main`, falls back to `master`.
- `--head` (optional): Head ref to compare. Defaults to `HEAD`.
- `--action` (optional): If specified, run this action only for affected packages.
- `--json` (optional): Output results as JSON for scripting.

## Workflow

### Step 1: Detect Monorepo Type

Run detection as described in SKILL.md Phase 1. Determine:
- The orchestrator tool available
- The workspace structure and package locations

### Step 2: Check for Native Affected Commands

If the monorepo has a native affected detection mechanism, prefer it:

- **Turborepo**: Use `npx turbo run build --dry-run --filter=...[<base>]` to list affected
- **Nx**: Use `npx nx affected --base=<base> --head=<head> --print-affected` to list affected
- **Lerna**: Use `npx lerna changed --since <base>` to list affected

If native tools are available, run them and skip to Step 5 (reporting).

### Step 3: Git-Based Change Detection (Fallback)

If no native tool is available, or for deeper analysis:

1. **Verify git availability**:
   ```bash
   git rev-parse --is-inside-work-tree
   ```

2. **Determine base branch**:
   ```bash
   git branch -r | grep -E 'origin/(main|master)$' | head -1
   ```
   Use the `--base` argument if provided.

3. **Get changed files**:
   ```bash
   git diff --name-only <base>...HEAD
   ```
   If there are uncommitted changes, also include:
   ```bash
   git diff --name-only HEAD
   git diff --name-only --cached
   ```

4. **Map files to packages**:
   For each changed file, determine which package it belongs to by matching against workspace directory patterns. Files at the root or in shared config directories affect ALL packages.

### Step 4: Dependency-Aware Affected Calculation

1. **Build the internal dependency graph**:
   - Read all package.json files
   - Map which packages depend on which internal packages

2. **Seed the affected set** with directly changed packages

3. **Propagate through dependents**:
   ```
   affected = set(directly_changed_packages)
   queue = list(directly_changed_packages)
   while queue is not empty:
       pkg = queue.pop()
       for dependent in packages_that_depend_on(pkg):
           if dependent not in affected:
               affected.add(dependent)
               queue.append(dependent)
   ```

4. **Handle root-level changes**:
   If any of these files changed, ALL packages are affected:
   - `package.json` (root)
   - `tsconfig.base.json` / `tsconfig.json` (root)
   - `.eslintrc.*` / `eslint.config.*` (root)
   - `turbo.json` / `nx.json` / `lerna.json`
   - `pnpm-workspace.yaml`
   - `.github/workflows/*` (CI config)
   - Lock files (`pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`)

### Step 5: Report Results

#### Standard Output

```
Affected Packages Analysis
==========================
Base: origin/main
Head: HEAD (abc1234)
Changed files: N

Directly Changed (M packages):
  - @scope/package-a        (5 files changed)
  - @scope/package-b        (2 files changed)

Transitively Affected (K packages):
  - @scope/app-web           <- depends on @scope/package-a
  - @scope/package-c         <- depends on @scope/package-b

Not Affected (J packages):
  - @scope/package-d
  - @scope/package-e

Recommended Commands:
  Build affected:  <pm> run build --filter=@scope/package-a --filter=@scope/package-b --filter=@scope/app-web --filter=@scope/package-c
  Test affected:   <pm> run test --filter=@scope/package-a --filter=@scope/package-b --filter=@scope/app-web --filter=@scope/package-c
  Lint affected:   <pm> run lint --filter=@scope/package-a --filter=@scope/package-b --filter=@scope/app-web --filter=@scope/package-c
```

Adapt the filter syntax to the monorepo tool:
- **Turborepo**: `--filter=<pkg>`
- **Nx**: `--projects=<pkg>` or use `nx affected`
- **pnpm**: `--filter=<pkg>`
- **yarn**: `yarn workspace <pkg> run <script>`
- **Lerna**: `--scope=<pkg>`

#### JSON Output (when --json is specified)

```json
{
  "base": "origin/main",
  "head": "abc1234",
  "changedFiles": 7,
  "directlyChanged": [
    { "name": "@scope/package-a", "filesChanged": 5 }
  ],
  "transitivelyAffected": [
    { "name": "@scope/app-web", "reason": "depends on @scope/package-a" }
  ],
  "notAffected": ["@scope/package-d"],
  "commands": {
    "build": "<full build command>",
    "test": "<full test command>",
    "lint": "<full lint command>"
  }
}
```

### Step 6: Execute Action (if --action specified)

If the user passed `--action`, run the appropriate command for affected packages only:

1. Build the filter/scope arguments for all affected packages
2. Run the command:
   ```bash
   <pm> run <action> <filter-flags>
   ```
3. Report results (pass/fail per package)

## Special Cases

### No Changes Detected
```
No changes detected between <base> and <head>.
All packages are up to date.
```

### All Packages Affected
```
Root-level configuration changed — all packages are affected.
Changed root files: <list>
```

### No Git Repository
```
Warning: Not inside a git repository. Cannot determine affected packages automatically.
Please specify which packages you want to check, or initialize git first.
```

## Performance Notes

- For large monorepos (50+ packages), reading all package.json files may take a moment. The command caches the dependency graph in memory for the session.
- Native tools (Turborepo, Nx) are significantly faster for affected detection — always prefer them when available.
- The `--dry-run` flags used for native tools do not execute any build steps; they only compute the affected graph.
