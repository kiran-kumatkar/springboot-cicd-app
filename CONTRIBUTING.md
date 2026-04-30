# Contributing

## How to Run Locally

1. Clone the repo
2. Follow the infrastructure setup in README.md
3. Make your changes in a feature branch
4. Push — Jenkins pipeline will validate your changes automatically

## Branch Strategy

| Branch | Purpose |
|---|---|
| `main` | Production — triggers full pipeline |
| `feature/*` | Feature development |
| `fix/*` | Bug fixes |

## Commit Message Format

```
type: short description

Examples:
feat: add new REST endpoint
fix: correct health check path
ci: update Jenkinsfile timeout
docs: update README screenshots
```

## Pipeline Rules

- All unit tests must pass
- SonarQube quality gate must pass
- No CRITICAL CVEs allowed (Trivy will block the build)
