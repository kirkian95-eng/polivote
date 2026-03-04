# Testing Rules

- Every new feature or bug fix must include tests
- Colocate test files with source: `foo.ts` → `foo.test.ts`
- Test behavior and outcomes, not implementation details
- Vote-counting and tabulation logic requires 100% branch coverage
- Use descriptive test names: `it("rejects duplicate votes from same voter")`
- Prefer unit tests for business logic; integration tests for API endpoints
- Mock external services (email, SMS) — never call real services in tests
- Tests must be deterministic — no reliance on wall-clock time or random data
- Run the full test suite before opening a PR
