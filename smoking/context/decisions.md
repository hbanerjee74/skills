# Decisions

## D1: Skill Structure — Progressive Disclosure
Use progressive disclosure: SKILL.md covers the 80% use case (common patterns, quick-start templates), reference files provide depth for advanced scenarios. Engineers should be productive immediately from SKILL.md alone.

## D2: Target Audience — Intermediate Engineers
Assume intermediate expertise: familiar with language basics and common frameworks, but needs guidance on architectural patterns, performance optimization, and edge cases. Do not explain basic syntax or standard library usage.

## D3: Code Examples — Copy-Paste Ready
Include production-ready code templates that engineers can copy and adapt. All examples include error handling, logging, and configuration. No pseudo-code — use the actual language/framework syntax.

## D4: Architecture — Pattern Decision Matrix
Present architectural choices as decision matrices with clear criteria: team size (<5, 5-20, >20), data volume (<100K, 100K-10M, >10M rows), latency requirements (<100ms, <1s, <10s). Each cell recommends a specific pattern.

## D5: Testing — Tiered Strategy
Recommend tiered testing: unit tests for business logic (>80% coverage), integration tests for API contracts, E2E tests for critical user flows only. Include specific testing patterns for each tier with copy-paste templates.

## D6: Error Handling — Result Types + Recovery
Use result types for expected errors with explicit recovery paths. Each error pattern includes: detection (how to identify), impact (what breaks), recovery (how to fix), and prevention (how to avoid). No generic try-catch blocks.

## D7: Performance — Thresholds + Optimization Sequence
Provide concrete performance thresholds: response time p95 < 200ms, throughput > 1000 req/s, error rate < 0.1%. Include an optimization sequence: measure first, identify bottleneck, apply targeted fix, validate improvement.

## D8: Security — Defense in Depth
Layer security controls: input validation at boundaries, authentication/authorization middleware, secrets in environment variables (never in code), dependency scanning in CI. Include security checklist for code reviews.

## D9: Monitoring — Key Metrics + Alerts
Track four golden signals: latency, traffic, errors, saturation. Set alert thresholds at 2x normal baseline. Include dashboard templates and runbook patterns for common incidents.

## D10: Documentation — Architecture Decision Records
Use ADRs (Architecture Decision Records) for significant decisions. Format: context, decision, consequences, alternatives considered. Store alongside code in docs/adr/ directory.

## D11: Deployment — Blue-Green with Automated Rollback
Use blue-green deployments with health checks. Automated rollback if error rate exceeds 1% in first 5 minutes. Canary releases for high-risk changes (10% traffic for 30 minutes before full rollout).

## D12: Configuration — Environment-Aware Defaults
Environment variables for secrets and environment-specific values. Config files with sensible defaults for development. Validate configuration at startup with clear error messages for missing required values.
