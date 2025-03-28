Below is a comprehensive document that outlines the entire process of setting up a GitHub Actions workflow for a Flutter project to automatically increment the version in `pubspec.yaml` using Semantic Versioning (SemVer) based on Conventional Commits. The document also includes the fallback for non-conventional commits, the setup for using the latest Flutter version, and the configuration of a commit message generator for GitHub Copilot to ensure commit messages align with the workflow.

---

# Automating Version Increment in a Flutter Project with GitHub Actions

This document provides a step-by-step guide to automate version incrementing in a Flutter project using GitHub Actions. The workflow will increment the version in `pubspec.yaml` based on Semantic Versioning (SemVer) rules, using Conventional Commits to determine the type of version bump. It also includes a fallback for non-conventional commits, ensures the latest Flutter version is used, and configures GitHub Copilot to generate appropriate commit messages.

---

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Setting Up the GitHub Actions Workflow](#setting-up-the-github-actions-workflow)
   - [Workflow File](#workflow-file)
   - [Explanation of the Workflow](#explanation-of-the-workflow)
   - [Semantic Versioning Logic](#semantic-versioning-logic)
   - [Fallback for Non-Conventional Commits](#fallback-for-non-conventional-commits)
   - [Using the Latest Flutter Version](#using-the-latest-flutter-version)
4. [Configuring GitHub Copilot for Commit Message Generation](#configuring-github-copilot-for-commit-message-generation)
   - [Copilot Configuration](#copilot-configuration)
   - [Alternative: Store Instructions in a File](#alternative-store-instructions-in-a-file)
5. [Testing the Workflow](#testing-the-workflow)
   - [Test Scenarios](#test-scenarios)
6. [Troubleshooting](#troubleshooting)
   - [Permissions Issues](#permissions-issues)
   - [Commit Message Parsing Issues](#commit-message-parsing-issues)
7. [Optional Enhancements](#optional-enhancements)
8. [Conclusion](#conclusion)

---

## Overview

The goal of this setup is to automate version management in a Flutter project by:
- Incrementing the version in `pubspec.yaml` (e.g., `1.0.0+1`) based on Semantic Versioning rules.
- Using Conventional Commits to determine the type of version bump:
  - `feat:` → MINOR bump (e.g., `1.0.0` → `1.1.0`).
  - `fix:` → PATCH bump (e.g., `1.0.0` → `1.0.1`).
  - `BREAKING CHANGE` → MAJOR bump (e.g., `1.0.0` → `2.0.0`).
- Incrementing the build number (e.g., `+1` → `+2`) with each version bump.
- Skipping version bumps for non-conventional commits (e.g., `docs:`, `chore:`).
- Using the latest stable Flutter version in the GitHub Actions environment.
- Configuring GitHub Copilot to generate commit messages that align with the Conventional Commits format.

The workflow runs on every push to the `main` branch, updates `pubspec.yaml`, and commits the change back to the repository.

---

## Prerequisites

1. **Flutter Project**:
   - A Flutter project with a `pubspec.yaml` file containing a `version` field (e.g., `version: 1.0.0+1`).
2. **GitHub Repository**:
   - A GitHub repository with your Flutter project.
3. **GitHub Actions Enabled**:
   - Ensure GitHub Actions is enabled for your repository (it is by default for public repositories).
4. **Optional: GitHub Copilot**:
   - If you want to use Copilot for commit message generation, you’ll need a GitHub Copilot subscription and the Copilot extension installed in your IDE (e.g., VS Code).

---

## Setting Up the GitHub Actions Workflow

### Workflow File

Create a file named `.github/workflows/version-increment.yml` in your repository with the following content:

```yaml
name: Increment Version

permissions:
  contents: write

on:
  push:
    branches:
      - main

jobs:
  version-increment:
    runs-on: ubuntu-latest

    steps:
      # Checkout the repository code
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Fetch full history to analyze commits

      # Set up Flutter environment (using latest stable version)
      - name: Setup Flutter
        uses: subosito/flutter-action@v2
        with:
          channel: 'stable' # Automatically uses the latest stable version

      # Analyze commits and increment version
      - name: Increment Version with Semantic Versioning
        id: version_increment
        run: |
          # Get the latest commit message
          COMMIT_MESSAGE=$(git log -1 --pretty=%B)
          echo "Commit message: $COMMIT_MESSAGE"

          # Check if the commit message follows Conventional Commits
          if ! echo "$COMMIT_MESSAGE" | grep -qiE "^(feat|fix):|^BREAKING CHANGE"; then
            echo "No version bump required for this commit (non-conventional commit)."
            echo "SKIPPED=true" >> $GITHUB_OUTPUT
            exit 0
          fi

          # Get current version from pubspec.yaml
          CURRENT_VERSION=$(grep 'version:' pubspec.yaml | sed 's/version: //')
          VERSION_NAME=$(echo $CURRENT_VERSION | cut -d'+' -f1)
          BUILD_NUMBER=$(echo $CURRENT_VERSION | cut -d'+' -f2)

          # Parse current version into MAJOR, MINOR, PATCH
          MAJOR=$(echo $VERSION_NAME | cut -d'.' -f1)
          MINOR=$(echo $VERSION_NAME | cut -d'.' -f2)
          PATCH=$(echo $VERSION_NAME | cut -d'.' -f3)

          # Default to PATCH increment
          BUMP_TYPE="patch"

          # Check for breaking changes (MAJOR bump)
          if echo "$COMMIT_MESSAGE" | grep -qi "BREAKING CHANGE"; then
            BUMP_TYPE="major"
          # Check for features (MINOR bump)
          elif echo "$COMMIT_MESSAGE" | grep -qi "^feat"; then
            BUMP_TYPE="minor"
          # Otherwise, default to PATCH bump (e.g., for "fix:" commits)
          fi

          # Increment version based on BUMP_TYPE
          case $BUMP_TYPE in
            "major")
              MAJOR=$((MAJOR + 1))
              MINOR=0
              PATCH=0
              ;;
            "minor")
              MINOR=$((MINOR + 1))
              PATCH=0
              ;;
            "patch")
              PATCH=$((PATCH + 1))
              ;;
          esac

          # Increment build number
          if [ -z "$BUILD_NUMBER" ]; then
            NEW_BUILD_NUMBER=1
          else
            NEW_BUILD_NUMBER=$((BUILD_NUMBER + 1))
          fi

          # Construct new version
          NEW_VERSION_NAME="$MAJOR.$MINOR.$PATCH"
          NEW_VERSION="$NEW_VERSION_NAME+$NEW_BUILD_NUMBER"

          # Update pubspec.yaml with new version
          sed -i "s/version: $CURRENT_VERSION/version: $NEW_VERSION/" pubspec.yaml

          # Display the new version
          echo "Updated version to: $NEW_VERSION"
          echo "NEW_VERSION=$NEW_VERSION" >> $GITHUB_OUTPUT
          echo "SKIPPED=false" >> $GITHUB_OUTPUT

      # Commit and push the updated pubspec.yaml (only if not skipped)
      - name: Commit version update
        if: ${{ steps.version_increment.outputs.SKIPPED != 'true' }}
        run: |
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add pubspec.yaml
          git commit -m "Bump version to ${{ steps.version_increment.outputs.NEW_VERSION }} [skip ci]"
          git push
```

### Explanation of the Workflow

1. **Trigger**:
   - The workflow runs on every `push` to the `main` branch (`on: push: branches: - main`).
   - You can adjust the branch name (e.g., `develop`) if needed.

2. **Permissions**:
   - `permissions: contents: write` grants the workflow write access to the repository, allowing it to push changes to `pubspec.yaml`.

3. **Steps**:
   - **Checkout Code**:
     - Uses `actions/checkout@v4` to fetch the repository code with full history (`fetch-depth: 0`) to analyze commit messages.
   - **Setup Flutter**:
     - Uses `subosito/flutter-action@v2` to set up the Flutter environment.
     - The `channel: 'stable'` ensures the latest stable Flutter version is used (more on this below).
   - **Increment Version with Semantic Versioning**:
     - Analyzes the latest commit message to determine the type of version bump.
     - Updates the version in `pubspec.yaml` based on SemVer rules.
     - Sets environment variables (`NEW_VERSION`, `SKIPPED`) for use in later steps.
   - **Commit Version Update**:
     - Commits the updated `pubspec.yaml` and pushes it to the repository.
     - Only runs if the version bump wasn’t skipped (`if: env.SKIPPED != 'true'`).
     - Includes `[skip ci]` in the commit message to prevent an infinite loop.

### Semantic Versioning Logic

The workflow uses **Conventional Commits** to determine the type of version bump:
- **MAJOR Bump**:
  - Triggered if the commit message contains `BREAKING CHANGE` (e.g., `feat: update API\n\nBREAKING CHANGE: remove old API`).
  - Increments the MAJOR version, resets MINOR and PATCH to 0 (e.g., `1.2.3` → `2.0.0`).
- **MINOR Bump**:
  - Triggered if the commit message starts with `feat:` (e.g., `feat: add login screen`).
  - Increments the MINOR version, resets PATCH to 0 (e.g., `1.2.3` → `1.3.0`).
- **PATCH Bump**:
  - Triggered if the commit message starts with `fix:` (e.g., `fix: resolve crash`).
  - Increments the PATCH version (e.g., `1.2.3` → `1.2.4`).
- **Build Number**:
  - The build number (e.g., `+3` in `1.2.3+3`) increments with each version bump (e.g., `+3` → `+4`).

### Fallback for Non-Conventional Commits

The workflow skips version bumps for commits that don’t follow the Conventional Commits format:
- A commit message is considered non-conventional if it doesn’t start with `feat:`, `fix:`, or contain `BREAKING CHANGE`.
- Examples of non-conventional commits: `Update README`, `chore: update deps`, `docs: add setup guide`.
- If a commit is non-conventional:
  - The workflow logs: `No version bump required for this commit (non-conventional commit).`
  - Sets `SKIPPED=true` to skip the commit step.
  - `pubspec.yaml` remains unchanged.

### Using the Latest Flutter Version

The `Setup Flutter` step uses the latest stable Flutter version by omitting the `flutter-version` field:
- `subosito/flutter-action@v2` with `channel: 'stable'` automatically fetches the latest stable version of Flutter.
- This ensures the workflow always uses the most recent Flutter version without manual updates.

If you want to explicitly fetch the latest version, you can add a step to query Flutter’s release data (see the [Optional Enhancements](#optional-enhancements) section).

---

## Configuring GitHub Copilot for Commit Message Generation

To ensure commit messages align with the workflow’s requirements, we’ll configure GitHub Copilot to generate commit messages following the Conventional Commits format.

### Copilot Configuration

Add the following configuration to your IDE’s settings (e.g., `settings.json` in VS Code):

```json
{
  "github.copilot.chat.commitMessageGeneration.instructions": [
    {
      "text": "Generate commit messages following the Conventional Commits specification to support Semantic Versioning."
    },
    {
      "text": "Use the prefix 'feat:' for new features or enhancements, which will trigger a MINOR version bump."
    },
    {
      "text": "Use the prefix 'fix:' for bug fixes, which will trigger a PATCH version bump."
    },
    {
      "text": "For breaking changes, include a 'BREAKING CHANGE:' footer in the commit message body to trigger a MAJOR version bump. Example:\nfeat: update API endpoints\n\nBREAKING CHANGE: remove old auth method"
    },
    {
      "text": "For non-version-bumping changes (e.g., documentation, chores, or refactors), use prefixes like 'docs:', 'chore:', or 'refactor:'. These commits will not trigger a version bump."
    },
    {
      "text": "Keep commit messages concise, starting with the prefix, followed by a short description (50 characters or less). Add a detailed body if needed, separated by a blank line."
    },
    {
      "text": "Examples:\n- 'feat: add user profile screen'\n- 'fix: resolve null pointer exception'\n- 'feat: update API endpoints\n\nBREAKING CHANGE: remove old auth method'\n- 'docs: update README with setup instructions'\n- 'chore: update dependencies'"
    }
  ]
}
```

### Alternative: Store Instructions in a File

If you prefer to store the instructions in a file, create `commit-message-style.md` in your repository:

```markdown
# Commit Message Style Guide

- Generate commit messages following the Conventional Commits specification to support Semantic Versioning.
- Use the prefix `feat:` for new features or enhancements, which will trigger a MINOR version bump.
- Use the prefix `fix:` for bug fixes, which will trigger a PATCH version bump.
- For breaking changes, include a `BREAKING CHANGE:` footer in the commit message body to trigger a MAJOR version bump. Example:
  ```
  feat: update API endpoints

  BREAKING CHANGE: remove old auth method
  ```
- For non-version-bumping changes (e.g., documentation, chores, or refactors), use prefixes like `docs:`, `chore:`, or `refactor:`. These commits will not trigger a version bump.
- Keep commit messages concise, starting with the prefix, followed by a short description (50 characters or less). Add a detailed body if needed, separated by a blank line.
- Examples:
  - `feat: add user profile screen`
  - `fix: resolve null pointer exception`
  - `feat: update API endpoints\n\nBREAKING CHANGE: remove old auth method`
  - `docs: update README with setup instructions`
  - `chore: update dependencies`
```

Then, update your Copilot configuration to reference the file:

```json
{
  "github.copilot.chat.commitMessageGeneration.instructions": [
    {
      "file": "commit-message-style.md"
    }
  ]
}
```

---

## Testing the Workflow

### Test Scenarios

1. **Initial Setup**:
   - Ensure your `pubspec.yaml` has a version field, e.g.:
     ```yaml
     version: 1.0.0+1
     ```
   - Commit and push the workflow file (`.github/workflows/version-increment.yml`) to your repository.

2. **Non-Conventional Commit**:
   - Commit: `git commit -m "Update README"`
   - Push to `main`.
   - Expected Result:
     - Workflow logs: `No version bump required for this commit (non-conventional commit).`
     - `pubspec.yaml` remains unchanged: `version: 1.0.0+1`.

3. **Bug Fix (PATCH Bump)**:
   - Commit: `git commit -m "fix: resolve null pointer exception"`
   - Push to `main`.
   - Expected Result:
     - Workflow updates `pubspec.yaml` to `version: 1.0.1+2`.
     - Logs show: `Updated version to: 1.0.1+2`.

4. **New Feature (MINOR Bump)**:
   - Commit: `git commit -m "feat: add user profile screen"`
   - Push to `main`.
   - Expected Result:
     - Workflow updates `pubspec.yaml` to `version: 1.1.0+3`.
     - Logs show: `Updated version to: 1.1.0+3`.

5. **Breaking Change (MAJOR Bump)**:
   - Commit:
     ```
     git commit -m "feat: update API endpoints

     BREAKING CHANGE: remove old auth method"
     ```
   - Push to `main`.
   - Expected Result:
     - Workflow updates `pubspec.yaml` to `version: 2.0.0+4`.
     - Logs show: `Updated version to: 2.0.0+4`.

6. **Test Copilot Commit Message Generation**:
   - Make a change (e.g., add a feature).
   - Use Copilot to generate a commit message (e.g., in VS Code).
   - Expected Result: Copilot suggests a message like `feat: add login screen`.
   - Commit and push, then verify the workflow updates the version accordingly.

---

## Troubleshooting

### Permissions Issues

If the workflow fails with a `403` error (e.g., `remote: Permission to refs/heads/main denied to github-actions[bot]`):
- Ensure `permissions: contents: write` is set in the workflow file.
- Check branch protection rules in your repository settings:
  - Go to **Settings > Branches**.
  - If `main` has protection rules (e.g., "Require a pull request before merging"), either:
    - Allow `github-actions[bot]` to push to `main`.
    - Or modify the workflow to create a pull request instead (see [Optional Enhancements](#optional-enhancements)).

### Commit Message Parsing Issues

If the workflow doesn’t correctly detect the commit message type:
- Verify the commit message format:
  - Use `feat:` or `fix:` at the start of the message.
  - For breaking changes, include `BREAKING CHANGE:` in the body (not the subject).
- Add debug logging to the workflow:
  ```bash
  echo "Commit message matched: $(echo "$COMMIT_MESSAGE" | grep -qiE "^(feat|fix):|^BREAKING CHANGE" && echo 'yes' || echo 'no')"
  ```
  Add this before the `if` condition in the `Increment Version with Semantic Versioning` step.

---

## Optional Enhancements

1. **Explicitly Fetch the Latest Flutter Version**:
   - Add a step to fetch the latest Flutter version dynamically:
     ```yaml
     - name: Get Latest Flutter Version
       id: get-flutter-version
       run: |
         LATEST_VERSION=$(curl -s https://storage.googleapis.com/flutter_infra_release/releases/releases_linux.json | \
           jq -r '.releases[] | select(.channel=="stable") | .version' | head -n 1)
         echo "Latest stable Flutter version: $LATEST_VERSION"
         echo "FLUTTER_VERSION=$LATEST_VERSION" >> $GITHUB_ENV

     - name: Setup Flutter
       uses: subosito/flutter-action@v2
       with:
         flutter-version: ${{ env.FLUTTER_VERSION }}
         channel: 'stable'
     ```

2. **Create a Pull Request Instead of Direct Push**:
   - If branch protection rules prevent direct pushes to `main`, modify the workflow to create a pull request:
     ```yaml
     - name: Create Pull Request
       uses: peter-evans/create-pull-request@v5
       with:
         commit-message: "Bump version to ${{ env.NEW_VERSION }} [skip ci]"
         branch: "version-update/${{ env.NEW_VERSION }}"
         title: "Bump version to ${{ env.NEW_VERSION }}"
         body: "This PR increments the version to ${{ env.NEW_VERSION }}."
         delete-branch: true
     ```
   - Add `permissions: pull-requests: write` to the workflow.

3. **Enforce Conventional Commits**:
   - Add a step to lint commit messages using `commitlint`:
     ```yaml
     - name: Lint Commit Messages
       uses: wagoid/commitlint-github-action@v5
       with:
         configFile: .commitlintrc.json
     ```
   - Create a `.commitlintrc.json` file with your Conventional Commits rules.

4. **More Specific Commit Message Patterns**:
   - Extend the `grep` pattern to include additional Conventional Commit types (e.g., `perf:` for PATCH bumps):
     ```bash
     if ! echo "$COMMIT_MESSAGE" | grep -qiE "^(feat|fix|perf):|^BREAKING CHANGE"; then
     ```

---

## Conclusion

This setup automates version management in your Flutter project by:
- Incrementing the version in `pubspec.yaml` based on Semantic Versioning and Conventional Commits.
- Skipping version bumps for non-conventional commits.
- Using the latest stable Flutter version in the GitHub Actions environment.
- Configuring GitHub Copilot to generate commit messages that align with the workflow’s requirements.

By following this process, you can streamline your development workflow, ensure consistent versioning, and reduce manual effort. If you encounter issues or need further customization, refer to the [Troubleshooting](#troubleshooting) and [Optional Enhancements](#optional-enhancements) sections.

--- 