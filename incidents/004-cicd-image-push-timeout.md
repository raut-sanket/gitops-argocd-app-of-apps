# Incident 004: CI/CD Image Push Timeout to Artifact Registry

**Date:** January 2024  
**Severity:** P3 (Build Pipeline)  
**Duration:** 45 minutes (pipeline blocked)  
**Impact:** Deployments delayed; no user-facing outage  

---

## Summary

GitHub Actions `docker push` step started timing out when pushing images to Google Artifact Registry. Builds that previously completed in 3-4 minutes were taking 15+ minutes and eventually failing with `context deadline exceeded`. This blocked all deployments for 45 minutes during a hotfix window.

## Detection

- **10:15 UTC** — Developer opens PR with critical swap-calculation fix
- **10:20 UTC** — GitHub Actions build succeeds but `docker push` step hangs at "Pushing layer..."
- **10:35 UTC** — Push times out after 15 minutes: `error pushing image: context deadline exceeded`
- **10:36 UTC** — Manual retry also fails

## Timeline

```
10:15  Hotfix PR opened for api-backend
10:20  Build completes; push begins, hangs on large layer
10:35  Push timeout (attempt 1)
10:38  Manual re-run (attempt 2) — same failure
10:42  Investigation: image size 1.8GB (was 450MB a week ago)
10:46  Root cause found: dev dependency + debug build flag left in Dockerfile
10:50  Dockerfile fixed; image back to 420MB
10:55  Push succeeds in 90 seconds
11:00  Hotfix deployed to production
```

## Root Cause

Two issues combined:

**1. Unintentional image bloat**

A developer added a debug flag and dev dependencies in a previous PR that was merged without noticing the image size jump:

```dockerfile
# Before (problematic)
FROM node:20
COPY package*.json ./
RUN npm install                # Installed devDependencies (800MB)
ENV NODE_OPTIONS="--inspect"   # Debug flag left in
COPY . .                       # Copied .git, node_modules, test fixtures
```

**2. No layer caching in GitHub Actions**

The workflow wasn't using Docker layer caching, so every push rebuilt and pushed ALL layers:

```yaml
# Before (no cache)
- name: Build and push
  run: |
    docker build -t $IMAGE .
    docker push $IMAGE
```

## Resolution

### Fix 1: Optimized multi-stage Dockerfile
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ src/

FROM node:20-alpine
RUN addgroup -g 1001 -S app && adduser -S app -u 1001
WORKDIR /app
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
COPY --from=builder --chown=app:app /app/src ./src
COPY package.json ./
USER app
EXPOSE 3000
CMD ["node", "src/index.js"]
```

Result: 1.8GB → 180MB (90% reduction)

### Fix 2: Added .dockerignore
```
.git
node_modules
*.md
tests/
.github/
coverage/
.env*
```

### Fix 3: Docker layer caching in GitHub Actions
```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ env.IMAGE }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Fix 4: Image size gate in CI
```yaml
- name: Check image size
  run: |
    SIZE=$(docker image inspect $IMAGE --format='{{.Size}}')
    SIZE_MB=$((SIZE / 1048576))
    echo "Image size: ${SIZE_MB}MB"
    if [ "$SIZE_MB" -gt 500 ]; then
      echo "::error::Image size ${SIZE_MB}MB exceeds 500MB limit"
      exit 1
    fi
```

## Monitoring Added

```yaml
# GitHub Actions workflow addition
- name: Report image metrics
  if: always()
  run: |
    echo "::notice::Build time: ${{ steps.build.outputs.duration }}s"
    echo "::notice::Image size: $(docker image inspect $IMAGE --format='{{.Size}}' | numfmt --to=iec)"
```

Grafana dashboard added tracking:
- Image sizes per service over time
- Build duration trends
- Push duration trends

## Preventive Measures

1. **Mandatory `.dockerignore`** — added to repo template
2. **Multi-stage builds** — all Dockerfiles converted; documented in team wiki
3. **Image size CI gate** — fails build if image exceeds 500MB
4. **Weekly audit** — cron job compares image sizes week-over-week:
   ```bash
   gcloud artifacts docker images list $REGISTRY --format=json \
     | jq '.[] | {package, size: (.sizeBytes/1048576 | floor)}' \
     | sort -k2 -nr
   ```
5. **PR checklist** updated: "Verify image size hasn't increased unexpectedly"

## Lessons Learned

- A 1-line dev dependency change caused a 4x image size increase that went unnoticed for a week.
- CI pipelines should enforce image size limits — it's a cheap check that catches real problems.
- Docker layer caching (`type=gha`) reduces push time by 60-80% for incremental changes.
- Multi-stage builds are non-negotiable for production images.
- The `.dockerignore` file is just as important as the `Dockerfile` itself.
