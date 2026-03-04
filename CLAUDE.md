# Polivote

Campaign vote tracking, counting, and voter engagement software.

## Tech Stack

(Update this section as the stack is chosen)

## Build & Test Commands

(Update as commands are established)
- Install: `npm install`
- Build: `npm run build`
- Test: `npm test`
- Lint: `npm run lint`
- Type check: `npm run type-check`

## Code Style

- Use ES modules (`import`/`export`), not CommonJS
- 2-space indentation
- Prefer `const` over `let`; avoid `var`
- Use TypeScript strict mode
- Destructure imports where possible
- Name files in kebab-case; components in PascalCase

## Architecture

(Update as project structure takes shape)
- Keep a clear separation between data layer, business logic, and presentation
- Colocate tests with source files (`*.test.ts` next to `*.ts`)

## Git Workflow

- Branch naming: `feature/<topic>`, `fix/<topic>`, `chore/<topic>`
- Write clear commit messages: imperative mood, explain *why* not *what*
- Run tests before committing
- Keep PRs focused — one concern per PR

## Important Notes

- IMPORTANT: Never hardcode secrets, API keys, or credentials. Use environment variables.
- IMPORTANT: All user input must be validated and sanitized server-side.
- IMPORTANT: Use parameterized queries for all database operations — no string concatenation in SQL.
- This is election/voting software — correctness and auditability are paramount. Prefer explicit, readable code over clever abstractions.
- All vote-counting logic must be deterministic and produce identical results on re-run.
