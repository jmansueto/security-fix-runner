# Security Fix Runner

Centralized GitHub Actions runner for automated vulnerability resolution across multiple repositories.

## Overview

Security Fix Runner is a standalone GitHub Actions workflow that can target any repository (or multiple repositories simultaneously) to automatically detect, analyze, and fix CodeQL vulnerabilities. The runner intelligently batches related vulnerabilities and leverages the Devin API to create Pull Requests that resolve security issues.

### Key Features

- **Enterprise-Scale Processing**: Process multiple repositories in parallel using GitHub Actions matrix strategy
- **Automatic SARIF Detection**: Finds existing CodeQL SARIF files or generates them on-demand
- **Intelligent Batching**: Groups related vulnerabilities by file, rule, or directory for efficient fixes
- **Controlled Concurrency**: Configure parallel processing limits to manage resource usage
- **Per-Repository Artifacts**: Separate results, batches, and logs for each repository
- **Consolidated Reporting**: Aggregate statistics across all processed repositories
- **Fail-Safe Design**: One repository's failure doesn't stop processing of others

## How It Works

The workflow operates in three stages:

1. **Preparation**: Parses single or multi-repository inputs into a JSON matrix for parallel processing
2. **Resolution**: For each repository in parallel:
   - Checks out the repository and searches for existing SARIF files
   - If no SARIF exists, runs CodeQL locally to generate one
   - Parses the SARIF and batches vulnerabilities using intelligent grouping strategies
   - For each batch, creates a Devin session that analyzes the issues and creates a PR with fixes
   - Uploads per-repository results, batches, and logs as artifacts
3. **Aggregation**: Collects results from all repositories and generates a consolidated summary with aggregate statistics

## Required Secrets

Before using this runner, you must configure two secrets in the `security-fix-runner` repository settings:

### 1. DEVIN_API_KEY (required)

- **Description**: API key for the Devin API
- **How to obtain**: From your Devin account settings at https://app.devin.ai
- **Scope**: Used to create Devin sessions for vulnerability fixes
- **Note**: Without this key, only dry-run mode will work

### 2. CROSS_REPO_GH_TOKEN (required)

- **Description**: Personal Access Token (PAT) with cross-repository access
- **Required scopes**: 
  - `repo` (for checking out private repositories)
