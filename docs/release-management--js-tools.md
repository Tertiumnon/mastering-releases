# Release Management — Technical Implementation Guide

This document contains technical implementations, CI/CD configurations, automation scripts, and developer-specific details. Refer to [release-management.md](./release-management.md) for high-level principles and workflows.

---

## CI/CD Pipeline Configurations

### GitHub Actions Workflows

#### Release Staging Workflow

Create `.github/workflows/release-staging.yml`:

```yaml
name: Release to Staging

on:
  push:
    branches:
      - 'release/*'

jobs:
  release-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build

      - name: Deploy to staging
        run: |
          # Your staging deployment logic here
          echo "Deploying to staging environment"
        env:
          STAGING_API_URL: ${{ secrets.STAGING_API_URL }}
          STAGING_DATABASE_URL: ${{ secrets.STAGING_DATABASE_URL }}

      - name: Create release PR
        if: github.event_name == 'push' && github.ref == 'refs/heads/release/*'
        run: |
          # Create PR from release branch to main
          gh pr create --base main --title "Release ${{ github.sha }}" --body "Staging deployment"
```

#### Production Deployment Workflow

Create `.github/workflows/release-production.yml`:

```yaml
name: Release to Production

on:
  push:
    tags:
      - 'v*'

jobs:
  release-production:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')

    steps:
      - uses: actions/checkout@v4

      - name: Verify tag format
        run: |
          TAG="${{ github.ref }}"
          if [[ ! "$TAG" =~ ^refs/tags/v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Invalid tag format: $TAG"
            exit 1
          fi

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run production tests
        run: npm run test:production

      - name: Build production
        run: npm run build:production

      - name: Validate release metadata
        run: |
          if [ -f release.json ]; then
            npx ajv validate -d schemas/release-metadata.schema.json release.json
          else
            echo "No release.json found, skipping validation"
          fi

      - name: Deploy to production
        run: |
          # Your production deployment logic here
          echo "Deploying to production environment"
        env:
          PROD_API_URL: ${{ secrets.PROD_API_URL }}
          PROD_DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
          RELEASE_VERSION: ${{ github.ref_name }}

      - name: Notify stakeholders
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🚀 Production deployment of ${{ github.ref_name }} completed successfully!`
            })
```

#### Hotfix Deployment Workflow

Create `.github/workflows/hotfix.yml`:

```yaml
name: Hotfix Deployment

on:
  push:
    branches:
      - 'hotfix/*'

jobs:
  hotfix-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run critical tests
        run: npm run test:critical

      - name: Build
        run: npm run build

      - name: Deploy to production (hotfix)
        run: |
          # Hotfix deployment logic
          echo "Deploying hotfix to production"
        env:
          PROD_API_URL: ${{ secrets.PROD_API_URL }}
          HOTFIX_VERSION: ${{ github.ref_name }}

      - name: Notify team
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `🔥 Hotfix deployed to production! Branch: ${{ github.ref_name }}`
            })
```

---

## Automation Scripts

### Release Script (Node.js)

Create `scripts/release.js`:

```javascript
#!/usr/bin/env node

const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

const VERSION = process.argv[2] || require('../package.json').version;

console.log(`Creating release branch for ${VERSION}`);

try {
  execSync('git checkout develop');
  execSync(`git checkout -b release/${VERSION}`);

  // Generate changelog
  const changelog = execSync(
    `git log --pretty=format:'- %s [%h]' develop..release/${VERSION}`,
    { encoding: 'utf8' }
  );

  // Create release.json
  const releaseData = {
    version: VERSION,
    date: new Date().toISOString(),
    releasedBy: process.env.GITHUB_ACTTOR || 'manual',
    artifacts: [],
    changelog: changelog.trim().split('\n').filter(Boolean)
  };

  fs.writeFileSync(
    path.join(__dirname, '../release.json'),
    JSON.stringify(releaseData, null, 2)
  );

  console.log(`Created release.json for ${VERSION}`);
} catch (error) {
  console.error('Release creation failed:', error.message);
  process.exit(1);
}
```

### Version Bumper Script

Create `scripts/bump-version.js`:

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

function bumpVersion(currentVersion, bumpType) {
  const [major, minor, patch] = currentVersion.split('.').map(Number);

  switch (bumpType) {
    case 'major':
      return `${major + 1}.0.0`;
    case 'minor':
      return `${major}.${minor + 1}.0`;
    case 'patch':
      return `${major}.${minor}.${patch + 1}`;
    default:
      throw new Error(`Invalid bump type: ${bumpType}`);
  }
}

function main() {
  const packageJsonPath = path.join(__dirname, '../package.json');
  const packageJson = JSON.parse(fs.readFileSync(packageJsonPath, 'utf8'));

  const currentVersion = packageJson.version;
  const bumpType = process.argv[2] || 'patch';

  const newVersion = bumpVersion(currentVersion, bumpType);

  packageJson.version = newVersion;
  fs.writeFileSync(packageJsonPath, JSON.stringify(packageJson, null, 2));

  console.log(`Bumped version from ${currentVersion} to ${newVersion}`);
}

main();
```

