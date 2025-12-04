# ECSO Platform

This repository contains the ECSO (ECS Operator) configuration for managing ECS deployments.

## Structure

```
ecso/
├── defaults.yaml           # Shared defaults for all environments
└── environments/
    └── dev.yaml            # Development environment configuration
```

## Services

| Service | Description | Port |
|---------|-------------|------|
| test-app | Demo application | 8080 |

## Environments

| Environment | Cluster | Auto-Sync | ALB DNS |
|-------------|---------|-----------|---------|
| dev | ecso-cluster | Yes | ecso-alb-216761709.us-east-1.elb.amazonaws.com |

## Usage

### Validate Configuration

```bash
ecso validate -c ./ecso
```

### Plan Changes

```bash
ecso plan dev -c ./ecso
```

### Apply Changes

```bash
ecso apply dev -c ./ecso
```

## Related Repositories

- [ecso-test-app](https://github.com/ivan-bax/ecso-test-app) - Test application
- [ecso](https://github.com/ivan-bax/ecso) - ECSO tool (coming soon)
