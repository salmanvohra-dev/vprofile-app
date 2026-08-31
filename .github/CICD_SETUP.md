# CI/CD Pipeline Setup Guide

This document provides instructions for setting up and configuring the GitHub Actions CI/CD pipeline for the vprofile application.

## Pipeline Overview

The CI/CD pipeline implements the following workflow:

### Trigger Conditions
- **Feature branch push** → No pipeline runs
- **Pull Request to main** → `build-and-sonar` job runs (Maven build, tests, Checkstyle, SonarQube scan, quality gate)
- **Merge to main** → `docker-build-push` and `update-helm` jobs run (Docker build, ECR push, Helm update)

### Jobs

#### 1. `build-and-sonar` (PR Only)
**Trigger:** `pull_request` event to `main` branch

**Steps:**
- Checkout code with full git history (required for SonarQube)
- Set up JDK 21
- Cache SonarQube packages for faster builds
- Cache Maven dependencies based on `pom.xml` hash
- Run Maven clean verify (includes unit tests)
- Run Checkstyle code quality checks
- Execute SonarQube scan using `sonarsource/sonarqube-scan-action@v2`
- Check SonarQube quality gate using `sonarsource/sonarqube-quality-gate-action@v1.1.0`

**Quality Gate Enforcement:** If quality gate fails, the workflow will fail and prevent PR merge (ensure branch protection rules are enabled).

**Outputs:** None

#### 2. `docker-build-push` (Main Push Only)
**Trigger:** `push` event to `main` branch

**Steps:**
- Checkout code
- Set up JDK 21
- Cache Maven dependencies
- Build WAR package with Maven (skip tests since they ran in PR)
- Configure AWS credentials
- Login to Amazon ECR
- Build Docker image from `Docker-files/app/multistage/Dockerfile`
- Tag image with commit SHA and `latest`
- Push both tags to ECR repository
- Output image tag and registry for downstream jobs

**Outputs:**
- `image_tag`: Commit SHA (e.g., `abc123def456`)
- `ecr_registry`: ECR registry URI (e.g., `123456789.dkr.ecr.us-east-1.amazonaws.com`)
- `full_image_uri`: Complete image URI with tag

#### 3. `update-helm` (Main Push Only, Depends on docker-build-push)
**Trigger:** `push` event to `main` branch

**Dependencies:** Requires successful completion of `docker-build-push` job

**Steps:**
- Checkout the Helm repository using `GITOPS_PAT`
- Install `yq` CLI tool
- Update `helm/vprofile/values.yaml` fields:
  - `app.image` → ECR repository URI
  - `app.tag` → Commit SHA
- Create a Pull Request in the Helm repo with update details
- PR is auto-deleted after merge

---

## Required GitHub Configuration

### 1. Secrets

Configure these secrets in your GitHub repository settings (`Settings → Secrets and variables → Actions`):

| Secret Name | Description | Example |
|---|---|---|
| `SONAR_TOKEN` | SonarQube authentication token | Generated from SonarQube UI |
| `AWS_ACCESS_KEY_ID` | AWS IAM access key for ECR | AKIA... |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret access key | (sensitive) |
| `HELM_REPO_USER` | GitHub username/org for Helm repo | `salmanvohra-dev` |
| `GITOPS_PAT` | GitHub Personal Access Token for Helm repo | (sensitive) |
| `SLACK_WEBHOOK` | (Optional) Slack webhook for notifications | https://hooks.slack.com/... |

**How to create secrets:**
1. Go to Repository → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Enter secret name and value
4. Click "Add secret"

### 2. Variables

Configure these variables in your GitHub repository settings (`Settings → Secrets and variables → Variables`):

| Variable Name | Description | Example |
|---|---|---|
| `AWS_REGION` | AWS region for ECR | `us-east-1` |
| `ECR_REPOSITORY` | ECR repository name | `vprofileappimg` |
| `HELM_REPO_NAME` | Name of the Helm repository | `vprofile-helm` |
| `SONAR_HOST_URL` | Self-hosted SonarQube URL | `http://sonar.example.com:9000` |

**How to create variables:**
1. Go to Repository → Settings → Secrets and variables → Variables
2. Click "New repository variable"
3. Enter variable name and value
4. Click "Add variable"

### 3. Branch Protection Rules

To enforce quality gate checks and prevent direct merges:

1. Go to Repository → Settings → Branches
2. Click "Add rule" under "Branch protection rules"
3. Configure for `main` branch:
   - ✅ Require pull request reviews before merging
   - ✅ Require status checks to pass before merging
   - ✅ Require branches to be up to date before merging
   - ✅ Require code quality checks (SonarQube)
   - ✅ Restrict who can push to matching branches

---

## Prerequisites

### Local Project Requirements

1. **pom.xml** with standard Maven configuration
2. **Dockerfile** at `Docker-files/app/multistage/Dockerfile`
3. **sonar-project.properties** in project root for SonarQube configuration:
   ```properties
   sonar.projectKey=vprofile-app
   sonar.projectName=VProfile Application
   sonar.sources=src
   sonar.java.binaries=target/classes
   sonar.java.source=21
   ```

### AWS Configuration