### Changelog Generator

Create `scripts/generate-changelog.js`:

```javascript
#!/usr/bin/env node

const { execSync } = require('child_process');
const fs = require('fs');
const path = require('path');

function generateChangelog(fromVersion, toVersion) {
  const commits = execSync(
    `git log ${fromVersion}..${toVersion} --pretty=format:"%h|%s|%b"`,
    { encoding: 'utf8' }
  );

  const entries = [];
  const commitMap = new Map();

  commits.split('\n').forEach(line => {
    if (!line) return;
    const [hash, subject, body] = line.split('|');
    const parts = subject.split(' ');
    const type = parts[0] || 'chore';
    const scope = parts[1] || '';
    const description = parts.slice(2).join(' ') || '';

    const key = `${type}${scope ? `(${scope})` : ''}: ${description}`;

    if (!commitMap.has(key)) {
      commitMap.set(key, []);
    }
    commitMap.get(key).push(hash);
  });

  const changelog = Array.from(commitMap.entries())
    .map(([key, hashes]) => `- ${key} (${hashes.join(', ')})`)
    .join('\n');

  return changelog;
}

function main() {
  const fromVersion = process.argv[2] || 'v1.0.0';
  const toVersion = process.argv[3] || 'HEAD';

  const changelog = generateChangelog(fromVersion, toVersion);

  fs.writeFileSync('CHANGELOG.md', changelog);
  console.log('Generated CHANGELOG.md');
}

main();
```

---

## Environment Configuration Templates

### Environment Variables Template

Create `.env.release.template`:

```bash
# Production environment
NODE_ENV=production
API_URL=https://api.production.example.com
DATABASE_URL=postgresql://prod-db.example.com
REDIS_URL=redis://prod-redis.example.com

# Feature flags
ENABLE_NEW_AUTH=false
ENABLE_BETA_FEATURES=false

# Release metadata
RELEASE_VERSION=1.2.0
RELEASE_BRANCH=main
RELEASE_TAG=v1.2.0

# Monitoring
SENTRY_DSN=https://sentry.example.com/123
LOG_LEVEL=info
```

### Environment-Specific Configurations

Create `config/` directory structure:

```
config/
  development.json
  staging.json
  production.json
  release.json
```

Example `config/staging.json`:

```json
{
  "api": {
    "baseUrl": "https://api.staging.example.com",
    "timeout": 30000
  },
  "database": {
    "host": "staging-db.example.com",
    "port": 5432,
    "pool": {
      "min": 2,
      "max": 10
    }
  },
  "features": {
    "newAuth": true,
    "betaFeatures": true,
    "maintenanceMode": false
  },
  "monitoring": {
    "sentryDsn": "https://sentry.staging.example.com/123",
    "logLevel": "debug"
  }
}
```

### Docker Environment Files

Create `docker-compose.staging.yml`:

```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        ENVIRONMENT: staging
        NODE_ENV: staging
    environment:
      - NODE_ENV=staging
      - API_URL=https://api.staging.example.com
    deploy:
      replicas: 2
```

Create `docker-compose.production.yml`:

```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        ENVIRONMENT: production
        NODE_ENV: production
    environment:
      - NODE_ENV=production
      - API_URL=https://api.production.example.com
    deploy:
      replicas: 4
      resources:
        limits:
          cpus: '4'
          memory: 4G
```

---

## Library and Tool Recommendations

### Release Automation Tools

| Tool | Purpose | When to Use |
|------|---------|-------------|
| **semantic-release** | Automated releases from commits | Standard projects with conventional commits |
| **changesets** | Version management for monorepos | Monorepos with multiple packages |
| **release-it** | CLI release automation | Simple projects, custom workflows |
| **daisyui** | UI component library | Not related to releases |

### Version Management Libraries

- **npm**: Use `npm version` for version bumping
- **changesets**: For monorepos with `npx changeset version`
- **lerna**: For npm workspaces with `lerna version`

### Changelog Generation

- **conventional-changelog**: Parse conventional commits
  ```bash
  npx conventional-changelog -p conventional-changelog-angular -r 1000
  ```
- **semantic-release**: Automated changelog generation
  ```bash
  npx semantic-release
  ```

### Validation Tools

- **ajv**: JSON Schema validation
  ```bash
  npx ajv validate -d schemas/release-metadata.schema.json release.json
  ```
