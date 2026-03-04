---
paths:
  - "src/api/**/*"
  - "src/routes/**/*"
---

# API Design Rules

- Use RESTful conventions: nouns for resources, HTTP verbs for actions
- Return consistent JSON response shape: `{ data, error, meta }`
- Use appropriate HTTP status codes (201 for creation, 404 for not found, 422 for validation errors)
- Paginate list endpoints by default
- Include request validation middleware on all mutating endpoints
- Document endpoints with OpenAPI/Swagger comments
- Version the API from day one (`/api/v1/`)
- Return meaningful error messages — never expose stack traces in production
