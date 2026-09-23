# Wazuh Single-Node Deployment on Dokploy

Containerized Wazuh SIEM deployment using Docker Compose and Dokploy, designed for centralized security monitoring and cloud log ingestion.

The deployment runs the three core Wazuh components:

- **Wazuh Manager** – log analysis, decoding, rule evaluation, alert generation, agent management, and active response.
- **Wazuh Indexer** – stores and indexes security events and alerts.
- **Wazuh Dashboard** – provides the web interface for threat hunting, security monitoring, and administration.

The setup is based on **Wazuh 4.14.7** and is deployed as a single-node stack.

---

## Architecture

```text
                        ┌──────────────────────┐
                        │       Dokploy        │
                        │   Docker Compose     │
                        └──────────┬───────────┘
                                   │
                  ┌────────────────┼────────────────┐
                  │                │                │
                  ▼                ▼                ▼
          Wazuh Manager      Wazuh Indexer    Wazuh Dashboard
                  │                ▲                │
                  │                │                │
                  └──── Filebeat ──┘                │
                                   │                │
                                   └────────────────┘
```

The stack uses persistent Docker volumes so that Wazuh configuration, logs, indexed security data, and dashboard configuration survive container recreation.

---

## Repository Structure

The repository contains the reusable Wazuh configuration required by Dokploy.

Example structure:

```text
.
├── single_node/
│   ├── docker-compose.yml
│   └── config/
│       ├── wazuh_cluster/
│       ├── wazuh_dashboard/
│       └── wazuh_indexer/
│
├── deploy.yml
└── README.md
```

Runtime-sensitive files are intentionally **not committed**.

These include:

```text
files/
├── admin-key.pem
├── admin.pem
├── root-ca.pem
├── root-ca-manager.pem
├── wazuh.dashboard-key.pem
├── wazuh.dashboard.pem
├── wazuh.indexer-key.pem
├── wazuh.indexer.pem
├── wazuh.manager-key.pem
├── wazuh.manager.pem
├── wazuh-gcp-reader.json
└── other environment-specific configuration
```

> **Security:** Never commit private keys, passwords, API tokens, GCP service-account credentials, or other secrets to the repository.

---

## Prerequisites

Recommended minimum requirements for the Wazuh single-node Docker deployment:

| Resource | Minimum |
|---|---:|
| CPU | 4 cores |
| RAM | 8 GB |
| Disk | 50 GB |
| Container Runtime | Docker |
| Orchestration | Docker Compose / Dokploy |

For production environments, additional memory and storage should be provisioned according to expected log ingestion and retention.

The Linux host must also support the Wazuh Indexer's required virtual memory mappings.

Check:

```bash
sysctl vm.max_map_count
```

Expected minimum:

```text
vm.max_map_count = 262144
```

If required:

```bash
sudo sysctl -w vm.max_map_count=262144
```

Persist the setting according to the host operating system configuration.

---

## Wazuh Version

This deployment pins the Wazuh containers to:

```text
4.14.7
```

Example:

```yaml
wazuh.manager:
  image: wazuh/wazuh-manager:4.14.7

wazuh.indexer:
  image: wazuh/wazuh-indexer:4.14.7

wazuh.dashboard:
  image: wazuh/wazuh-dashboard:4.14.7
```

Pinning the version prevents unexpected changes caused by using `latest`.

---

# Deployment

## 1. Clone the Repository

```bash
git clone <repository-url>
cd dokploy-wahuz-integration
```

Verify the configuration before deployment.

---

## 2. Generate Wazuh Certificates

Wazuh uses TLS certificates for secure communication between the Manager, Indexer, Dashboard, and Filebeat.

Certificates should be generated from the appropriate Wazuh certificate configuration rather than stored permanently in Git.

Typical generated files include:

```text
admin.pem
admin-key.pem

root-ca.pem
root-ca.key

root-ca-manager.pem
root-ca-manager.key

wazuh.manager.pem
wazuh.manager-key.pem

wazuh.indexer.pem
wazuh.indexer-key.pem

wazuh.dashboard.pem
wazuh.dashboard-key.pem
```

After generation, copy the certificates to the runtime `files/` directory used by the Dokploy deployment.

The Compose configuration mounts these files into the appropriate containers.

Example:

```yaml
- ../../files/root-ca.pem:/usr/share/wazuh-indexer/config/certs/root-ca.pem
- ../../files/wazuh.indexer-key.pem:/usr/share/wazuh-indexer/config/certs/wazuh.indexer.key
- ../../files/wazuh.indexer.pem:/usr/share/wazuh-indexer/config/certs/wazuh.indexer.pem
```