- **eslint**: Code quality checks
- **prettier**: Code formatting

---

## Advanced Troubleshooting Commands

### Issue: Tag already exists

```bash
# Force push tag (use with caution!)
git push origin --delete v1.2.0
git tag -f -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0 --force
```

### Issue: Merge conflicts in release branch

```bash
# Abort release
git checkout main
git merge --abort

# Or merge with conflict resolution
git merge release/0.2.0 -m "Merge release 0.2.0 (conflicts resolved)"
```

### Issue: Staging deployment fails

1. Check CI logs for build errors
2. Verify environment variables are set
3. Ensure artifact paths are correct
4. Review rollback procedure

### Issue: Production deployment blocked

- Ensure tag follows pattern `v*`
- Verify CI pipeline status is green
- Check release metadata is valid against schema
- Confirm QA sign-off is documented

### Debug: Check deployment status

```bash
# Check running containers
docker ps

# Check logs
docker logs <container_id>

# Check deployment artifacts
ls -la dist/
```

### Debug: Validate release metadata

```bash
# Validate against schema
npx ajv validate -d schemas/release-metadata.schema.json release.json

# Pretty print release.json
cat release.json | jq .
```

---

## Security Considerations

### Security checklist

- [ ] Dependencies scanned for vulnerabilities
- [ ] No sensitive data in release artifacts
- [ ] Environment variables not committed to repository
- [ ] Secrets rotated after production deployment
- [ ] Release notes don't expose internal details
- [ ] Rollback plan tested before release
- [ ] Security team notified for major releases

### Security scanning

```bash
# Scan dependencies
npm audit

# Scan for vulnerabilities
npm audit report

# SAST scanning
npm install -D @commitlint/cli
```

---

## Monitoring and Observability

### Monitoring checklist

- [ ] Set up release-specific dashboards
- [ ] Configure alerts for error rate spikes
- [ ] Monitor performance metrics (latency, throughput)
- [ ] Set up release notes in documentation
- [ ] Prepare rollback procedure
- [ ] Document known issues

### Example: Prometheus metrics

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app:3000']
```

---

## Migration Guide

### Migrating from GitFlow to trunk-based

1. **Phase 1: Preparation** (1 week)
   - Document current workflow
   - Train team on new practices
   - Set up CI/CD pipelines

2. **Phase 2: Parallel run** (2-4 weeks)
   - Continue GitFlow releases
   - Start trunk-based development
   - Merge trunk to main daily

3. **Phase 3: Transition** (1-2 weeks)
   - Stop creating release branches
   - Use feature flags for long-running features
   - Deploy from main with tags

4. **Phase 4: Cleanup** (1 week)
   - Delete old release branches
   - Update documentation
   - Celebrate success!

---

## Resources

### Books
- [Release It! by Michael Nygard](https://www.amazon.com/Release-It-Nygard/dp/032139923X)
- [Continuous Delivery by Jez Humble](https://www.amazon.com/Continuous-Delivery-Deployment-Software-Change/dp/0321601417)

### Articles
- [GitFlow vs GitHub Flow vs Trunk-Based Development](https://www.atlassian.com/agile/software-development/gitflow)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)

### Tools
- [GitHub Releases](https://docs.github.com/repositories/releasing-projects-on-github/about-releases)
- [npm Dist Tags](https://docs.npmjs.com/cli/v9/configuring-npm/package-json#dist-tags)
- [Docker Versioning](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#version-labels)

---

## Next steps

- ✅ Review your current release workflow
- ✅ Define automation principles for versioning and changelog generation
- ✅ Set up CI/CD pipeline for staging and production
- ✅ Create environment-specific configurations
- ✅ Configure monitoring and alerting
- ✅ Document rollback procedures
- ✅ Train team on new workflow
- ✅ Schedule first release

## Recommended release workflow (example)

1. Create a release branch from `develop` when feature-complete:

```pwsh
git checkout develop
git pull origin develop
git checkout -b release/1.2.0
```

2. Stabilize: fixes, docs, bump version, update changelog.

3. Run full CI and deploy to staging for QA. Example: GitHub Actions job `release-staging`.

4. After QA sign-off, merge to `main`, tag and push the tag. Use annotated tags:

```pwsh
git checkout main
git pull origin main
git merge --no-ff release/1.2.0 -m "Merge release 1.2.0"
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin main --follow-tags
```

5. Merge the release back into `develop` to keep fixes:

```pwsh
git checkout develop
git merge --no-ff release/1.2.0 -m "Merge release 1.2.0 back into develop"
git push origin develop
```

6. Delete the release branch (optional):

```pwsh
git branch -d release/1.2.0
git push origin --delete release/1.2.0
```
