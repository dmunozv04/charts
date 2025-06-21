# Apache Guacamole Helm Chart

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 1.5.5](https://img.shields.io/badge/AppVersion-1.5.5-informational?style=flat-square)

A Helm chart for deploying Apache Guacamole, a clientless remote desktop gateway that supports standard protocols like VNC, RDP and SSH.

## Overview

Apache Guacamole is a clientless remote desktop gateway. It supports standard protocols like VNC, RDP, and SSH. This Helm chart deploys Guacamole with separate deployments for:

- **Guacamole Client**: The web application providing the user interface
- **Guacd**: The proxy daemon that handles the actual protocol connections

## Features

- 🔐 **Multiple Authentication Backends**: PostgreSQL, MySQL, LDAP, Header-based, SAML, TOTP, OpenID Connect
- 🎯 **Separate Deployments**: Independent scaling and configuration for client and guacd
- 📈 **Auto-scaling**: Horizontal Pod Autoscaler support for both components
- 🔧 **Highly Configurable**: Support for all major Guacamole environment variables
- 🛡️ **Security**: Service accounts, security contexts, and proxy support
- 📊 **Monitoring**: Health checks and resource management
- 🌐 **Ingress**: Built-in ingress support with TLS
- 📝 **Documentation**: Comprehensive values documentation

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- PV provisioner support in the underlying infrastructure (optional, for session recordings)

## Installation

### Add Helm Repository

```bash
helm repo add guacamole https://your-repo-url
helm repo update
```

### Install Chart

```bash
# Install with default values
helm install my-guacamole guacamole/guacamole

# Install with custom values
helm install my-guacamole guacamole/guacamole -f values.yaml

# Install in specific namespace
helm install my-guacamole guacamole/guacamole --namespace guacamole --create-namespace
```

## Database Authentication Setup

### PostgreSQL Authentication

1. **Initialize Database Schema**:

   ```bash
   # Generate the schema initialization script
   docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql
   
   # Apply to your PostgreSQL database
   psql -h your-postgres-host -U postgres -d guacamole -f initdb.sql
   ```

2. **Configure Values**:

   ```yaml
   client:
     auth:
       postgresql:
         enabled: true
         hostname: "your-postgres-host"
         database: "guacamole"
         user: "guacamole_user"
         password: "your-secure-password"
   ```

### MySQL Authentication

1. **Initialize Database Schema**:

   ```bash
   # Generate the schema initialization script
   docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
   
   # Apply to your MySQL database
   mysql -h your-mysql-host -u root -p guacamole < initdb.sql
   ```

2. **Configure Values**:

   ```yaml
   client:
     auth:
       mysql:
         enabled: true
         hostname: "your-mysql-host"
         database: "guacamole"
         user: "guacamole_user"
         password: "your-secure-password"
   ```

### Default Database Credentials

⚠️ **IMPORTANT**: When using a fresh database, the default credentials are:

- **Username**: `guacadmin`
- **Password**: `guacadmin`

**Change these credentials immediately after first login!**

## Authentication Backends

### LDAP Authentication

```yaml
client:
  auth:
    ldap:
      enabled: true
      hostname: "your-ldap-server"
      port: "389"
      userBaseDn: "ou=users,dc=example,dc=com"
      searchBindDn: "cn=admin,dc=example,dc=com"
      searchBindPassword: "admin-password"
```

### SAML Authentication

```yaml
client:
  auth:
    saml:
      enabled: true
      idpUrl: "https://your-idp.com/saml/sso"
      entityId: "guacamole"
      callbackUrl: "https://your-guacamole.com/guacamole/"
```

### OpenID Connect Authentication

```yaml
client:
  auth:
    openid:
      enabled: true
      authorizationEndpoint: "https://your-provider.com/auth"
      jwksEndpoint: "https://your-provider.com/jwks"
      issuer: "https://your-provider.com"
      clientId: "your-client-id"
      redirectUri: "https://your-guacamole.com/guacamole/"
```

## Configuration Examples

### Basic Deployment with PostgreSQL

```yaml
client:
  replicaCount: 2
  auth:
    postgresql:
      enabled: true
      hostname: "postgres.default.svc.cluster.local"
      database: "guacamole"
      user: "guacamole"
      password: "secure-password"
  
  resources:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi

guacd:
  replicaCount: 1
  resources:
    limits:
      cpu: 300m
      memory: 256Mi
    requests:
      cpu: 150m
      memory: 128Mi

ingress:
  enabled: true
  hosts:
    - host: guacamole.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: guacamole-tls
      hosts:
        - guacamole.example.com
```

### Production Deployment with Auto-scaling

```yaml
client:
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
  
  auth:
    postgresql:
      enabled: true
      hostname: "postgres.database.svc.cluster.local"
      database: "guacamole"
      user: "guacamole"
      password: "production-password"
  
  api:
    sessionTimeout: "120"  # 2 hours
    maxRequestSize: "10485760"  # 10MB
  
  proxy:
    enabled: true
    allowedIpsRegex: "192\\.168\\..*|10\\..*"

guacd:
  autoscaling:
    enabled: true
    minReplicas: 1
    maxReplicas: 5
    targetCPUUtilizationPercentage: 80
  
  logLevel: "info"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
  hosts:
    - host: guacamole.company.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: guacamole-tls
      hosts:
        - guacamole.company.com
```

## Advanced Configuration

### Session Recording

```yaml
client:
  recording:
    enabled: true
    searchPath: "/opt/guacamole/recordings"

# Mount shared volume for recordings
volumeMounts:
  - name: recordings
    mountPath: /opt/guacamole/recordings

volumes:
  - name: recordings
    persistentVolumeClaim:
      claimName: guacamole-recordings
```

### Custom Extensions

```yaml
client:
  guacamoleHome:
    path: "/etc/guacamole"
  
  extensionPriority:
    order: "saml, postgresql, ldap"
  
  extraEnv:
    - name: CUSTOM_EXTENSION_VAR
      value: "custom-value"
```

### Multi-Authentication Setup

```yaml
client:
  auth:
    postgresql:
      enabled: true
      hostname: "postgres.default.svc.cluster.local"
      database: "guacamole"
      user: "guacamole"
      password: "db-password"
    
    ldap:
      enabled: true
      hostname: "ldap.company.com"
      port: "389"
      userBaseDn: "ou=users,dc=company,dc=com"
      searchBindDn: "cn=guacamole,ou=services,dc=company,dc=com"
      searchBindPassword: "ldap-password"
    
    saml:
      enabled: true
      idpUrl: "https://sso.company.com/saml/sso"
      entityId: "guacamole-prod"
  
  extensionPriority:
    order: "saml, ldap, postgresql"
  
  authProviders:
    skipIfUnavailable: "ldap"  # Continue if LDAP is down
```

## Upgrading

### From 0.x to 0.1.0

This version introduces separate deployments for client and guacd. Update your values.yaml:

```yaml
# Old format
image:
  repository: guacamole/guacamole

# New format
client:
  image:
    repository: guacamole/guacamole
guacd:
  image:
    repository: guacamole/guacd
```

## Troubleshooting

### Common Issues

1. **Database Connection Failed**:
   - Verify database credentials and hostname
   - Ensure database schema is initialized
   - Check network connectivity between pods and database

2. **Authentication Not Working**:
   - Verify extension priority order
   - Check authentication provider configuration
   - Review Guacamole logs for specific errors

3. **Connection to Remote Desktop Failed**:
   - Verify guacd deployment is running
   - Check connectivity between client and guacd pods
   - Ensure target systems are accessible from guacd pods

### Debugging

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=guacamole

# View client logs
kubectl logs -l app.kubernetes.io/component=client

# View guacd logs
kubectl logs -l app.kubernetes.io/component=guacd

# Test database connectivity
kubectl exec -it deployment/my-guacamole-client -- nc -zv postgres.default.svc.cluster.local 5432
```

## Security Considerations

- Always change default database credentials
- Use TLS for all external connections
- Configure proper network policies
- Regularly update to latest versions
- Use strong passwords and enable MFA when available
- Monitor access logs and failed login attempts

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| client.replicaCount | int | `1` | Number of client replicas |
| client.image.repository | string | `"guacamole/guacamole"` | Client container image repository |
| client.image.pullPolicy | string | `"IfNotPresent"` | Client image pull policy |
| client.service.type | string | `"ClusterIP"` | Client service type |
| client.service.port | int | `8080` | Client service port |
| client.auth.postgresql.enabled | bool | `false` | Enable PostgreSQL authentication |
| client.auth.mysql.enabled | bool | `false` | Enable MySQL authentication |
| client.auth.ldap.enabled | bool | `false` | Enable LDAP authentication |
| guacd.replicaCount | int | `1` | Number of guacd replicas |
| guacd.image.repository | string | `"guacamole/guacd"` | Guacd container image repository |
| guacd.service.port | int | `4822` | Guacd service port |
| ingress.enabled | bool | `false` | Enable ingress |

For a complete list of values, see `values.yaml`.

## Links

- [Apache Guacamole Official Documentation](https://guacamole.apache.org/doc/gug/)
- [Guacamole Docker Images](https://hub.docker.com/u/guacamole)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
