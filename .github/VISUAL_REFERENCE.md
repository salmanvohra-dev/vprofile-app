# GitHub Actions CI/CD Pipeline - Visual Reference

## Pipeline Trigger Flow

### Scenario 1: Push to Feature Branch
```
Developer Push to feature/my-change
           │
           ▼
    GitHub Event
           │
           ▼
   Evaluate Triggers
           │
           ├─ on.push.branches: [main] ✗ (branch is feature/*)
           └─ on.pull_request.branches: [main] ✗ (not a PR yet)
           │
           ▼
   ❌ NO JOBS RUN
```

### Scenario 2: Create Pull Request to Main
```
Developer creates PR: feature/my-change → main
           │
           ▼
    GitHub Event (pull_request)
           │
           ▼
   Trigger Workflow
           │
           ▼
   Evaluate Job Conditions
           │
           ├─ build-and-sonar:
           │  ├─ on.pull_request.branches: [main] ✓
           │  ├─ if: github.event_name == 'pull_request' ✓
           │  ├─ if: github.base_ref == 'main' ✓
           │  └─ ✅ JOB RUNS
           │
           ├─ docker-build-push:
           │  ├─ if: github.ref == 'refs/heads/main' ✗ (branch is refs/pull/*/merge)
           │  └─ ❌ SKIPPED
           │
           └─ update-helm:
              ├─ needs: docker-build-push (skipped)
              └─ ❌ SKIPPED
```

**build-and-sonar Job Steps:**
```
1. Checkout code (fetch-depth: 0 for SonarQube)
2. Set up JDK 21
3. Cache SonarQube packages (~/.sonar/cache)
4. Cache Maven packages (~/.m2)
5. Run: mvn clean verify -DskipITs
6. Run: mvn checkstyle:check
7. Run: SonarQube scan (sonarsource/sonarqube-scan-action@v2)
8. Run: SonarQube quality gate check (sonarsource/sonarqube-quality-gate-action@v1.1.0)

Result:
├─ If ALL pass → PR shows ✅ (merge allowed)
└─ If ANY fail → PR shows ❌ (merge blocked)
```

### Scenario 3: Merge PR to Main
```
Developer merges PR: feature/my-change → main
           │
           ▼
    GitHub Event (push)
           │
           ▼
   Trigger Workflow
           │
           ▼
   Evaluate Job Conditions
           │
           ├─ build-and-sonar:
           │  ├─ if: github.event_name == 'pull_request' ✗ (event is push)
           │  └─ ❌ SKIPPED
           │
           ├─ docker-build-push:
           │  ├─ on.push.branches: [main] ✓
           │  ├─ if: github.ref == 'refs/heads/main' ✓
           │  ├─ if: github.event_name == 'push' ✓
           │  └─ ✅ JOB RUNS
           │
           └─ update-helm:
              ├─ if: github.ref == 'refs/heads/main' ✓
              ├─ if: github.event_name == 'push' ✓
              ├─ needs: docker-build-push (running) ✓
              └─ ✅ WAITS FOR docker-build-push, THEN RUNS
```

**docker-build-push Job Steps:**
```
1. Checkout code
2. Set up JDK 21
3. Cache Maven packages (~/.m2)
4. Build WAR: mvn clean package -DskipTests
5. Configure AWS credentials (from secrets)
6. Login to Amazon ECR
7. Build Docker image:
   - docker build -f Docker-files/app/multistage/Dockerfile
   - Tag: {ecr_registry}/{ecr_repo}:{sha}
   - Tag: {ecr_registry}/{ecr_repo}:latest
8. Push both tags to ECR
9. Output: image_tag, ecr_registry, full_image_uri

Result: Docker images pushed to ECR ✅
```

**update-helm Job Steps:**
```
Runs ONLY after docker-build-push succeeds

1. Checkout Helm repository (using GITOPS_PAT)
2. Install yq CLI tool
3. Update helm/vprofile/values.yaml:
   - Set: app.image = {ecr_registry}/{ecr_repo}
   - Set: app.tag = {commit_sha}
4. Create Pull Request in vprofile-helm repo:
   - Branch: update-image-{commit_sha}
   - Title: "chore: update vprofile app image - {commit_sha}"
   - Auto-delete branch after merge
5. PR shows changes for manual review

Result: PR created in vprofile-helm repo ✅
```

