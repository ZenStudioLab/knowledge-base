# CI/CD

Pipelines, environments, releases, deployments, incidents.

## Subfolders

- `github-actions/` - workflow definitions
- `deployment/` - deployment processes
- `environments/` - dev, staging, prod configs
- `release-process/` - release procedures
- `incident-notes/` - post-mortems and lessons

## Rules

- Keep workflows in `github-actions/`
- Document environment differences
- Include rollback procedures
- Log incidents for future reference
