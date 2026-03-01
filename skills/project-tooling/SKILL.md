---
name: project-tooling
description: ESLint, Prettier, and Ruff config patterns for full-stack TypeScript + Python projects. Use when setting up a new repo, configuring CI static analysis, or aligning AI agent code output with project style. Covers Prettier/ESLint conflict resolution, Ruff rule selection, and the reasoning behind each decision.
---

# Project Tooling — ESLint, Prettier, Ruff

Standardised tooling config so that every contributor — human or AI agent — produces code that passes CI on the first try. Derived from production TravelAgent repo.

## Core Principle

Split concerns cleanly:
- **Prettier** owns all formatting (indentation, quotes, line length, commas)
- **ESLint** owns logic and correctness (no-unused-vars, no-explicit-any, eqeqeq)
- **Ruff** owns Python formatting + lint in one pass
- **Never let ESLint and Prettier conflict** — use `eslint-config-prettier` to disable all ESLint formatting rules

## Prettier

### Config (`.prettierrc` at repo root — covers frontend AND any markdown/JSON/config files)

```json
{
  "semi": true,
  "singleQuote": false,
  "quoteProps": "as-needed",
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "endOfLine": "lf",
  "arrowParens": "always",
  "bracketSpacing": true
}
```

**Root-level, not in `frontend/`** — covers any future markdown, JSON, or config files anywhere in the monorepo.

### Key decisions
| Setting | Why |
|---|---|
| `"semi": true` | Explicit semicolons; avoids ASI edge cases |
| `"singleQuote": false` | Double quotes; matches JSX convention |
| `"trailingComma": "all"` | Cleaner git diffs — adding a param = 1 line changed, not 2 |
| `"printWidth": 100` | 80 is too narrow for modern monitors; 120 too wide for side-by-side diffs |
| `"endOfLine": "lf"` | Prevents Windows CRLF contamination in CI |

### `.prettierignore`
```
node_modules/
dist/
build/
.next/
*.lock
```

## ESLint (Frontend — TypeScript/React)

### Install
```bash
bun add -D eslint @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  eslint-config-prettier eslint-plugin-react eslint-plugin-react-hooks
```

### `eslint.config.js` (flat config)
```js
import tsPlugin from '@typescript-eslint/eslint-plugin';
import tsParser from '@typescript-eslint/parser';
import prettierConfig from 'eslint-config-prettier';

export default [
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: { parser: tsParser, parserOptions: { project: './tsconfig.json' } },
    plugins: { '@typescript-eslint': tsPlugin },
    rules: {
      // Logic rules (Prettier handles formatting)
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_', varsIgnorePattern: '^_' }],
      '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
      'eqeqeq': ['error', 'always'],
      'curly': 'error',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
    },
  },
  prettierConfig,  // MUST be last — disables all formatting rules
];
```

### Key decisions
| Rule | Setting | Why |
|---|---|---|
| `no-explicit-any` | warn (not error) | AI agents sometimes need `any` during scaffolding; error blocks CI unnecessarily |
| `no-unused-vars` | error + `_` prefix escape | `_unused` signals intentional skip without triggering the rule |
| `consistent-type-imports` | error | Keeps type imports separate; helps tree-shaking |
| `no-console` | warn, allow warn/error | Console.log in prod = noise; warn/error = legitimate |
| `prettierConfig` last | required | Disables ESLint formatting rules that would conflict with Prettier |

## Ruff (Backend — Python)

### Install
```bash
uv add --dev ruff
```

### `pyproject.toml`
```toml
[tool.ruff]
line-length = 100
target-version = "py312"
exclude = [".venv", "dist", "__pycache__"]

[tool.ruff.lint]
# Explicit opt-in — never use "ALL" (breaks when ruff adds new rules)
select = [
  "E",     # pycodestyle errors
  "W",     # pycodestyle warnings
  "F",     # pyflakes (undefined names, unused imports)
  "I",     # isort (import ordering)
  "B",     # flake8-bugbear (common pitfalls)
  "UP",    # pyupgrade (auto-modernize syntax: f-strings, walrus, etc.)
  "SIM",   # flake8-simplify (unnecessary code patterns)
  "T20",   # no print() in production code
  "PTH",   # prefer pathlib over os.path
]
ignore = [
  "ERA001",  # commented-out code — too many false positives on section headers
  "B008",    # do not perform function calls in default arguments (FastAPI Depends pattern)
]

[tool.ruff.lint.per-file-ignores]
"models/sse.py" = ["N815"]  # camelCase SSE models match frontend contract — intentional
"services/timeline/loader.py" = ["PTH"]  # legacy os.path — low priority refactor

[tool.ruff.lint.isort]
known-first-party = ["backend", "services", "tools", "models", "utils"]
force-sort-within-sections = true

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"
```

### Run
```bash
ruff check .          # lint
ruff check . --fix    # lint + auto-fix
ruff format .         # format (replaces black)
```

### Key decisions
| Decision | Why |
|---|---|
| Explicit `select` over `"ALL"` | Avoids surprise CI breakage when ruff adds new rules — you opt in deliberately |
| `ERA001` ignored globally | Section header comments like `# PostgreSQL (asyncpg)` are not commented-out code |
| `B008` ignored | FastAPI's `Depends()` in default args is the canonical pattern; bugbear incorrectly flags it |
| `per-file-ignores` for camelCase SSE models | SSE field names must match frontend contract — enforcing snake_case would break the API |
| `known-first-party` in isort | Prevents ruff from treating your own packages as third-party |

## CI Integration

```yaml
# .github/workflows/ci.yml
- name: Lint frontend
  run: bun eslint . --max-warnings 0

- name: Check formatting
  run: bun prettier --check .

- name: Lint + format backend
  run: |
    ruff check .
    ruff format --check .
```

## Git Workflow for Feature Branches

When you need to merge a feature branch into main before adding new work:

```bash
# Clean approach: stash → merge → pop → commit new work
git stash                    # stash uncommitted changes
git merge feature/logging    # fast-forward if branch is ahead; no merge commit
git stash pop                # restore your in-progress changes on top
git add -p                   # stage selectively
git commit -m "feat: ..."
```

**Separate commits by concern:**
```
feat: add structured logging (structlog)
chore: add prettier config
chore: add eslint + ruff static analysis
```
Each commit independently revertible. Don't bundle tooling + feature work.

## Repo Structure (TravelAgent pattern)

```
/                          ← monorepo root
  .prettierrc              ← covers all files in repo
  .prettierignore
  frontend/
    eslint.config.js
    tsconfig.json
    src/
      app/                 ← Next.js app router
      components/
      hooks/
      types/
  backend/
    pyproject.toml         ← ruff + coverage config
    main.py                ← FastAPI entry, SSE endpoint
    agent.py               ← ADK runner, root_agent
    config.py              ← settings via pydantic-settings
    models/
      sse.py               ← SSE event types (camelCase to match frontend)
    services/
      session_store/       ← nested package: __init__.py + cache.py + helpers.py
      timeline/            ← nested package: __init__.py + analyzer.py + loader.py
      user_context.py      ← simple module
    tools/
      search_places.py
      search_flights.py
    utils/
      logging.py           ← structlog setup (see python-structlog skill)
```