---

## Conditional Logic Reference

### Job Conditions

**build-and-sonar runs when:**
```yaml
if: github.event_name == 'pull_request' && github.base_ref == 'main'
```
- GitHub event is `pull_request` (not `push`)
- AND target branch (`base_ref`) is `main`

**docker-build-push runs when:**
```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```
- Current ref (`github.ref`) is exactly `refs/heads/main`
- AND GitHub event is `push` (not `pull_request`)

**update-helm runs when:**
```yaml
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
needs: docker-build-push
```
- Same as docker-build-push conditions
- PLUS it requires successful completion of `docker-build-push`
- If docker-build-push is skipped or fails, update-helm is skipped

---

## GitHub Context Variables

| Variable | Value | Example |
|----------|-------|---------|
| `github.event_name` | Type of event | `push`, `pull_request`, `push` |
| `github.ref` | Full ref path | `refs/heads/main`, `refs/pull/123/merge` |
| `github.base_ref` | Target branch (PRs) | `main` |
| `github.head_ref` | Source branch (PRs) | `feature/my-change` |
| `github.sha` | Commit SHA | `abc123def456789...` |
| `github.repository` | Repo path | `salmanvohra-dev/vprofile-app` |
| `github.run_id` | Workflow run ID | `12345678` |

---

## Caching Strategy Details

### Maven Cache

**Activated in:** build-and-sonar, docker-build-push

**Configuration:**
```yaml
uses: actions/cache@v4
with:
  path: ~/.m2
  key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
  restore-keys: ${{ runner.os }}-m2
```

**How it works:**
1. `key` uses hash of all `pom.xml` files
2. If exact hash matches previous run → cache restored (fast)
3. If hash differs → downloads dependencies (slower, but saves them)
4. `restore-keys` falls back to any previous m2 cache if exact match not found

**Speed improvement:** 60-70% faster builds on average

**Cache location:** `~/.m2/repository/` (Maven local repository)

### SonarQube Cache

**Activated in:** build-and-sonar only

**Configuration:**
```yaml
uses: actions/cache@v4
with:
  path: ~/.sonar/cache
  key: ${{ runner.os }}-sonar
  restore-keys: ${{ runner.os }}-sonar
```

**How it works:**
1. Caches SonarQube scanner downloaded files
2. Same for all runs (simple key-value)
3. Reused on every PR to same branch

**Speed improvement:** Faster SonarQube analysis on repeated runs

---

## Job Output Passing

### From docker-build-push

Job definition:
```yaml
outputs:
  image_tag: ${{ steps.image.outputs.tag }}
  ecr_registry: ${{ steps.login-ecr.outputs.registry }}
  full_image_uri: ${{ steps.image.outputs.full_uri }}
```

Set in step:
```bash
echo "tag=$IMAGE_TAG" >> $GITHUB_OUTPUT
echo "full_uri=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
```

### To update-helm

Access in update-helm job:
```yaml
env:
  IMAGE: ${{ needs.docker-build-push.outputs.ecr_registry }}/${{ env.ECR_REPOSITORY }}
  IMAGE_TAG: ${{ needs.docker-build-push.outputs.image_tag }}
```

**Examples:**
- `IMAGE` → `123456789.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg`
- `IMAGE_TAG` → `abc123def456` (commit SHA)
- Combined → `123456789.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg:abc123def456`

---

## Environment Variables vs Secrets vs Variables

### Environment Variables (Workflow Level)

```yaml
env:
  AWS_REGION: ${{ vars.AWS_REGION }}
  ECR_REPOSITORY: ${{ vars.ECR_REPOSITORY }}
  HELM_REPO_NAME: ${{ vars.HELM_REPO_NAME }}
  SONAR_HOST_URL: ${{ vars.SONAR_HOST_URL }}
```

- Defined at workflow level
- Available in all jobs and steps
- Reference using `${{ env.VAR_NAME }}`
- Values come from GitHub Variables

### Secrets

