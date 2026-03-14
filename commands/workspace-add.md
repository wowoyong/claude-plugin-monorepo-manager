# /workspace-add Command

Add a new package or application to the monorepo.

## Usage

```
/workspace-add [name] [--type app|package] [--template <template>] [--scope <scope>]
```

## Arguments

- `name` (optional): Name of the new package/app. If omitted, prompt the user.
- `--type` (optional): Either `app` or `package`. Defaults to `package`.
- `--template` (optional): Template to use. If omitted, infer from existing packages.
- `--scope` (optional): npm scope (e.g., `@myorg`). If omitted, infer from existing packages.

## Workflow

### Step 1: Detect Monorepo Type

Run detection as described in SKILL.md Phase 1. You need to know:
- The primary orchestrator (Turborepo, Nx, Lerna, etc.)
- The workspace manager (pnpm, yarn, npm)
- The workspace glob patterns (to know where to place the new package)
- The existing package scope (e.g., `@myorg`)

### Step 2: Gather Information

If not provided via arguments, ask the user:

1. **Name**: What should the package be called?
2. **Type**: Is this an `app` (deployable application) or a `package` (shared library)?
3. **Description**: Brief description of the package purpose.
4. **Internal dependencies**: Which existing internal packages should it depend on?

### Step 3: Determine Conventions

Analyze 1-2 existing packages of the same type to determine:
- Directory structure (e.g., `src/`, `lib/`, `test/`, `__tests__/`)
- TypeScript configuration (tsconfig extends pattern)
- Build tool (tsc, tsup, vite, esbuild, rollup, webpack)
- Test framework (jest, vitest, mocha)
- Linting setup (extends shared config or local)
- Script names and patterns
- Package.json field conventions (main, module, types, exports)
- Package version (fixed versioning or independent)

### Step 4: Scaffold

Create the following files, matching conventions found in Step 3:

#### For a `package` type:

```
<workspace-dir>/<name>/
├── package.json
├── tsconfig.json
├── src/
│   └── index.ts
├── test/          (if test dir convention exists)
│   └── index.test.ts
└── README.md      (minimal, only if other packages have one)
```

**package.json** must include:
```json
{
  "name": "<scope>/<name>",
  "version": "<matching version strategy>",
  "private": false,
  "description": "<user-provided description>",
  "main": "./dist/index.js",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.js",
      "types": "./dist/index.d.ts"
    }
  },
  "scripts": {
    "build": "<matching build command>",
    "dev": "<matching dev command>",
    "test": "<matching test command>",
    "lint": "<matching lint command>",
    "clean": "rm -rf dist"
  },
  "dependencies": {},
  "devDependencies": {}
}
```

Adjust `main`, `module`, `types`, and `exports` to match the existing convention. If packages use `tsup`, use tsup conventions. If they use plain `tsc`, use tsc conventions.

**tsconfig.json**:
```json
{
  "extends": "<root tsconfig path>",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"]
}
```

**src/index.ts**:
```typescript
export {};
```

#### For an `app` type:

Follow the same pattern but:
- Set `"private": true` in package.json
- Place under the `apps/` directory (or equivalent)
- Include app-specific scripts (e.g., `start`, `dev`, `build`)
- If a framework is common (Next.js, Remix, Express), scaffold accordingly

### Step 5: Register and Install

1. **Check workspace globs**: Verify the new package location matches existing workspace globs. If not, inform the user that the workspace config needs updating.

2. **Tool-specific registration**:
   - **Nx**: Create `project.json` if the monorepo uses explicit project configuration
   - **Lerna**: Verify `lerna.json` packages array includes the new location
   - **Turborepo**: Usually automatic, but verify `turbo.json` pipeline covers the new package's scripts

3. **TypeScript project references**: If the monorepo uses TypeScript project references (`references` in root tsconfig), add the new package.

4. **Install dependencies**: Run the appropriate install command:
   - pnpm: `pnpm install`
   - yarn: `yarn install` or `yarn`
   - npm: `npm install`

### Step 6: Add Internal Dependencies

If the user specified internal dependencies:
1. Add them using the workspace protocol:
   - pnpm: `"<dep>": "workspace:*"`
   - yarn berry: `"<dep>": "workspace:*"`
   - yarn classic: `"<dep>": "*"` (resolved via workspaces)
   - npm: `"<dep>": "*"` (resolved via workspaces)
2. Run install again to link them

### Step 7: Verify

1. Run the build command for the new package to verify it compiles
2. Run lint if configured
3. Report success with a summary:

```
Successfully created <scope>/<name>

Location: <full path>
Type: <app|package>
Internal Dependencies: <list or none>

Next steps:
- Start developing in <path>/src/
- Add internal deps: <package manager> add <dep> --filter <name>
- Build: <package manager> run build --filter <name>
```

## Error Handling

- If a package with the same name already exists, abort and inform the user
- If the workspace glob does not cover the target directory, warn and suggest updating the workspace config
- If the build fails after scaffolding, show the error and suggest fixes
- Never overwrite existing files without explicit confirmation