### Important

Generated private keys must remain outside Git:

```gitignore
*.key
*.pem
*.json
.env
```

Review files individually before committing because some JSON files may legitimately belong in source control.

---

## 3. Configure Credentials

The deployment uses separate credentials for communication between Wazuh components.

Example:

```yaml
INDEXER_USERNAME=admin
INDEXER_PASSWORD=${INDEXER_PASSWORD}

DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=${DASHBOARD_PASSWORD}

API_USERNAME=wazuh-wui
API_PASSWORD=${API_PASSWORD}
```

Configure the passwords securely through **Dokploy environment variables**:

```text
INDEXER_PASSWORD=<secure-password>
DASHBOARD_PASSWORD=<secure-password>
API_PASSWORD=<secure-password>
```

Do not commit these values to Git.

### Credential Roles

| Credential | Purpose |
|---|---|
| `admin` | Wazuh Indexer/OpenSearch administrative user |
| `kibanaserver` | Dashboard-to-Indexer service account |
| `wazuh-wui` | Dashboard-to-Wazuh-API account |

Changing passwords is different from renaming these users. Usernames are referenced by Wazuh/OpenSearch security configuration and should not simply be renamed in `docker-compose.yml`.

---

## 4. Deploy Through Dokploy

Create a new **Docker Compose** application in Dokploy and connect this Git repository.

Configure the repository/branch and point Dokploy to the Wazuh Compose definition.

Add the required environment variables through the Dokploy UI.

The runtime directory must also contain the files referenced by the Compose bind mounts:

```text
files/
├── root-ca.pem
├── root-ca-manager.pem
├── wazuh.manager.pem
├── wazuh.manager-key.pem
├── wazuh.indexer.pem
├── wazuh.indexer-key.pem
├── wazuh.dashboard.pem
├── wazuh.dashboard-key.pem
├── admin.pem
├── admin-key.pem
├── wazuh_manager.conf
├── wazuh.indexer.yml
├── internal_users.yml
├── opensearch_dashboards.yml
└── wazuh.yml
```

Deploy the Compose application after the runtime files and environment variables are in place.

---

# Services and Ports

The deployment exposes the following Wazuh services:

| Port | Protocol | Purpose |
|---|---|---|
| `1514` | TCP | Agent communication |
| `1515` | TCP | Agent enrollment |
| `514` | UDP | Syslog |
| `55000` | TCP | Wazuh API |
| `9200` | TCP | Wazuh Indexer API |
| `5601` | TCP | Dashboard in this Dokploy deployment |

Only expose ports that are actually required by your architecture.

For production environments, administrative interfaces should not be unnecessarily exposed to the public Internet.

---

# Persistent Storage

The deployment uses named Docker volumes including:

```text
wazuh_api_configuration
wazuh_etc
wazuh_logs
wazuh_queue
wazuh_var_multigroups
wazuh_integrations
wazuh_active_response
wazuh_agentless
wazuh_wodles
filebeat_etc
filebeat_var
wazuh-indexer-data
wazuh-dashboard-config
wazuh-dashboard-custom
```

This allows containers to be restarted or recreated without losing persistent Wazuh data.

---

# Post-Deployment Validation

## Check Containers

```bash
sudo docker ps
```

The following components should be running:

```text
wazuh.manager
wazuh.indexer
wazuh.dashboard
```

Dokploy automatically prefixes container names, so the actual names may resemble:

```text
<dokploy-project>-wazuh.manager-1
<dokploy-project>-wazuh.indexer-1
<dokploy-project>-wazuh.dashboard-1
```

---

## Check Wazuh Manager

```bash
sudo docker exec <manager-container> \
  /var/ossec/bin/wazuh-control status
```

---

## Check Indexer

```bash
sudo docker exec <manager-container> \
  curl -sk \
  -u admin:<INDEXER_PASSWORD> \
  https://wazuh.indexer:9200/_cat/indices?v
```

This verifies communication between the Manager environment and the Wazuh Indexer.

---

## Check Mounted Configuration

One important deployment lesson is that Wazuh stores `/var/ossec/etc` in a persistent volume.

The source configuration may therefore exist at:

```text
/wazuh-config-mount/etc/ossec.conf
```

while the active configuration is:

```text
/var/ossec/etc/ossec.conf
```

Verify the active configuration after deployment:

```bash
sudo docker exec <manager-container> \
  grep -n -E 'logall|logall_json' \
  /var/ossec/etc/ossec.conf
```

