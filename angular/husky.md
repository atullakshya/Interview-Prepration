
# Implementing Husky in Angular (Step-by-Step)

This guide explains how to set up Husky in an Angular project so Git hooks automatically run quality checks before commit and push.

## Why use Husky?

Husky helps teams enforce code quality locally by running checks like:
- lint
- unit tests
- formatting checks
- commit message validation (optional)

This reduces broken builds and bad commits in shared branches.

## Prerequisites

- Node.js installed
- Angular project already initialized
- Git repository initialized

## 1) Install Husky

Run this inside your Angular project root (where `package.json` exists):

```bash
npm install --save-dev husky
```

## 2) Add prepare script

Add this script to `package.json`:

```json
{
  "scripts": {
    "prepare": "husky"
  }
}
```

Then run:

```bash
npm run prepare
```

This creates a `.husky` folder in your project.

## 3) Create a pre-commit hook

Create hook:

```bash
npx husky add .husky/pre-commit "npm run lint"
```

If your Husky version uses the newer pattern and `add` is unavailable, create the hook file manually:

```bash
mkdir -p .husky
printf "#!/usr/bin/env sh\nnpm run lint\n" > .husky/pre-commit
chmod +x .husky/pre-commit
```

### Recommended pre-commit content

```sh
#!/usr/bin/env sh

npm run lint
npm run test -- --watch=false --browsers=ChromeHeadless
```

Use only the checks your team can run quickly. Keep pre-commit fast.

## 4) Create a pre-push hook (optional)

Use pre-push for heavier checks:

```bash
npx husky add .husky/pre-push "npm run test -- --watch=false --browsers=ChromeHeadless"
```

Manual file content alternative:

```sh
#!/usr/bin/env sh

npm run build
npm run test -- --watch=false --browsers=ChromeHeadless
```

## 5) Add lint-staged for staged-file checks (recommended)

This improves speed by running linters only on staged files.

Install:

```bash
npm install --save-dev lint-staged
```

Add to `package.json`:

```json
{
  "lint-staged": {
    "*.{ts,html,scss,css}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md}": [
      "prettier --write"
    ]
  }
}
```

Update `.husky/pre-commit`:

```sh
#!/usr/bin/env sh

npx lint-staged
```

## 6) Example Angular scripts in package.json

Make sure these scripts exist (adjust to your project):

```json
{
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint",
    "prepare": "husky"
  }
}
```

## 7) Verify Husky setup

1. Stage a file change.
2. Run commit:

```bash
git add .
git commit -m "test husky hooks"
```

3. Confirm the hook runs.

If lint/test fails, commit should stop.

## 8) Team onboarding notes

- Keep hooks deterministic and fast.
- Avoid very heavy pre-commit checks.
- Move expensive checks to pre-push or CI.
- Document required local tooling (ChromeHeadless, ESLint, Prettier, etc.).

## 9) Troubleshooting

### Hook not running

- Confirm `.husky/pre-commit` exists and is executable.
- Confirm `npm run prepare` has been run.
- Confirm `.git` folder exists in the project root.

### Command not found in hook

Use `npx` in hook scripts:

```sh
#!/usr/bin/env sh

npx ng lint
npx ng test --watch=false --browsers=ChromeHeadless
```

### Hooks too slow

- Use `lint-staged`.
- Move full test/build to pre-push.
- Keep pre-commit focused on staged-file checks.

## 10) Suggested enterprise pattern

- `pre-commit`: `lint-staged`
- `pre-push`: unit tests + build
- CI pipeline: full lint + test + build + security checks

This gives fast local feedback and strong quality gates in CI.
