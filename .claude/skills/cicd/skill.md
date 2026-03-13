---
name: cicd
description: Set up or modify CI/CD pipelines using GitHub Actions and Fastlane for Android builds, tests, and deployments. Use "deploy" to auto commit, push, and trigger the pipeline.
argument-hint: "setup|deploy|add-lane|fix|status"
---

## CI/CD Pipeline Management with Fastlane (Android)

Manage GitHub Actions CI/CD pipelines that use Fastlane for building, testing, and deploying the Android app.

### Current State
- Android workflow: !`ls .github/workflows/android.yml 2>/dev/null && echo "✓ Found" || echo "✗ Not found"`
- Fastfile: !`ls fastlane/Fastfile 2>/dev/null && echo "✓ Found" || echo "✗ Not found"`
- Appfile: !`ls fastlane/Appfile 2>/dev/null && echo "✓ Found" || echo "✗ Not found"`
- Env example: !`ls fastlane/.env.example 2>/dev/null && echo "✓ Found" || echo "✗ Not found"`
- Git status: !`git status --short 2>/dev/null | head -5`
- Current branch: !`git branch --show-current`
- Remote: !`git remote -v 2>/dev/null | head -1`

### Arguments: ${ARGUMENTS:-status}

---

## Setup Flow (when argument is "setup")

When the user says `/cicd setup` or asks to set up CI/CD, handle EVERYTHING from scratch. This should work even on a brand new project that only has git initialized.

### Step 1: Check & Install Prerequisites
Run these checks and install anything missing:
```bash
gh --version          # If missing, tell user to install: https://cli.github.com
firebase --version    # If missing, run: npm install -g firebase-tools
bundle --version      # If missing, run: gem install bundler
git remote -v         # If no remote, tell user to create a GitHub repo first
```

### Step 2: Initialize Fastlane (if not exists)
- Check if `fastlane/Fastfile` exists
- If NOT, create the full Fastlane setup:
  - Create `Gemfile` with fastlane gem (if not exists)
  - Run `bundle install`
  - Create `fastlane/Appfile` with app identifiers
  - Create `fastlane/Fastfile` with these Android lanes:
    - `clean` — gradle clean
    - `build_debug` — assembleDebug
    - `build_release` — assembleRelease
    - `build_aab` — bundleRelease
    - `test` — run unit tests
    - `deploy_firebase` — build debug + upload to Firebase App Distribution
  - Create `fastlane/.env.example` documenting required env vars

### Step 3: Install Firebase Plugin
- Check if `fastlane/Pluginfile` exists and contains `firebase_app_distribution`
- If NOT, run: `bundle exec fastlane add_plugin firebase_app_distribution`
- Then run `bundle install` to update Gemfile.lock

### Step 4: Add deploy_firebase Lane (if not exists)
- Read `fastlane/Fastfile` and check if `deploy_firebase` lane exists
- If NOT, add it inside the `platform :android` block:
```ruby
desc "Deploy to Firebase App Distribution"
lane :deploy_firebase do
  build_debug
  firebase_app_distribution(
    app: ENV["FIREBASE_APP_ID"],
    release_notes: "CI build from commit #{`git log -1 --pretty=%h`.strip}"
  )
end
```
**IMPORTANT**: Do NOT include `groups` or `testers` parameters — the deploy lane should only upload the build to Firebase App Distribution, not invite or distribute to testers. The `groups` parameter causes an "Invalid request" error due to a known Fastlane plugin bug.

### Step 5: Create GitHub Actions Workflow (if not exists)
- Check if `.github/workflows/android.yml` exists
- If NOT, create it with 3 jobs:
  - **test**: checkout → setup Node → npm ci → npm test
  - **build**: checkout → setup Node + Java 17 + Ruby 3.2 → npm ci → chmod +x android/gradlew → fastlane android build_debug → upload APK artifact
  - **deploy** (only on push to main): checkout → setup Node + Java 17 + Ruby 3.2 → npm ci → chmod +x android/gradlew → fastlane android deploy_firebase (with FIREBASE_APP_ID and FIREBASE_TOKEN from secrets)

### Step 6: Firebase Setup
- Check if Firebase CLI is logged in: `firebase projects:list`
- If not logged in, run `firebase login:ci` and tell user to copy the token
- Ask user for their **Firebase App ID** (from Firebase Console → Project Settings → Your Apps → App ID like `1:123456789:android:abcdef`)
- If user doesn't have a Firebase project yet, guide them:
  1. Go to https://console.firebase.google.com
  2. Create project → Add Android app with package name
  3. Go to App Distribution to verify it's enabled

### Step 7: Set GitHub Secrets
- Ask user for `FIREBASE_TOKEN` and `FIREBASE_APP_ID` values
- Set them automatically:
```bash
gh secret set FIREBASE_TOKEN --body "the-token-value"
gh secret set FIREBASE_APP_ID --body "the-app-id-value"
```
- Verify: `gh secret list`

### Step 8: Final Verification
- Run `gh secret list` to confirm secrets exist
- Check all files are in place:
  - `Gemfile` ✓
  - `fastlane/Fastfile` with `deploy_firebase` lane ✓
  - `fastlane/Pluginfile` with `firebase_app_distribution` ✓
  - `.github/workflows/android.yml` with Firebase deploy ✓
