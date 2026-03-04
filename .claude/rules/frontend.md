---
paths:
  - "src/components/**/*"
  - "src/pages/**/*"
  - "src/app/**/*"
---

# Frontend Rules

- Use functional components with hooks
- Type all props with TypeScript interfaces
- Keep components small and focused — extract when a component exceeds ~150 lines
- Accessibility is required: semantic HTML, ARIA labels, keyboard navigation
- Forms must show clear validation errors inline
- Use loading and error states for all async operations
- Never store sensitive data (tokens, PII) in localStorage — use httpOnly cookies
