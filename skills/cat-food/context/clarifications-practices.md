# Practice Clarifications

## Development Workflow
1. **What branching strategy works best?**
   - Recommendation: Trunk-based development with short-lived feature branches.

2. **How should code reviews be structured?**
   - Recommendation: Focus on architectural concerns and correctness, not style.

3. **What CI/CD pipeline stages are essential?**
   - Recommendation: Lint, test, build, deploy-staging, deploy-production with manual gate.

## Testing Strategy
4. **What test pyramid ratio is recommended?**
   - Recommendation: 70% unit, 20% integration, 10% E2E.

5. **When should tests be written vs skipped?**
   - Recommendation: Always test business logic and edge cases. Skip purely cosmetic changes.

## Documentation
6. **What documentation is essential vs nice-to-have?**
   - Recommendation: API docs and architecture decisions are essential. Tutorials are nice-to-have.
