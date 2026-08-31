# GitHub Actions Setup Checklist

Use this checklist to configure all required secrets and variables for the CI/CD pipeline.

## 1. GitHub Repository Secrets
Go to: **Settings → Secrets and variables → Actions → New repository secret**

- [ ] `SONAR_TOKEN`
  - Value: Your SonarQube authentication token
  - How to get: SonarQube Dashboard → My Account → Security → Generate Tokens
  
- [ ] `AWS_ACCESS_KEY_ID`
  - Value: AWS IAM user access key ID
  - How to get: AWS Console → IAM → Users → Select user → Security credentials → Create access key
  
- [ ] `AWS_SECRET_ACCESS_KEY`
  - Value: AWS IAM user secret access key (save this immediately, cannot be retrieved later)
  - How to get: Same as above
  
- [ ] `HELM_REPO_USER`
  - Value: GitHub username or organization that owns the Helm repo
  - Example: `salmanvohra-dev`
  
- [ ] `GITOPS_PAT`
  - Value: GitHub Personal Access Token with full `repo` scope
  - How to get: GitHub → Settings → Developer settings → Personal access tokens (classic) → Generate new token
  - Required scopes: `repo` (full control)
  
- [ ] `SLACK_WEBHOOK` (Optional)
  - Value: Slack webhook URL for notifications
  - How to get: Slack workspace → App Directory → Incoming Webhooks → Add to Workspace

## 2. GitHub Repository Variables
Go to: **Settings → Secrets and variables → Variables → New repository variable**

- [ ] `AWS_REGION`
  - Value: `us-east-1`
  - Description: AWS region where ECR repository is located
  
- [ ] `ECR_REPOSITORY`
  - Value: `vprofileappimg`
  - Description: Name of the ECR repository
  
- [ ] `HELM_REPO_NAME`
  - Value: `vprofile-helm`
  - Description: Name of the Helm chart repository (GitHub repo name)
  
- [ ] `SONAR_HOST_URL`
  - Value: `http://your-sonar-server:9000` or `https://sonar.example.com`
  - Description: Self-hosted SonarQube server URL
  - Note: Must be accessible from GitHub Actions runners

## 3. Project File Checks
- [ ] `pom.xml` exists with correct Java 21 configuration
- [ ] `Docker-files/app/multistage/Dockerfile` exists
- [ ] `sonar-project.properties` exists in project root
- [ ] `.gitignore` excludes `target/` and `.sonar/` directories

## 4. AWS Configuration
- [ ] ECR repository `vprofileappimg` exists in `us-east-1`
  ```bash
  aws ecr create-repository \
    --repository-name vprofileappimg \
    --region us-east-1
  ```
  
- [ ] IAM user has ECR permissions
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
        "Resource": "*"
      }
    ]
  }
  ```

## 5. SonarQube Configuration
- [ ] SonarQube server is running and accessible
- [ ] Project `vprofile-app` exists in SonarQube
- [ ] Authentication token generated
- [ ] Quality gate configured (or using default)

## 6. Helm Repository Configuration
- [ ] Separate GitHub repository created: `vprofile-helm`
- [ ] Repository structure:
  ```
  vprofile-helm/
  ├── helm/
  │   └── vprofile/
  │       ├── Chart.yaml
  │       ├── values.yaml (with app.image and app.tag fields)
  │       └── templates/
  ```
- [ ] Initial `values.yaml` structure:
  ```yaml
  app:
    image: 
    tag: 
    replicas: 1
    containerPort: 8080
    servicePort: 8080
  ```

## 7. Branch Protection Rules
Go to: **Settings → Branches → Add rule** (for `main` branch)

- [ ] Require pull request reviews before merging
- [ ] Require status checks to pass before merging
  - Select `build-and-sonar` as required status check
- [ ] Require branches to be up to date before merging
- [ ] Require code quality checks to pass
- [ ] Restrict who can push to matching branches (optional)

## 8. Testing the Pipeline

### Test PR Build (build-and-sonar)
1. Create a feature branch
2. Make a small code change
3. Create a Pull Request to `main`
4. Verify workflow runs in Actions tab
5. Check that all checks pass

### Test Main Push (docker-build-push + update-helm)
1. Merge PR to `main`
2. Verify `docker-build-push` job completes
3. Check ECR repository for new image tags (commit SHA and `latest`)
4. Check `vprofile-helm` repository for new PR with updated values

## 9. Verify Workflow Execution

### For Pull Requests:
- [ ] Workflow triggers on PR creation/update
- [ ] Maven build completes successfully
- [ ] Unit tests pass
- [ ] Checkstyle validation passes
- [ ] SonarQube scan completes
- [ ] Quality gate check passes or fails appropriately
- [ ] GitHub shows status check on PR

### For Main Branch Push:
- [ ] Docker image builds and pushes to ECR
- [ ] Image tagged with commit SHA
- [ ] Image tagged with `latest`
- [ ] Helm values repository receives PR
- [ ] PR updates `app.image` and `app.tag` correctly

## Troubleshooting Command Reference

### Test AWS Credentials
```bash
aws ecr describe-repositories --region us-east-1
```

### Test SonarQube Connection
```bash
curl -u admin:SONAR_TOKEN http://SONAR_HOST_URL/api/system/health
```

### Test GitHub PAT
```bash
curl -H "Authorization: token GITOPS_PAT" https://api.github.com/user
```

### View ECR Images
```bash
aws ecr describe-images \
  --repository-name vprofileappimg \
  --region us-east-1
```

## Quick Links
- Repository Actions: `https://github.com/YOUR_ORG/vprofile-app/actions`
- Secrets Settings: `https://github.com/YOUR_ORG/vprofile-app/settings/secrets/actions`
- Variables Settings: `https://github.com/YOUR_ORG/vprofile-app/settings/variables/actions`
- Branch Protection: `https://github.com/YOUR_ORG/vprofile-app/settings/branches`

---

**After completing all checkboxes, your CI/CD pipeline is ready for use!**