- Show summary of everything that was set up
- Tell user: "Run `/cicd deploy` to commit, push, and trigger the pipeline"

---

## Auto Deploy Flow (when argument is "deploy")

When the user says `/cicd deploy` or asks to deploy, do ALL of these steps automatically:

1. **Check for changes**: Run `git status` to see what's changed
2. **Stage all changes**: Run `git add .` (but warn if .env or secret files are staged)
3. **Commit**: Create a commit with a descriptive message about the changes
4. **Push to main**: Run `git push origin main`
5. **Monitor pipeline**: Run `gh run list --limit 1` to show the triggered workflow
6. **Show status**: Keep checking `gh run view <run-id>` until the pipeline completes or fails
7. **Report result**: Tell the user if the pipeline passed or failed, and if the APK was deployed to Firebase

### Important:
- If not on `main` branch, ask the user if they want to merge to main first
- If there are no changes to commit, just push existing commits
- If push fails, show the error and suggest fixes
- Deploy job only triggers on push to `main` — remind user if they're on another branch

---

## Pipeline Architecture

This project uses a 3-stage Android pipeline:

```
PR / push to develop    push to main
       │                     │
   ┌───▼───┐            ┌───▼───┐
   │ Test  │            │ Test  │
   └───┬───┘            └───┬───┘
       │                     │
   ┌───▼───┐            ┌───▼───┐
   │ Build │            │ Build │
   └───────┘            └───┬───┘
                             │
                        ┌────▼────┐
                        │ Deploy  │
                        └─────────┘
```

- **Test**: Runs `npm test` (JS/TS unit tests)
- **Build**: Builds debug APK via Fastlane (`android build_debug`)
- **Deploy**: Only on `main` push — deploys to Firebase App Distribution

---

## Workflow File

### Android: `.github/workflows/android.yml`
- **Runner**: `ubuntu-latest`
- **Toolchain**: Node 22, Java 17 (Zulu), Ruby 3.2
- **Fastlane lanes used**: `android build_debug`, `android deploy_firebase`
- **Deploy trigger**: Push to `main` only
- **Secrets required**:
  - `FIREBASE_APP_ID` — Firebase App ID for App Distribution
  - `FIREBASE_TOKEN` — Firebase CI token

---

## Available Fastlane Lanes

### Android (`bundle exec fastlane android <lane>`)
| Lane | Purpose | Used in CI |
|------|---------|-----------|
| `clean` | Clean Gradle build | No |
| `build_debug` | Debug APK | Yes (build job) |
| `build_release` | Release APK | No |
| `build_aab` | Release AAB for Play Store | Via deploy_internal |
| `test` | Run Android unit tests | No (uses `npm test`) |
| `deploy_internal` | Build AAB + upload to Play Store internal | No |
| `promote_to_beta` | Promote internal to beta track | No |
| `deploy_production` | Build AAB + upload to production | No |
| `deploy_firebase` | Build debug + upload to Firebase (no tester invites) | Yes (deploy job) |

---

## Common Tasks

### Add a new Fastlane lane to CI

1. Add the lane to `fastlane/Fastfile` under the `platform :android` block
2. Update `.github/workflows/android.yml`
3. Add any new secrets via `gh secret set SECRET_NAME`
4. If new env vars are needed, document them in `fastlane/.env.example`

### Add Android tests to CI
Currently the pipeline runs only JS tests. To add Gradle tests:
```yaml
# Add after "Install dependencies" in the test job of android.yml
- name: Run Android unit tests
  run: bundle exec fastlane android test
```
Requires adding Ruby/Java setup steps to the test job.

### Add production deployment lane to CI
Create a new workflow or add a manual trigger:
```yaml
on:
  workflow_dispatch:
    inputs:
      track:
        description: 'Play Store track'
        required: true
        type: choice
        options: [internal, beta, production]
```

### Check GitHub Actions status
```bash
gh run list --limit 5
gh run view <run-id>
gh run view <run-id> --log-failed
```

### Set GitHub secrets
```bash
gh secret set FIREBASE_APP_ID --body "your-app-id"
gh secret set FIREBASE_TOKEN --body "your-token"
gh secret set ANDROID_KEYSTORE_BASE64 < encoded-keystore.txt
```

---

## Environment Variables Reference

All env vars for deployment are documented in `fastlane/.env.example`. For CI/CD, these must be set as GitHub repository secrets.

### Encoding the Android keystore for CI
```bash
base64 -i android/app/release.keystore -o keystore-base64.txt
gh secret set ANDROID_KEYSTORE_BASE64 < keystore-base64.txt
rm keystore-base64.txt
```

---

## Troubleshooting CI Failures

### "Could not find bundler" in CI
Ensure `bundler-cache: true` is set in the `ruby/setup-ruby` step and `Gemfile.lock` is committed.

### Android build fails with "Permission denied - gradlew"
Ensure `chmod +x android/gradlew` step exists before the build step in the workflow.

### Android build fails with signing errors
Verify the keystore secret is properly base64-encoded and `ANDROID_KEY_ALIAS` matches the alias in the keystore.

### Workflow not triggering
- PRs trigger test+build only
- Deploy only runs on push to `main` (not PRs, not `develop`)
- Check branch protection rules aren't blocking

---

## Related Skills

- `/setup-fastlane` — Initial Fastlane configuration