When troubleshooting configuration changes, compare the source and destination:

```bash
sudo docker exec <manager-container> sh -c '
echo "SOURCE:"
sha256sum /wazuh-config-mount/etc/ossec.conf
echo "ACTIVE:"
sha256sum /var/ossec/etc/ossec.conf
'
```

This is especially useful when a configuration change appears correct in the mounted source file but is not reflected inside Wazuh.

---

# Archive Logging

For complete event visibility, this deployment can enable Wazuh archives:

```xml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>yes</logall>
  <logall_json>yes</logall_json>
</global>
```

The important distinction is:

```text
Incoming Event
      │
      ▼
Wazuh Manager
      │
      ├── No alert ─────► Archives
      │
      └── Rule matched ─► Alerts + Archives
```

Therefore:

- `wazuh-alerts-*` contains events that generated alerts.
- `wazuh-archives-*` provides visibility into all archived events when archive indexing is configured.

This is useful when developing or troubleshooting custom security detections.

---

# Security Detection

Wazuh should not be treated only as a log viewer.

Its Manager processes incoming events through:

```text
Log ingestion
      ↓
Decoding
      ↓
Rule evaluation
      ↓
Threat detection
      ↓
Alert generation
      ↓
Optional notification / response
```

Wazuh already includes built-in detection rules for many platforms and event types.

Before creating a custom rule, verify whether Wazuh already provides a suitable built-in rule.

Example GCP detections observed during implementation include:

```text
65064  GCP service account deleted
65070  GCP new service account created
65071  GCP service account key created
65072  GCP service account key deleted
```

Custom rules should therefore be reserved for organization-specific security requirements, severity changes, correlations, allowlists, or detections not covered by the default ruleset.

---

# Testing Rules

Wazuh provides `wazuh-logtest` for validating decoding and rule matching:

```bash
sudo docker exec -it <manager-container> \
  /var/ossec/bin/wazuh-logtest
```

Processing occurs in phases:

```text
Phase 1 → Pre-decoding
Phase 2 → Decoding
Phase 3 → Rule filtering
```

Always test a custom rule before relying on it for production alerting.

---

# Security Considerations

For production deployments:

- Do not use default passwords.
- Do not commit certificates or private keys.
- Do not commit cloud service-account credentials.
- Restrict Wazuh Dashboard access.
- Restrict Indexer/API ports where external access is unnecessary.
- Use TLS between Wazuh components.
- Maintain backups of persistent volumes and important configuration.
- Pin Wazuh Docker versions.
- Review Wazuh upgrades before deploying them.
- Use least-privilege cloud credentials for integrations.
- Review built-in rules before adding custom detections.
- Monitor disk utilization because indexed security data grows over time.

---

# Troubleshooting

### Configuration changed but Wazuh still uses the old configuration

Compare:

```text
/wazuh-config-mount/etc/ossec.conf
```

with:

```text
/var/ossec/etc/ossec.conf
```

and restart the Manager when appropriate.

### Dashboard starts before Indexer

The Dashboard may temporarily report that the Indexer is unavailable while the Indexer initializes. Allow the Indexer to become healthy before treating this as a deployment failure.

### Events exist but are not visible in Threat Hunting

Check whether they generated an alert.

Raw events may exist only in:

```text
wazuh-archives-*
```

while detected events appear in:

```text
wazuh-alerts-*
```

### Check Manager logs

```bash
sudo docker logs <manager-container> --tail 100
```

### Check Indexer indices

```bash
sudo docker exec <manager-container> \
  curl -sk \
  -u admin:<INDEXER_PASSWORD> \
  https://wazuh.indexer:9200/_cat/indices?v
```

---

# Future Integrations

This deployment provides the base SIEM platform for additional security integrations such as:

```text
GCP Cloud Audit Logs
Cloud Run
API Gateway
IAM
Cloudflare
Appwrite
Redis Cloud
Algolia
On-premise workloads
Custom security rules
Email notifications
Active Response / SOAR
```

These integrations should be implemented independently of the core Wazuh deployment so the base SIEM stack remains reusable across projects.

---

## References

- Wazuh Docker deployment documentation
- Wazuh Docker utilities documentation
- Wazuh ruleset documentation
- Dokploy Docker Compose documentation

---

## Disclaimer

This repository provides a reusable deployment baseline. Production security requirements vary by infrastructure, compliance requirements, threat model, log volume, and availability requirements. Review network exposure, credentials, backup strategy, retention, and high-availability requirements before production use.
