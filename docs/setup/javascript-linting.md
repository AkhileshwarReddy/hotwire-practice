# JavaScript & CSS Linting Setup (ESLint + Prettier)

## What it is

[ESLint](https://eslint.org/) lints JavaScript; [Prettier](https://prettier.io/)
formats JS/CSS (and more). This app uses **importmap-rails**, so there is **no
Node bundler** — JS lives in `app/javascript/` and is served as-is. We add
ESLint/Prettier purely as **dev tooling** (a `package.json` with `devDependencies`
only), not as a build step.

## Why we use it

- Keep Stimulus controllers and other JS in `app/javascript/` consistent.
- Prettier removes formatting debates; ESLint catches real bugs (unused vars,
  `==` vs `===`, etc.).

## Prerequisites

- Node.js + npm on your machine (or use the `web` container from
  `docker-compose.md`, which installs Node).

## Install

```bash
npm init -y
npm install --save-dev eslint @eslint/js globals prettier eslint-config-prettier
```

> `eslint-config-prettier` turns off ESLint rules that would fight Prettier.

## Configuration

### 1. `package.json` scripts

Add a `scripts` block (and mark the package private):

```json
{
  "name": "hotwire-practice",
  "private": true,
  "type": "module",
  "scripts": {
    "lint": "eslint app/javascript",
    "lint:fix": "eslint app/javascript --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "devDependencies": {
    "@eslint/js": "*",
    "eslint": "*",
    "eslint-config-prettier": "*",
    "globals": "*",
    "prettier": "*"
  }
}
```

### 2. `eslint.config.js` (flat config, ESLint 9+)

```js
import js from "@eslint/js";
import globals from "globals";
import prettier from "eslint-config-prettier";

export default [
  js.configs.recommended,
  prettier,
  {
    files: ["app/javascript/**/*.js"],
    languageOptions: {
      ecmaVersion: "latest",
      sourceType: "module",
      globals: {
        ...globals.browser,
        Stimulus: "readonly",
        Turbo: "readonly",
      },
    },
    rules: {
      "no-unused-vars": "warn",
      "no-console": ["warn", { allow: ["warn", "error"] }],
    },
  },
  {
    ignores: ["vendor/**", "public/**", "node_modules/**", "coverage/**"],
  },
];
```

### 3. `.prettierrc`

```json
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### 4. `.prettierignore`

```gitignore
vendor/
public/assets/
node_modules/
coverage/
tmp/
log/
db/schema.rb
```

### 5. Git-ignore Node output

Append to `.gitignore`:

```gitignore
/node_modules
```

> Commit `package.json` **and** `package-lock.json` so CI installs identical
> versions.

## How to run

```bash
npm run lint          # report JS problems
npm run lint:fix      # autofix JS
npm run format        # rewrite files with Prettier
npm run format:check  # verify formatting (used in CI)
```

## How to verify

- `npm run lint` exits `0` on a clean tree.
- `npm run format:check` prints
  `All matched files use Prettier code style!`.
- Add `var x=1` to a controller → `npm run lint` flags it; `npm run format`
  reformats it.

## Common gotchas

- **ESLint 9 uses flat config** (`eslint.config.js`), not the old
  `.eslintrc.*`. Don't mix the two.
- **importmap globals**: `Turbo`, `Stimulus`, and browser globals must be
  declared (as above) or ESLint reports `no-undef`.
- **Prettier vs RuboCop/erb_lint**: Prettier does **not** format `.erb` well —
  keep ERB under erb_lint and add `*.erb` to `.prettierignore` if it complains.
- On CI, run `npm ci` (not `npm install`) for reproducible installs.

## Hands-on checklist

- [ ] `npm init -y` then install eslint, @eslint/js, globals, prettier, eslint-config-prettier
- [ ] Add `scripts` + `private: true` + `type: module` to `package.json`
- [ ] Create `eslint.config.js`, `.prettierrc`, `.prettierignore`
- [ ] Add `/node_modules` to `.gitignore`
- [ ] Run `npm run lint` and `npm run format:check`
- [ ] Verify: both pass; commit `package.json` + `package-lock.json`; wire into CI (see `github-ci.md`)
