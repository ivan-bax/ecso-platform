# Teleport Zero Trust Implementation Plan for ECSO

## Overview

This document outlines the implementation plan for integrating Teleport as the zero trust access layer for the ECSO platform, providing secure access to:
- ECS services and containers
- RDS databases (PostgreSQL/MySQL)
- SSH access to infrastructure
- Web application access

## Architecture

```
                                    ┌─────────────────────────────┐
                                    │      Teleport Cluster       │
                                    │  (Auth + Proxy Services)    │
                                    │                             │
                                    │  ┌─────────────────────┐    │
                                    │  │   Auth Service      │    │
                                    │  │  - RBAC/ABAC        │    │
                                    │  │  - SSO Integration  │    │
                                    │  │  - Audit Logging    │    │
                                    │  └─────────────────────┘    │
                                    │                             │
                                    │  ┌─────────────────────┐    │
                                    │  │   Proxy Service     │    │
                                    │  │  - TLS Termination  │    │
                                    │  │  - Protocol Routing │    │
                                    │  └─────────────────────┘    │
                                    └─────────────┬───────────────┘
                                                  │
                    ┌─────────────────────────────┼─────────────────────────────┐
                    │                             │                             │
          ┌─────────┴─────────┐         ┌─────────┴─────────┐         ┌─────────┴─────────┐
          │   ECS Agent       │         │   DB Service      │         │   App Agent       │
          │   (Per Service)   │         │   (RDS Access)    │         │   (Web Apps)      │
          └─────────┬─────────┘         └─────────┬─────────┘         └─────────┬─────────┘
                    │                             │                             │
          ┌─────────┴─────────┐         ┌─────────┴─────────┐         ┌─────────┴─────────┐
          │   ECS Services    │         │   RDS Databases   │         │   ALB/Services    │
          │   - test-app      │         │   - main-db       │         │   - ECSO API      │
          │   - nginx         │         │   - future DBs    │         │   - Apps          │
          └───────────────────┘         └───────────────────┘         └───────────────────┘
```

## Implementation Phases

### Phase 4.1: Teleport Cluster Setup

**Objective**: Deploy Teleport cluster on ECS with Auth and Proxy services

**Tasks**:
1. Create Teleport ECS task definition
2. Set up DynamoDB backend for Teleport state
3. Configure S3 bucket for audit logs
4. Set up ACM certificate for Teleport domain
5. Configure ALB routing for Teleport

**Configuration (teleport.yaml)**:
```yaml
# New section in infrastructure/base.yaml
teleport:
  enabled: true
  domain: teleport.bax.dev

  cluster:
    name: ecso-teleport

  # State storage
  dynamodb:
    tableName: ecso-teleport-state

  # Audit logs
  auditLogs:
    s3Bucket: ecso-teleport-audit

  # Authentication
  auth:
    sso:
      # GitHub SSO integration
      github:
        enabled: true
        clientId: "${GITHUB_CLIENT_ID}"
        clientSecretRef: ecso/teleport/github-secret
        allowedOrganizations:
          - bax-dev
```

**AWS Resources Required**:
- DynamoDB table: `ecso-teleport-state`
- S3 bucket: `ecso-teleport-audit-085122410972`
- ACM certificate: `teleport.bax.dev`
- Security Group: `ecso-teleport-sg`
- IAM Role: `ecso-teleport-role`

### Phase 4.2: Database Access Service

**Objective**: Enable secure, audited access to RDS databases via Teleport

**Tasks**:
1. Deploy Teleport Database Service on ECS
2. Configure database agents for each RDS instance
3. Set up RBAC roles for database access
4. Configure automatic credential rotation

**ECSO Configuration Extension**:
```yaml
# environments/dev.yaml - extend databases section
databases:
  main-db:
    engine: postgres
    engineVersion: "15.10"
    # ... existing config ...

    # NEW: Teleport integration
    teleport:
      enabled: true
      allowedRoles:
        - db-admin    # Full access
        - db-readonly # Read-only access
      autoRotateCredentials: true
      rotationInterval: 24h
```

**Access Flow**:
```
User → tsh db connect main-db → Teleport Proxy → DB Service → RDS
                                    ↓
                           Audit Log (S3)
```

### Phase 4.3: Container Access (ECS Exec Enhancement)

**Objective**: Replace direct ECS Exec with Teleport-mediated access

**Tasks**:
1. Deploy Teleport Agent as sidecar in ECS tasks
2. Configure SSH-like access to containers
3. Set up session recording
4. Implement role-based container access