```yaml
env:
  SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

- Stored in GitHub Secrets (encrypted)
- Never logged or displayed
- Reference using `${{ secrets.SECRET_NAME }}`
- Available in all jobs and steps

### GitHub Variables

```yaml
env:
  AWS_REGION: ${{ vars.AWS_REGION }}
```

- Stored in GitHub Variables (visible to admins)
- Non-sensitive configuration
- Reference using `${{ vars.VAR_NAME }}`
- Better for URLs, region names, repo names

**Security principle:**
- Sensitive data (tokens, keys, passwords) → Secrets
- Non-sensitive config (URLs, regions, names) → Variables

---

## Docker Image Tagging Strategy

### On docker-build-push job:

```bash
IMAGE_TAG=${{ github.sha }}  # Commit SHA, e.g., abc123def456

docker build ... \
  -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \    # Immutable
  -t $ECR_REGISTRY/$ECR_REPOSITORY:latest .        # Mutable

docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
```

**Results in ECR:**
```
Repository: vprofileappimg
├─ Image: :abc123def456 (immutable, linked to commit)
└─ Image: :latest (mutable, always points to latest main push)
```

**Benefits:**
- SHA tag allows rollback to exact commit
- latest tag for quick deployments
- Both tags point to same image layer
- SonarQube quality gate requirement ensures quality before image push

---

## Complete Job Dependency Graph

```
GitHub Event
    │
    ├─ PR to main
    │   │
    │   └─ build-and-sonar ────────────┐
    │       │                          │
    │       ├─ Maven Build             │
    │       ├─ Unit Tests              ├─ If FAILS: PR blocked
    │       ├─ Checkstyle              │
    │       ├─ SonarQube Scan          │
    │       └─ Quality Gate Check      │
    │                                  │
    └─────────────────────────────────┘

GitHub Event
    │
    ├─ Push to main
    │   │
    │   └─ docker-build-push ──────────┐
    │       │                          │
    │       ├─ Maven Build (WAR)       │
    │       ├─ Docker Build            │
    │       ├─ ECR Login               ├─ If succeeds:
    │       ├─ Push to ECR             │    Outputs: image_tag,
    │       │   ├─ tag: {sha}          │    ecr_registry,
    │       │   └─ tag: latest         │    full_image_uri
    │       └─ Output values           │
    │                                  │
    │       (outputs available)        │
    │              │                   │
    │              ▼                   │
    │       update-helm ───────────────┤
    │       │                          │
    │       ├─ Clone Helm repo         ├─ If docker-build-push
    │       ├─ Install yq              │    FAILS or SKIPPED:
    │       ├─ Update values.yaml      │    This job is SKIPPED
    │       ├─ Commit changes          │
    │       └─ Create PR in helm repo  │
    │                                  │
    └─────────────────────────────────┘
```

---

## GitHub Context Decision Tree

```
GitHub Push/PR Event
│
├─ Is event_name == 'pull_request'?
│  ├─ YES: Is base_ref == 'main'?
│  │  ├─ YES: ✅ Run build-and-sonar
│  │  └─ NO: ❌ Skip all jobs
│  │
│  └─ NO: Continue...
│
├─ Is event_name == 'push'?
│  ├─ YES: Is ref == 'refs/heads/main'?
│  │  ├─ YES: ✅ Run docker-build-push
│  │  │       ✅ Run update-helm (after docker-build-push succeeds)
│  │  └─ NO: ❌ Skip all jobs
│  │
│  └─ NO: ❌ Skip all jobs
│
└─ No other events trigger this workflow
```

---

## GitHub Workflow Status Badges (Optional)

Add to your README.md to show pipeline status:

```markdown
## CI/CD Status

[![CI/CD Pipeline](https://github.com/YOUR_ORG/vprofile-app/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/YOUR_ORG/vprofile-app/actions/workflows/ci.yml)
```

This displays the latest build status from the main branch.

---

## Comparison: Feature Branch vs PR vs Main

| Trigger | Event Type | build-and-sonar | docker-build-push | update-helm |
|---------|------------|:---:|:---:|:---:|
| Push to feature/* | push | ❌ | ❌ | ❌ |
| PR feature→main | pull_request | ✅ | ❌ | ❌ |
| Push to main | push | ❌ | ✅ | ✅ |

---

This visual reference should help you understand exactly when and why each job runs.