- **Optional scopes**:
  - `workflow` (only needed if you later add a "dispatch" mode to trigger target repo's CodeQL workflow)
- **How to create**:
  1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
  2. Click "Generate new token (classic)"
  3. Select the `repo` scope
  4. Generate and copy the token
- **Important**: The default `GITHUB_TOKEN` won't work for checking out external repositories

### Configuring Secrets

1. Navigate to the `security-fix-runner` repository on GitHub
2. Go to Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Add both `DEVIN_API_KEY` and `CROSS_REPO_GH_TOKEN`

## Usage Instructions

### Step 1: Configure Secrets

Ensure both required secrets (`DEVIN_API_KEY` and `CROSS_REPO_GH_TOKEN`) are configured in the repository settings as described above.

### Step 2: Run the Workflow

1. Navigate to the [Actions tab](../../actions) in the `security-fix-runner` repository
2. Select "Resolve CodeQL Vulnerabilities (Runner)" workflow
3. Click "Run workflow"
4. Fill in the inputs based on your use case (see examples below)

### Step 3: Monitor Progress

- Watch the workflow run in real-time in the Actions tab
- Each repository processes in parallel (up to `max_parallel` at once)
- View the consolidated summary for aggregate results across all repositories
- Download per-repository artifacts for detailed logs and batch information

## Input Parameters Reference

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `targets` | string | Yes | - | Target repository or repositories. Single line for one repo (e.g., `jmansueto/user-management-vulns`) or multi-line for multiple repos (one per line). Lines starting with `#` are treated as comments. |
| `target_branch` | string | No | - | Branch to target for PRs (optional, defaults to repo's default branch). Applied to all repos. |
| `sarif_path` | string | No | - | Relative path to SARIF file in target repo (optional, e.g., `codeql-results/results-123.sarif`). Applied to all repos. |
| `batch_size` | string | No | `4` | Number of vulnerabilities per batch |
| `dry_run` | boolean | No | `true` | Dry run mode (output batches without calling Devin API) |
| `max_batches` | string | No | - | Maximum number of batches to process per repo (optional, for limiting demo time) |
| `max_parallel` | string | No | `3` | Maximum number of repositories to process in parallel (controls concurrency) |
| `session_timeout` | string | No | `1800` | Maximum seconds to wait for each Devin session (default: 1800 = 30 minutes) |
| `poll_interval` | string | No | `30` | Seconds between status polls when waiting for Devin sessions |
| `codeql_languages` | string | No | `python` | Languages for CodeQL analysis (comma-separated if multiple) |

**Note**: The CodeQL query pack is hardcoded to `security-extended` for consistency across all runs.

## Example Runs

### Example 1: Single Repository Dry Run (Testing)

Test the workflow without making actual API calls:

```yaml
targets: jmansueto/user-management-vulns
dry_run: true
batch_size: 4
max_batches: 2
```

This will parse the SARIF, create batches, and show what would be processed, but won't call the Devin API.

### Example 2: Single Repository Production Run

Process a single repository and create actual PRs:

```yaml
targets: jmansueto/user-management-vulns
target_branch: main
dry_run: false
batch_size: 4
session_timeout: 1800
```

This will create actual Devin sessions and PRs for the specified repository.

### Example 3: Multi-Repository Enterprise Demo (Dry Run)

Test enterprise-scale processing with multiple repositories:

```yaml
targets: |
  jmansueto/user-management-vulns
  jmansueto/security-fix-runner
  orgname/service-api
dry_run: true
batch_size: 4
max_parallel: 3
max_batches: 2
```

This demonstrates enterprise-scale processing: 3 repositories processed in parallel, with dry run mode for testing. The workflow will process up to 3 repos simultaneously and limit each to 2 batches.

### Example 4: Multi-Repository Production Run

Process multiple repositories in production mode:

```yaml
targets: |
  jmansueto/user-management-vulns
  jmansueto/another-repo
  # orgname/disabled-repo (commented out)
target_branch: main
dry_run: false
batch_size: 4
max_parallel: 2
session_timeout: 1800
```

This processes multiple repositories in production mode, creating actual PRs. Lines starting with `#` are treated as comments and ignored. The `max_parallel: 2` setting ensures only 2 repositories are processed simultaneously.

### Example 5: Custom SARIF Path

Use a specific SARIF file from a custom location:

```yaml
targets: jmansueto/user-management-vulns
sarif_path: custom-results/analysis-results.sarif
dry_run: false
batch_size: 4
```

This will use the SARIF file at the specified path instead of auto-detecting.

## Vulnerability Batching Strategy

The runner uses a three-tier hierarchical batching strategy implemented in `vulnerability-resolver/parse_sarif.py`:

### Bucket A: Same File + Same Rule
Groups vulnerabilities that occur in the same file AND are triggered by the same CodeQL rule. This creates highly focused batches where all issues are related and can often be fixed with a single pattern.

**Example**: Multiple SQL injection vulnerabilities (same rule) in `models.py` (same file)

### Bucket B: Same Rule Across Files
Groups remaining vulnerabilities by the same CodeQL rule across different files. This handles cases where the same type of vulnerability appears in multiple locations.

**Example**: XSS vulnerabilities (same rule) in `routes/users.py`, `routes/profile.py`, and `routes/admin.py` (different files)

### Bucket C: Same Directory
Groups all remaining vulnerabilities by directory. This is a catch-all to ensure complete coverage while maintaining some logical grouping.

**Example**: Various different vulnerabilities in the `routes/` directory

Each batch is limited to the configured `batch_size` (default: 4 vulnerabilities), ensuring manageable PR scope.

## Architecture Notes

### Enterprise-Scale Design

The workflow uses GitHub Actions matrix strategy to process multiple repositories in parallel, demonstrating enterprise capability:

- **Three-Job Architecture**: Separate jobs for preparation, resolution, and aggregation
- **Matrix Strategy**: Dynamic matrix generation from repository list enables parallel processing
- **Controlled Concurrency**: `max_parallel` input limits simultaneous processing to manage resource usage
- **Fail-Safe Processing**: `fail-fast: false` ensures one repo's failure doesn't stop others

### Per-Repository Artifacts

Each repository gets separate artifacts with sanitized names for clear tracking:

- `results-{owner-repo}` - Processing results with status, PR URLs, and metrics
- `batches-{owner-repo}` - Vulnerability batch metadata with grouping details
- `logs-{owner-repo}` - Detailed execution logs for debugging

### Consolidated Reporting

The aggregate job creates an enterprise-wide summary showing:

- Per-repository table with vulnerabilities, batches, success/failure counts, and status
- Aggregate totals across all repositories
- Overall success rate
- Links to download per-repository artifacts

### Technical Details

- The runner uses the existing `vulnerability-resolver/` Python code without modifications
- SARIF files are detected automatically or generated via local CodeQL runs
- The runner never commits SARIF files back to target repositories
- All results are provided as downloadable artifacts
- CodeQL runs locally in the runner's environment with `source-root: target` to scan checked-out repos
- `upload: false` prevents the runner from trying to upload security events to target repos

## Troubleshooting

### "Failed to checkout target repository"

**Cause**: The `CROSS_REPO_GH_TOKEN` secret is missing or doesn't have the required permissions.

**Solution**: 
1. Verify the secret is configured in repository settings
2. Ensure the PAT has the `repo` scope
3. Check that the PAT hasn't expired
4. For private repositories, ensure the PAT owner has access to the target repo

### "No SARIF file found"

**Cause**: The runner couldn't find an existing SARIF file in the expected locations.

**Solution**: This is usually not an error - the runner will automatically run CodeQL to generate a SARIF file. If you want to use an existing SARIF file, specify the `sarif_path` input parameter.

### "Devin API error" or "Authentication failed"

**Cause**: The `DEVIN_API_KEY` secret is missing, invalid, or expired.

**Solution**:
1. Verify the secret is configured in repository settings
2. Check that the API key is valid and hasn't expired
3. Ensure the API key has the necessary permissions
4. Try regenerating the API key from your Devin account settings

### "Session timeout"

**Cause**: A Devin session took longer than the configured `session_timeout` to complete.

**Solution**:
1. Increase the `session_timeout` input (default is 1800 seconds / 30 minutes)
2. Consider reducing `batch_size` to create smaller, faster-to-process batches
3. Check the logs artifact to see what the session was working on
4. For complex fixes, try `session_timeout: 3600` (1 hour)

### "CodeQL analysis failed"

**Cause**: CodeQL couldn't analyze the target repository.

**Solution**:
1. Verify the `codeql_languages` input matches the repository's languages
2. Check that the repository contains code in the specified language
3. Review the workflow logs for specific CodeQL error messages
4. The workflow uses the `security-extended` query pack by default, which should work for most repositories

### Workflow fails on some repos but not others

**Cause**: This is expected behavior with `fail-fast: false` - the workflow continues processing other repos even if one fails.

**Solution**:
1. Check the aggregate summary to see which repos succeeded/failed
2. Download the logs artifact for failed repos to diagnose issues
3. Review per-repository summaries in the workflow output
4. Failed repos can be reprocessed individually by running the workflow again with just that repo

## Future Enhancements

Optional features that could be added in future versions:

- **Dispatch Mode**: Support for triggering the target repo's CodeQL workflow via `workflow_dispatch` instead of running CodeQL locally
- **Multi-Language Support**: Enhanced handling for repositories with multiple languages
- **Webhook Integration**: Automatic triggering when new CodeQL results are available
- **Custom Batching Strategies**: Configurable batching logic beyond the three-tier system
- **SARIF Source Flexibility**: Support for other SARIF sources beyond CodeQL (e.g., Semgrep, Snyk)
- **PR Review Automation**: Automatic approval of PRs that pass all checks
- **Slack/Email Notifications**: Real-time notifications of processing status and results

## Contributing

This is a centralized runner designed to work with any repository. To use it with your repositories:

1. Ensure the required secrets are configured in this repository
2. Run the workflow via manual dispatch with your target repository
3. Monitor the results and review the generated PRs

For issues or feature requests, please contact the repository maintainer.

## License

This project is provided as-is for automated vulnerability remediation purposes.