1. **IAM User** with ECR permissions:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage",
           "ecr:PutImage",
           "ecr:InitiateLayerUpload",
           "ecr:UploadLayerPart",
           "ecr:CompleteLayerUpload",
           "ecr:GetAuthorizationToken",
           "ecr:DescribeRepositories"
         ],
         "Resource": "arn:aws:ecr:us-east-1:ACCOUNT_ID:repository/vprofileappimg"
       }
     ]
   }
   ```

2. **ECR Repository** must exist:
   ```bash
   aws ecr create-repository \
     --repository-name vprofileappimg \
     --region us-east-1
   ```

### SonarQube Configuration

1. **Self-hosted SonarQube** running and accessible
2. **Authentication Token** generated:
   - Login to SonarQube → My Account → Security → Generate Token
   - Use this token as `SONAR_TOKEN` secret
3. **Project created** in SonarQube with key `vprofile-app`

### Helm Repository Configuration

1. **Separate GitHub repository** for Helm charts (e.g., `vprofile-helm`)
2. **Branch protection** on `main` to allow PRs from CI/CD
3. **Structure:**
   ```
   vprofile-helm/
   ├── helm/
   │   └── vprofile/
   │       ├── Chart.yaml
   │       ├── values.yaml
   │       └── templates/
   ```

---

## Environment Variables

All environment variables are defined at the workflow level and available to all jobs:

```yaml
env:
  AWS_REGION: ${{ vars.AWS_REGION }}                    # From Variables
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}            # From Variables
  HELM_REPO_NAME: ${{ vars.HELM_REPO_NAME }}            # From Variables
  SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}            # From Variables
```

---

## Caching Strategy

### Maven Cache
- **Key:** `${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}`
- **Path:** `~/.m2`
- **Purpose:** Cache Maven dependencies based on `pom.xml` hash
- **Benefit:** 60-70% faster PR builds on average

### SonarQube Cache
- **Key:** `${{ runner.os }}-sonar`
- **Path:** `~/.sonar/cache`
- **Purpose:** Cache SonarQube scanner and analysis data
- **Benefit:** Faster SonarQube scans on repeated runs

---

## Output Passing Between Jobs

The `docker-build-push` job outputs values used by the `update-helm` job:

```yaml
outputs:
  image_tag: ${{ steps.image.outputs.tag }}                    # Commit SHA
  ecr_registry: ${{ steps.login-ecr.outputs.registry }}        # ECR registry URL
  full_image_uri: ${{ steps.image.outputs.full_uri }}          # Complete image URI
```

Access in dependent jobs using `needs.JOBNAME.outputs.OUTPUTNAME`:

```yaml
env:
  IMAGE: ${{ needs.docker-build-push.outputs.ecr_registry }}/${{ env.ECR_REPOSITORY }}
  IMAGE_TAG: ${{ needs.docker-build-push.outputs.image_tag }}
```

---

## Troubleshooting

### Quality Gate Failures
- Check SonarQube quality gate configuration in your SonarQube server
- Review code quality issues in SonarQube dashboard
- Fix violations or adjust quality gate thresholds

### ECR Push Failures
- Verify AWS credentials are correct and have ECR permissions
- Check ECR repository exists in `us-east-1`
- Ensure Docker image builds successfully locally first

### Helm Values Update Issues
- Verify `GITOPS_PAT` has write access to Helm repo
- Check `yq` command syntax and field paths
- Ensure `helm/vprofile/values.yaml` has expected structure

### SonarQube Connection Issues
- Verify `SONAR_HOST_URL` is accessible from GitHub Actions
- Check `SONAR_TOKEN` is valid and hasn't expired
- Ensure firewall rules allow GitHub Actions runner IPs

---

## Monitoring and Notifications

### Viewing Workflow Runs
1. Go to Repository → Actions
2. Click workflow name to view history
3. Click specific run to view job details and logs

### Slack Notifications (Optional)
To add Slack notifications, add this step to any job:

```yaml
- name: Notify Slack
  if: always()
  uses: slackapi/slack-github-action@v1.24.0
  with:
    webhook-url: ${{ secrets.SLACK_WEBHOOK }}
    payload: |
      {
        "text": "Pipeline Status: ${{ job.status }}",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*${{ github.workflow }}*\nJob: ${{ job.status }}\nRef: ${{ github.ref }}"
            }
          }
        ]
      }
```

---

## Best Practices

1. **Keep secrets secure** - Never commit secrets, use GitHub Secrets
2. **Regularly rotate credentials** - Update AWS keys, GitHub tokens quarterly
3. **Monitor build times** - Watch for cache misses or performance degradation
4. **Review SonarQube reports** - Address code quality issues proactively
5. **Tag Docker images properly** - Use both commit SHA (immutable) and latest (mutable)
6. **Version your actions** - Pin action versions (e.g., `@v4`) rather than major versions
7. **Test locally first** - Run Maven build and Docker build locally before pushing

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [SonarQube GitHub Action](https://github.com/SonarSource/sonarqube-scan-action)
- [Amazon ECR GitHub Action](https://github.com/aws-actions/amazon-ecr-login)
- [yq CLI Documentation](https://mikefarah.gitbook.io/yq/)
