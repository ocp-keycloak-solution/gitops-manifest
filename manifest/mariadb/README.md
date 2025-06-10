# MariaDB Deployment for Keycloak

This directory contains the MariaDB deployment configuration with Master-Slave replication for Red Hat Build of Keycloak.

## Architecture

- **Master**: Handles all write operations and serves as the primary database
- **Slave**: Replicates data from master and can handle read operations
- **Two separate PVCs**: Each instance has its own persistent storage

## Components

- `mariadb-master-deployment.yaml` - Master MariaDB instance
- `mariadb-slave-deployment.yaml` - Slave MariaDB instance (with replication)
- `mariadb-master-pvc.yaml` - Persistent storage for master (20Gi)
- `mariadb-slave-pvc.yaml` - Persistent storage for slave (20Gi)
- `mariadb-secret.yaml` - Database credentials (needs secure passwords)
- `mariadb-configmap.yaml` - Database configuration and initialization scripts
- `mariadb-*-service.yaml` - Service definitions

## Services

- `mariadb-master` - Direct access to master instance
- `mariadb-slave` - Direct access to slave instance  
- `mariadb` - Load balanced access (use with caution for writes)

## Database Configuration

- **Database**: `keycloak`
- **Username**: `keycloak`
- **Character Set**: `utf8mb4` with `utf8mb4_unicode_ci` collation
- **Replication**: Binary log based replication

## Security Requirements

⚠️ **IMPORTANT**: Before deploying, update the following passwords in `mariadb-secret.yaml`:

1. `mariadb-root-password` - Root user password
2. `mariadb-replication-password` - Replication user password  
3. `mariadb-keycloak-password` - Keycloak application user password

Generate secure passwords:
```bash
# Generate base64 encoded passwords
echo -n "your-secure-password" | base64
```

Also update the plaintext passwords in:
- `mariadb-configmap.yaml` (in init-master.sql)

## Deployment

```bash
# Deploy via ArgoCD
oc apply -f application.yaml

# Or deploy directly
oc apply -k .
```

## Monitoring Replication

```bash
# Check master status
oc exec -it deployment/mariadb-master -n keycloak -- mysql -u root -p -e "SHOW MASTER STATUS;"

# Check slave status
oc exec -it deployment/mariadb-slave -n keycloak -- mysql -u root -p -e "SHOW SLAVE STATUS\G"
```

## Connecting from Keycloak

Use the master service for Keycloak connections:
- **Host**: `mariadb-master.keycloak.svc.cluster.local`
- **Port**: `3306`
- **Database**: `keycloak`
- **Username**: `keycloak`
- **Password**: (from secret)

## Performance Tuning

The configuration includes optimizations for Keycloak:
- InnoDB buffer pool: 512MB
- Connection limits: 200 concurrent connections
- Timeouts: 10 minutes
- Binary logging for replication

## Backup Strategy

Consider implementing:
- Regular mysqldump backups
- PVC snapshots
- Cross-region replication for DR

## Troubleshooting

```bash
# Check pods
oc get pods -n keycloak -l app=mariadb

# Check logs
oc logs deployment/mariadb-master -n keycloak
oc logs deployment/mariadb-slave -n keycloak

# Check replication lag
oc exec -it deployment/mariadb-slave -n keycloak -- mysql -u root -p -e "SHOW SLAVE STATUS\G" | grep Seconds_Behind_Master
```
