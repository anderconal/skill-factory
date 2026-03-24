# Technical Stories Base

Stories técnicas comunes a todos los tipos de proyecto. Se incluyen automáticamente cuando se usa `--greenfield`.

> **Nota:** Estas stories son templates. El agente PM debe adaptarlas al contexto específico del proyecto.

## Stories

```yaml
stories:
  - ref: "tech-infra"
    title: "Infrastructure as Code setup"
    effort: 2
    area: "infrastructure"
    description: |
      As a development team,
      I want infrastructure defined as code,
      so that environments are reproducible and deployments are consistent.
    acceptanceCriteria:
      - "Infrastructure code in repository (Terraform/Pulumi/CDK)"
      - "At least one environment (dev/staging) provisioned from code"
      - "README documents how to provision new environments"
    adaptations:
      web: "Azure/AWS with App Service or container hosting"
      mobile: "Backend API infrastructure + CDN for assets"
      ai: "Compute instances with GPU support if needed"

  - ref: "tech-cicd-basic"
    title: "Basic CI/CD pipeline"
    effort: 2
    area: "ci-cd"
    description: |
      As a development team,
      I want automated build and deployment,
      so that users receive bug fixes and new features faster without manual intervention.
    acceptanceCriteria:
      - "Push to main branch triggers automatic deployment to staging"
      - "Build failures notify the team (email/Slack/Teams)"
      - "Deployment completes in under 10 minutes"
    verifiableValue: "Commit on main → deployed to staging automatically"

  - ref: "tech-cicd-tests"
    title: "Tests gate in CI pipeline"
    effort: 1
    area: "ci-cd"
    description: |
      As a development team,
      I want PRs blocked when tests fail,
      so that broken code doesn't reach production and users don't experience regressions.
    acceptanceCriteria:
      - "PR cannot be merged if tests fail"
      - "Test results visible in PR checks"
      - "At least unit tests running (integration tests optional for MVP)"
    verifiableValue: "PR blocked if tests fail"
    dependsOn: ["tech-cicd-basic"]

  - ref: "tech-telemetry"
    title: "Basic telemetry and error tracking"
    effort: 1
    area: "observability"
    description: |
      As a development team,
      I want errors automatically captured and reported,
      so that users experience fewer errors because we detect and fix issues before they spread.
    acceptanceCriteria:
      - "Unhandled exceptions sent to monitoring system (AppInsights/Sentry/Datadog)"
      - "Errors include stack trace and request context"
      - "Team receives alerts for error spikes"
    verifiableValue: "Force error → appears in monitoring system within 1 minute"

  - ref: "tech-traces"
    title: "Distributed tracing setup"
    effort: 1
    area: "observability"
    description: |
      As a development team,
      I want request traces visible end-to-end,
      so that we can diagnose slow requests and users experience faster load times.
    acceptanceCriteria:
      - "Each request has a correlation ID"
      - "Traces visible in tracing system (Jaeger/Zipkin/AppInsights)"
      - "Can see latency breakdown by service/component"
    verifiableValue: "Open tracing UI and see a real request from production"

  - ref: "tech-auth"
    title: "User authentication"
    effort: 2
    area: "auth"
    description: |
      As a user,
      I want to log in securely,
      so that my personal data is protected from unauthorized access.
    acceptanceCriteria:
      - "Login with identity provider (AAD/Auth0/Cognito/custom)"
      - "Session persists across browser refresh"
      - "Logout clears session completely"
    adaptations:
      web: "OAuth2/OIDC flow with secure token storage"
      mobile: "Biometric option + secure token storage (Keychain/Keystore)"

  - ref: "tech-authz"
    title: "Role-based authorization"
    effort: 3
    area: "auth"
    description: |
      As a user,
      I want access restricted based on my role,
      so that I only see features relevant to me and sensitive data is protected.
    acceptanceCriteria:
      - "At least 2 roles defined (e.g., user, admin)"
      - "Protected routes/endpoints check role before access"
      - "Unauthorized access returns 403, not 500"
    dependsOn: ["tech-auth"]
```

## Usage Notes

- Stories marked with `adaptations` should be customized based on `--type`
- Stories with `dependsOn` should have that dependency reflected in Jira
- `verifiableValue` is the specific outcome to check in production
- All effort values are in Fibonacci story points (1, 2, 3, 5, 8, 13)
