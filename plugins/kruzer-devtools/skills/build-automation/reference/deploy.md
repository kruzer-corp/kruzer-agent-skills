# Deploy

## How Deployment Works

There is no separate deploy command. Deployment is **triggered by a git commit on the configured branch**.

```
git commit → git push → Kruzer platform detects the change
                        → builds and deploys automatically
```

The platform monitors a specific branch (e.g., `main`). When a commit lands on that branch, the deployment runs automatically.

## Workflow

```bash
# Develop and test locally
krz run my-automation -p ./params.json

# Build before committing (required)
npm run build

# Commit and push
git add automations/my-automation.automation.ts src/...
git commit -m "feat: add my-automation"
git push origin main   # triggers deploy on the platform
```

The build step is mandatory before every commit. Committing without running the build will result in runtime errors on the platform.

## Branch Configuration

The monitored branch is configured in the platform per repository. Common conventions:

| Branch | Environment |
|---|---|
| `main` | Production |
| `staging` or `homolog` | Staging / Homologation |
| `develop` | Development |

Check with your platform admin which branch maps to which environment.

## What the Platform Builds

On each detected commit, the platform:

1. Installs dependencies from `package.json` using the versions already pinned there
2. Compiles TypeScript
3. Deploys all files in `automations/` as executable entrypoints

## Package Dependencies

**You cannot add external npm packages.** The pods that execute automations use only the packages already present in the original `package.json`. Adding new entries to `package.json` does not cause the pods to install them.

Only use:
- `@kruzer/idk` — the official SDK, already included
- Native Node.js modules (`path`, `fs`, `crypto`, etc.)
- TypeScript standard library types

Do not add `axios`, `zod`, `joi`, `lodash`, or any other external dependency — they will not be available at runtime.

## Key Points

- Run `npm run build` before every commit — mandatory.
- Deploy = commit to the configured branch. No `krz deploy` command exists.
- Dependencies are frozen at the original `package.json` — do not add new packages.
- All files in `automations/` are picked up automatically — no registration step needed.
- TypeScript compilation errors will cause the deploy to fail.
