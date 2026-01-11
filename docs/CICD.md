# CI/CD Documentation - IQFeed Docker

## Overview
Automated CI/CD pipeline for building and deploying the IQFeed Docker container to Google Artifact Registry.

## Repository
- **GitHub**: https://github.com/Zaradacht/iqfeed-docker
- **Workflow File**: `.github/workflows/build-docker.yml`

## Pipeline Architecture

### Trigger Events
1. **Push to master/main branch**: Automatic build on code changes
2. **Manual dispatch**: On-demand build via GitHub Actions UI

### Build Process

```
┌─────────────────┐
│   Push Code     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Checkout Code  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Setup Buildx    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Authenticate   │
│   to GCP        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Build Image    │
│  (linux/amd64)  │
└────────┬────────┘
         │
         ├─────────────────────┐
         │                     │
         ▼                     ▼
┌─────────────────┐   ┌──────────────────┐
│  Push to GAR    │   │  Export TAR      │
│  - latest       │   │  - Artifact      │
│  - SHA tag      │   │  - 7 day retain  │
└─────────────────┘   └──────────────────┘
```

## Configuration

### Required GitHub Secrets

| Secret Name | Description | Example Value |
|------------|-------------|---------------|
| `GCP_SA_KEY` | Service account JSON key | `{"type": "service_account", ...}` |
| `GCP_PROJECT_ID` | GCP project ID | `wa-equity-trading` |
| `GAR_LOCATION` | Artifact Registry region | `us-central1` |
| `GAR_REPOSITORY` | Repository name | `docker-images` |

### Setup Instructions

1. **Create GCP Service Account**:
```bash
gcloud iam service-accounts create github-actions \
  --project wa-equity-trading

gcloud projects add-iam-policy-binding wa-equity-trading \
  --member="serviceAccount:github-actions@wa-equity-trading.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud iam service-accounts keys create key.json \
  --iam-account=github-actions@wa-equity-trading.iam.gserviceaccount.com
```

2. **Add GitHub Secrets**:
   - Go to: https://github.com/Zaradacht/iqfeed-docker/settings/secrets/actions
   - Click "New repository secret"
   - Add each secret from the table above

3. **Create Artifact Registry Repository**:
```bash
gcloud artifacts repositories create docker-images \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images for IQFeed"
```

## Build Outputs

### 1. Google Artifact Registry
**Image URI**: `us-central1-docker.pkg.dev/wa-equity-trading/docker-images/iqfeed-docker`

**Tags**:
- `:latest` - Always points to the most recent build
- `:SHA` - Specific commit hash (e.g., `:a8f858b7e30d0b69f5c08c50256e7dd62db441f2919a0476592def90048285f9`)

**Pull Command**:
```bash
docker pull us-central1-docker.pkg.dev/wa-equity-trading/docker-images/iqfeed-docker:latest
```

### 2. GitHub Artifacts
**Artifact Name**: `iqfeed-docker`

**Location**: GitHub Actions → Workflow run → Artifacts section

**Download**: Available for 7 days after build

**Usage**:
```bash
# Download from GitHub UI, then:
docker load < iqfeed-docker.tar
```

## Running the Built Image

### From Artifact Registry

```bash
# Authenticate Docker
gcloud auth configure-docker us-central1-docker.pkg.dev

# Pull image
docker pull us-central1-docker.pkg.dev/wa-equity-trading/docker-images/iqfeed-docker:latest

# Run container
docker run -d \
  --platform linux/amd64 \
  --name iqfeed \
  -p 5009:5009 -p 9100:9100 -p 9200:9200 -p 9300:9300 -p 9400:9400 \
  -e IQFEED_PRODUCT_ID="YOUR_PRODUCT_ID" \
  -e IQFEED_LOGIN="YOUR_LOGIN" \
  -e IQFEED_PASSWORD="YOUR_PASSWORD" \
  us-central1-docker.pkg.dev/wa-equity-trading/docker-images/iqfeed-docker:latest
```

### From Downloaded Artifact

```bash
# Load image
docker load < iqfeed-docker.tar

# Run container
docker run -d \
  --platform linux/amd64 \
  --name iqfeed \
  -p 5009:5009 -p 9100:9100 -p 9200:9200 -p 9300:9300 -p 9400:9400 \
  -e IQFEED_PRODUCT_ID="YOUR_PRODUCT_ID" \
  -e IQFEED_LOGIN="YOUR_LOGIN" \
  -e IQFEED_PASSWORD="YOUR_PASSWORD" \
  iqfeed-docker:latest
```

## Monitoring & Troubleshooting

### View Build Status
- **GitHub Actions**: https://github.com/Zaradacht/iqfeed-docker/actions
- **Latest Run**: Click on the most recent workflow run to see logs

### Common Issues

#### 1. Authentication Failed
**Error**: `Error: google-github-actions/auth failed`

**Solution**: 
- Verify `GCP_SA_KEY` secret contains valid JSON
- Check service account has `artifactregistry.writer` role

#### 2. Push Declined
**Error**: `push declined due to email privacy restrictions`

**Solution**: Already handled by workflow (uses GitHub noreply email)

#### 3. Build Timeout
**Error**: Build exceeds 6 hours

**Solution**: 
- This is expected on ARM64 runners due to emulation
- Consider using self-hosted x86_64 runners for faster builds

### Build Times
- **GitHub Actions (x86_64 runner)**: ~15-20 minutes
- **ARM64 system with emulation**: ~60-90 minutes (if built locally)

## Maintenance

### Updating the Workflow

1. Edit `.github/workflows/build-docker.yml`
2. Commit and push to master/main
3. Workflow automatically uses the updated version

### Updating Dockerfile

1. Modify `Dockerfile`
2. Commit and push to master/main
3. CI/CD automatically builds and pushes new image

### Image Cleanup

List old images:
```bash
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/wa-equity-trading/docker-images/iqfeed-docker
```

Delete specific image:
```bash
gcloud artifacts docker images delete \
  us-central1-docker.pkg.dev/wa-equity-trading/docker-images/iqfeed-docker:OLD_SHA
```

## Security Best Practices

1. **Rotate Service Account Keys**: Every 90 days
2. **Use Specific Tags**: In production, pin to SHA tags instead of `:latest`
3. **Review IAM Permissions**: Quarterly audit of service account roles
4. **Scan Images**: Consider adding vulnerability scanning step to workflow
5. **Secrets Rotation**: Update GitHub Secrets when rotating GCP keys

## Future Enhancements

- [ ] Add image vulnerability scanning (Trivy, Snyk)
- [ ] Add multi-architecture support (linux/amd64, linux/arm64)
- [ ] Add automated testing step before push
- [ ] Add Slack/email notifications on build success/failure
- [ ] Add deployment to Cloud Run or GKE
- [ ] Add image signing for supply chain security

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Google Artifact Registry](https://cloud.google.com/artifact-registry/docs)
- [Docker Buildx](https://docs.docker.com/buildx/working-with-buildx/)
