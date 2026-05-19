# test-release-process

Test repository for validating the automated release process workflows. This repo mirrors the folder structure of the production repository and contains the GitHub Actions workflows, composite actions, and branch setup needed to validate the automation end-to-end.

---

> **⚠️ IMPORTANT:** For the sake of testing, the required number of reviewers to trigger deployment is currently set to **1**. Only one approval is needed to trigger deployments to production on `release/` or `hotfix/` branches (instead of the 2 required in the real repository).

---

## Scenarios to Validate

### Prerequisites - Already configured

- The repository has `main` and `develop` branches
- The composite actions exist in `.github/actions/` (`retrieve-version`, `calculate-release-version`, `bump-version`)
- The workflows exist in `.github/workflows/` (`create-release.yml`, `create-github-release.yml`, modified `main.yml`)
- A GitHub Environment named `production` (or equivalent) is configured with required reviewers for the approval gate

---

### Scenario 1: Happy Path (Auto Version)

**Goal:** Verify the full release flow with automatic version calculation.

**Steps:**
1. Ensure `main` has a known version in `package.json` (e.g., `1.11.0`)
2. Go to Actions → `Create Release` → Run workflow (leave `version` input **empty**)
3. Verify:
   - Version is auto-calculated as `1.12.0` (minor bump, patch reset)
   - Branch `release/1.12.0` is created from `develop`
   - `package.json` on the release branch is bumped to `1.12.0` with commit message `Release: Bump version to 1.12.0`
   - PR to `main` created with title `Release: 1.12.0 [main]` and `release` label
   - PR to `develop` created with title `Release: 1.12.0 [develop]` and `release` label
   - Workflow pauses at the approval gate ("Review deployments" button appears)
4. Approve the PR to `main` with 2 reviewers → PROD deployment triggers
5. After deployment, click "Review deployments" to approve continuation
6. Verify both PRs are merged (merge strategy, not squash)
7. Verify `create-github-release.yml` triggers → tag `1.12.0` and GitHub Release are created on `main`

---

### Scenario 2: Happy Path (Manual Override)

**Goal:** Verify the release flow when a version is explicitly provided.

**Steps:**
1. Go to Actions → `Create Release` → Run workflow with `version` input set to `2.0.0`
2. Verify:
   - The provided version `2.0.0` is used (no auto-calculation)
   - Branch `release/2.0.0` is created from `develop`
   - `package.json` bumped to `2.0.0`
   - PR titles use `2.0.0`
   - Rest of the flow is identical to Scenario 1

---

### Scenario 3: Retrieve Version

**Goal:** Verify that `retrieve-version` correctly reads the version from `package.json` on a given branch.

**Steps:**
1. Ensure `main`'s `package.json` has a known version (e.g., `1.11.0`)
2. Go to Actions → `Test Retrieve Version` → Run workflow
3. Check the `test-retrieve` job output
4. Verify it prints the correct version matching `main`'s `package.json`

**Workflow:** `.github/workflows/test-retrieve-version.yml`

---

### Scenario 4: Version Calculation (Release & Hotfix)

**Goal:** Verify that `calculate-release-version` correctly calculates the next version for both release (minor bump, patch reset) and hotfix (patch bump only).

**Steps:**
1. Ensure `main`'s `package.json` has a known version (e.g., `1.5.3`)
2. Go to Actions → `Test Calculate Release Version` → Run workflow
3. Check both job outputs:
   - `test-calculate-release`: should output `1.6.0` (minor bumped, patch reset)
   - `test-calculate-hotfix`: should output `1.5.4` (only patch bumped)
4. Verify neither produces an incorrect value (e.g., `1.5.4` for release, or `1.6.0` for hotfix)

**Workflow:** `.github/workflows/test-calculate-release-version.yml`

---

### Scenario 5: Bump Version

**Goal:** Verify that `bump-version` updates `package.json`, runs `npm install`, and commits with the correct message.

**Steps:**
1. **Create a dedicated test branch first** (e.g., `test/bump-version`) from `develop` to avoid polluting `develop`:
   ```bash
   git checkout develop
   git checkout -b test/bump-version
   git push -u origin test/bump-version
   ```
2. Go to Actions → `Test Bump Version` → Run workflow **from the `test/bump-version` branch**
3. Check the `test-bump` job output
4. Verify:
   - `package.json` on `test/bump-version` is updated to `1.99.0`
   - `package-lock.json` is also updated
   - A commit exists with message `Release: Bump version to 1.99.0`
   - The commit is pushed to the branch

**Workflow:** `.github/workflows/test-bump-version.yml`

**Cleanup:** Delete the test branch after verifying:
```bash
git push origin --delete test/bump-version
git branch -D test/bump-version
```

---

### Scenario 6: Conflict Scenario

**Goal:** Verify the workflow stops and reports when PRs have merge conflicts.

**Setup — Create a conflict:**
1. On `main`, change the `name` field in `package.json` to `"test-release-main"` and push:
   ```bash
   git checkout main
   sed -i '' 's/"name": ".*"/"name": "test-release-main"/' package.json
   git add package.json && git commit -m "Change package name on main" && git push
   ```
2. On `develop`, change the same `name` field to `"test-release-develop"` and push:
   ```bash
   git checkout develop
   sed -i '' 's/"name": ".*"/"name": "test-release-develop"/' package.json
   git add package.json && git commit -m "Change package name on develop" && git push
   ```
   This guarantees the release branch (created from `develop`) will conflict with `main` on `package.json`.

**Steps:**
1. Trigger the `Create Release` workflow (leave version empty)
2. The workflow will create the release branch and PRs normally (the `create-release` job succeeds)
3. The workflow will pause at the `approve-and-merge` job waiting for environment approval — click "Review deployments" to approve
4. After approval, the `approve-and-merge` job runs and checks mergeability before attempting to merge
5. Verify:
   - The `approve-and-merge` job **fails** at the conflict check step
   - The error annotation indicates which PR has conflicts
   - Neither PR is merged (no partial merge)

**Cleanup:** After testing, set the `name` field back to `"test-release-process"` on both branches:
```bash
git checkout main
sed -i '' 's/"name": ".*"/"name": "test-release-process"/' package.json
git add package.json && git commit -m "Restore package name on main" && git push

git checkout develop
sed -i '' 's/"name": ".*"/"name": "test-release-process"/' package.json
git add package.json && git commit -m "Restore package name on develop" && git push
```

---

### Scenario 7: Hotfix Deployment Trigger

**Goal:** Verify that a PR from a `hotfix/` branch targeting `main` triggers PROD deployment on 2 approvals. 

**Steps:**
1. Create a branch `hotfix/1.11.1` from `main`
2. Make a change and push
3. Open a PR from `hotfix/1.11.1` → `main`
4. Have 2 reviewers approve the PR
5. Verify:
   - The modified `main.yml` detects the `pull_request_review` event
   - PROD deployment is triggered (same logic as release PRs)
   - Changed services are detected and deployed

---

### Scenario 8: Partial Failure (Deployment Fails)

**Goal:** Verify the workflow remains paused if approval is never given.

**Steps:**
1. Trigger the `Create Release` workflow
2. Let the workflow reach the approval gate
3. **Do NOT approve** — simply leave it
4. Verify:
   - The workflow remains in "Waiting" state indefinitely
   - PRs are **not** merged
   - No tag or release is created
   - The workflow can be cancelled manually if needed