**Task Definition Modification**:
```yaml
# Service configuration with Teleport sidecar
services:
  test-app:
    # ... existing config ...

    teleport:
      enabled: true
      allowedRoles:
        - container-admin
        - container-debug
      sessionRecording: true
      # Agent runs as sidecar
      sidecar:
        cpu: 128
        memory: 256
```

### Phase 4.4: Application Access

**Objective**: Protect web applications behind Teleport

**Tasks**:
1. Configure Teleport Application Service
2. Set up JWT/OIDC token injection
3. Configure per-app access policies

**Application Configuration**:
```yaml
services:
  test-app:
    # ... existing config ...

    teleport:
      appAccess:
        enabled: true
        publicAddr: app.bax.dev
        allowedRoles:
          - app-users
          - app-admins
```

## Teleport RBAC Configuration

```yaml
# roles/db-admin.yaml
kind: role
version: v5
metadata:
  name: db-admin
spec:
  allow:
    db_labels:
      environment: ['dev', 'staging', 'prod']
    db_names: ['*']
    db_users: ['dbadmin']
  options:
    max_session_ttl: 8h

# roles/db-readonly.yaml
kind: role
version: v5
metadata:
  name: db-readonly
spec:
  allow:
    db_labels:
      environment: ['dev']
    db_names: ['*']
    db_users: ['readonly']
  options:
    max_session_ttl: 4h

# roles/container-admin.yaml
kind: role
version: v5
metadata:
  name: container-admin
spec:
  allow:
    node_labels:
      environment: ['dev', 'staging']
    logins: ['root', 'app']
  options:
    max_session_ttl: 4h
    enhanced_recording:
      - command
      - network
```

## Implementation Timeline

| Phase | Description | Dependencies | Estimated Effort |
|-------|-------------|--------------|------------------|
| 4.1 | Teleport Cluster Setup | ACM Certificate | Core setup |
| 4.2 | Database Access | Phase 4.1, RDS | DB access |
| 4.3 | Container Access | Phase 4.1 | Container access |
| 4.4 | Application Access | Phase 4.1 | App protection |

## ECSO Schema Extensions Required

### New Schema Models (schema.py)

```python
class TeleportDBAccessSpec(BaseModel):
    """Teleport database access configuration."""
    enabled: bool = False
    allowed_roles: list[str] = Field(default_factory=list, alias="allowedRoles")
    auto_rotate_credentials: bool = Field(default=True, alias="autoRotateCredentials")
    rotation_interval: str = Field(default="24h", alias="rotationInterval")

class TeleportContainerSpec(BaseModel):
    """Teleport container access configuration."""
    enabled: bool = False
    allowed_roles: list[str] = Field(default_factory=list, alias="allowedRoles")
    session_recording: bool = Field(default=True, alias="sessionRecording")
    sidecar: TeleportSidecarSpec | None = None

class TeleportAppAccessSpec(BaseModel):
    """Teleport application access configuration."""
    enabled: bool = False
    public_addr: str = Field(alias="publicAddr")
    allowed_roles: list[str] = Field(default_factory=list, alias="allowedRoles")

class TeleportClusterSpec(BaseModel):
    """Teleport cluster configuration."""
    enabled: bool = False
    domain: str
    cluster_name: str = Field(alias="clusterName")
    dynamodb_table: str = Field(alias="dynamodbTable")
    audit_bucket: str = Field(alias="auditBucket")
    sso: TeleportSSOSpec | None = None
```

### New Terraform Templates

1. `teleport_cluster.tf.j2` - Deploys Teleport Auth/Proxy
2. `teleport_db_service.tf.j2` - Database access agents
3. `teleport_agent.tf.j2` - Container access sidecar
4. `teleport_app.tf.j2` - Application access configuration

## Security Considerations

1. **Certificate Management**
   - Use ACM for TLS certificates
   - Teleport generates its own CA for internal certs

2. **Secrets**
   - Store Teleport join tokens in Secrets Manager
   - Rotate tokens regularly
   - GitHub OAuth secrets in Secrets Manager

3. **Network**
   - Teleport proxy in public subnet
   - Agents in private subnet
   - Security groups restrict agent communication

4. **Audit**
   - All sessions recorded to S3
   - CloudWatch logs for operational events
   - Enable S3 versioning and MFA delete

## Next Steps

1. Review and approve this plan
2. Set up Teleport domain DNS (teleport.bax.dev)
3. Request ACM certificate for teleport.bax.dev
4. Begin Phase 4.1 implementation
