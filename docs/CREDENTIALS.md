# Credentials Storage Strategy

## Overview
This document outlines the strategy for securely storing and managing credentials across the IQFeed integration project.

## Credential Types

### 1. IQFeed Credentials
- **Product ID**: Developer product identifier for IQFeed API access
- **Login**: IQFeed account username
- **Password**: IQFeed account password

### 2. Google Cloud Platform (GCP) Credentials
- **Service Account Key**: JSON key file for GCP API access
- **Project ID**: GCP project identifier
- **Artifact Registry Location**: Region for container storage
- **Repository Name**: Container repository identifier

### 3. GitHub Secrets
- **Personal Access Tokens**: For repository access (if needed)

## Storage Solutions

### GitHub Secrets (For CI/CD)

**Location**: Repository Settings → Secrets and variables → Actions

**Secrets Configured**:
- `GCP_SA_KEY`: Full JSON content of GCP service account key
- `GCP_PROJECT_ID`: GCP project ID (e.g., `wa-equity-trading`)
- `GAR_LOCATION`: Artifact Registry region (e.g., `us-central1`)
- `GAR_REPOSITORY`: Repository name (e.g., `docker-images`)

**Access**: Only available to GitHub Actions workflows in the repository

**Rotation**: Update manually when service account keys are rotated

### GCP Secret Manager (For Runtime)

**Use Cases**:
- IQFeed credentials for production containers
- Database connection strings
- API keys for external services

**Setup**:
```bash
# Create secrets
gcloud secrets create iqfeed-product-id --data-file=- <<< "YOUR_PRODUCT_ID"
gcloud secrets create iqfeed-login --data-file=- <<< "YOUR_LOGIN"
gcloud secrets create iqfeed-password --data-file=- <<< "YOUR_PASSWORD"

# Grant access to service account
gcloud secrets add-iam-policy-binding iqfeed-product-id \
  --member="serviceAccount:YOUR_SA@PROJECT.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

**Access from Code**:
```python
from google.cloud import secretmanager

client = secretmanager.SecretManagerServiceClient()
name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
response = client.access_secret_version(request={"name": name})
secret_value = response.payload.data.decode("UTF-8")
```

### Local Development (.env files)

**File**: `.env` (gitignored)

**Example**:
```bash
IQFEED_PRODUCT_ID=YOUR_PRODUCT_ID
IQFEED_LOGIN=YOUR_LOGIN
IQFEED_PASSWORD=YOUR_PASSWORD
GCP_PROJECT_ID=wa-equity-trading
```

**Security**:
- Never commit `.env` files to git
- Use `.env.example` templates with placeholder values
- Store actual values in password manager (1Password, LastPass, etc.)

## Service Account Management

### GCP Service Account for CI/CD

**Name**: `github-actions@wa-equity-trading.iam.gserviceaccount.com`

**Roles**:
- `roles/artifactregistry.writer`: Push Docker images to Artifact Registry

**Key Rotation**:
1. Create new key: `gcloud iam service-accounts keys create new-key.json --iam-account=github-actions@PROJECT.iam.gserviceaccount.com`
2. Update GitHub Secret `GCP_SA_KEY` with new key content
3. Test CI/CD pipeline
4. Delete old key: `gcloud iam service-accounts keys delete OLD_KEY_ID --iam-account=github-actions@PROJECT.iam.gserviceaccount.com`

### GCP Service Account for Runtime

**Name**: `iqfeed-service@wa-equity-trading.iam.gserviceaccount.com`

**Roles**:
- `roles/secretmanager.secretAccessor`: Read secrets
- `roles/storage.objectCreator`: Write to GCS buckets
- `roles/bigquery.dataEditor`: Load data into BigQuery

## Security Best Practices

### Do's ✓
- Use separate service accounts for CI/CD vs runtime
- Rotate keys every 90 days
- Use GCP Secret Manager for runtime secrets
- Use GitHub Secrets for CI/CD credentials
- Enable audit logging for secret access
- Use least-privilege IAM roles

### Don'ts ✗
- Never commit credentials to git
- Never log credentials in application logs
- Never share service account keys via email/Slack
- Never use the same credentials across environments
- Never grant overly broad IAM permissions

## Environment-Specific Configuration

### Development
- Use `.env` files locally
- Use test/sandbox IQFeed account
- Use separate GCP project or dataset

### Production
- Use GCP Secret Manager exclusively
- Use production IQFeed credentials
- Use production GCP project and datasets
- Enable audit logging and monitoring

## Credential Rotation Schedule

| Credential Type | Rotation Frequency | Owner |
|----------------|-------------------|-------|
| GCP Service Account Keys | 90 days | DevOps Team |
| IQFeed Password | 180 days | Data Team |
| GitHub Personal Access Tokens | 365 days | Developer |

## Emergency Procedures

### Compromised Credentials
1. **Immediately**: Disable/delete the compromised key or secret
2. **Generate**: Create new credentials
3. **Update**: Update all systems using the credentials
4. **Verify**: Test all affected services
5. **Audit**: Review access logs to assess impact
6. **Document**: Log the incident and response

## Access Control

### Who Has Access
- **GitHub Secrets**: Repository admins and Actions workflows
- **GCP Secret Manager**: Service accounts with secretAccessor role
- **Service Account Keys**: Stored in 1Password, accessible to DevOps team

### Access Audit
- Review GitHub repository access quarterly
- Review GCP IAM policies monthly
- Review Secret Manager access logs weekly

## References

- [GitHub Secrets Documentation](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [GCP Secret Manager Documentation](https://cloud.google.com/secret-manager/docs)
- [GCP IAM Best Practices](https://cloud.google.com/iam/docs/best-practices-service-accounts)
