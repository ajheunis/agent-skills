# Database branching

Read this only when `.attieflow/database.json` exists. These are instructions for the agent executing attieflow; there is no installed `attieflow` shell binary or Git hook. Use the configured provider CLI/API and the application's existing local configuration loader.

## Repository configuration

Store non-secret settings in `.attieflow/database.json`. Inspect the local application's connection loading and restart mechanism before filling this in; do not assume editing an environment file refreshes a running process. Example for Neon (replace illustrative IDs and application commands):

```json
{
  "version": 1,
  "provider": "neon",
  "project": "example-project-id",
  "development_branch": "br-example-development",
  "production_branches": ["br-example-production"],
  "issue_prefix": "myapp-issue-",
  "database": "app",
  "role": "app_developer",
  "lifetime": {"mode": "persistent"},
  "compute": {"min_cu": 0.5, "max_cu": 1, "suspend_seconds": 300},
  "connection": {"pooled": false},
  "local": {
    "file": ".env.local",
    "keys": ["DATABASE_URL"],
    "stop": ["npm", "run", "local:stop"],
    "reload": ["npm", "run", "local:restart"],
    "verify": ["npm", "run", "local:db-check"]
  }
}
```

For Lakebase use `provider: "lakebase"`, add `profile` and `workspace_host`, and use full resource names for `project` (`projects/app`) and all branches (`projects/app/branches/development`). Choose an existing Postgres database and OAuth role; the Databricks API identity is not automatically the application's database role. Use `connection.pooled: false` unless the repository has an explicitly verified Lakebase pooler integration.

Contract:

- Version 1 supports exactly `neon` and `lakebase`. All example fields are required; Lakebase additionally requires the profile and workspace host. Reject unknown fields or unsupported combinations rather than guessing their meaning.
- `development_branch` is an immutable provider ID/resource name, never a name resolved through a CLI default. It must exist and differ from every `production_branches` entry. Record all production branch IDs in the project; do not derive development from a provider's default branch, which may be production. Never create the shared base as part of issue start.
- `issue_prefix` is stable and unique to this GitHub repository within the project. Append the positive issue number, not a mutable issue title or Git branch slug. For both providers use lowercase letters, digits, and hyphens, start with a letter, and keep the resulting name within 63 characters. Verify the GitHub repository identity before reusing the prefix in a fork.
- `lifetime` is either `{"mode":"persistent"}` or `{"mode":"ttl","seconds":604800}` with a positive duration supported by the provider. Persistent is the safe initial choice. A TTL can archive or permanently delete data; require explicit authorization for automatic expiry before adopting a TTL policy, and record that decision in repository guidance. Never silently shorten or extend an existing branch's lifetime on selection.
- `compute` specifies positive minimum/maximum CU, with minimum no greater than maximum, and a supported positive idle suspension duration. Validate against the current provider and plan. Apply settings to newly created issue compute only. Do not resize shared development, production, or existing issue compute during selection; report a mismatch for a separate decision.
- `local.file` must resolve inside this checkout, be untracked and ignored (`git ls-files --error-unmatch -- <file>` must not succeed; `git check-ignore --quiet -- <file>` must succeed). Reject symlinks/reparse points and paths into deployment, production, or shared configuration. Verify that the local application actually loads this file. Never use `.env.production`, deployment secrets, or global environment settings.
- `local.keys` is the explicit allowlist of local connection variables to replace together. Include direct and pooled URLs if the application consumes both. Preserve other settings. Reject duplicate or ambiguous definitions. For applications using separate PG fields, list those keys instead and derive each value from the verified endpoint.
- `stop`, `reload`, and `verify` are argument arrays for existing, inspected, local-only application commands. Run without a shell, from the repository root, and never interpolate credentials into arguments. Stop must quiesce local workers as well as the web process. Reload must discard old pools and load the new settings. Verify must open a connection through the application's actual pool and check its configured endpoint, database, and role. If suitable commands do not exist, implement these small repository-specific hooks before enabling switching; do not claim a completed switch from a file edit alone.

No credentials belong in this JSON, issue comments, PRs, state reports, or Git. Use the provider's authenticated profile/environment for API access. Read only [Neon](neon.md) or [Lakebase](lakebase.md) as selected by configuration.

## Repeatable selection procedure

`attieflow db issue <issue-number>` and `attieflow db development` follow the same validation and installation steps. With no database configuration, the issue selector reports Git-only mode and does nothing; the development command performs the ordinary clean-tree checkout to the default Git branch without provider calls.

1. **Resolve intent and Git state.** For an issue, verify the issue belongs to this GitHub repository and the current Git branch is its linked development branch (query `gh issue develop <number> --list`; do not assume a number in the branch name proves linkage). The selector does not switch from another Git branch. Normal Phase 1 already creates/checks out the branch. For development, require a clean tree before checkout; ask what to do with local changes and never stash or discard automatically. Stop the local application before changing Git. Use the resolved default Git branch, not a hardcoded `main`, and never reset a diverged branch.
2. **Validate configuration and identity.** Authenticate with the specified provider context, fetch the exact project and development branch, and confirm workspace/project identity and branch readiness. Validate production exclusions and local paths before creating anything. Failure or insufficient permission is not evidence that a branch is absent. Leave local connection settings unchanged.
3. **Select or create.** First validate and read the prior-selection state described below; apply its missing/replaced-branch checks before any creation. Development selection uses the configured shared branch directly. For an issue, list all pages of branches in the exact project and find an exact match for prefix plus issue number. Require a unique match and check its actual parent equals `development_branch`, its project matches, and it is not a production branch. Reuse the existing branch without reset, migrations, or expiry changes. Only a verified first-time absence permits creation from the explicit development parent. Re-fetch and validate after creation; on a create conflict re-read and validate instead of retrying blindly.
4. **Handle prior selections.** Maintain ignored, non-secret `.attieflow/database-state.local.json`, keyed by provider context, project, repository, and issue number, with branch ID, parent, and last successful selection. Verify this state file is untracked, ignored, inside the checkout, and not a symlink, just like the connection file. It is a hint, never proof of remote identity. If a previously recorded branch is missing, archived, expired, or replaced by a new ID, stop and explain that its data may be unavailable. Ask whether to restore/reactivate it or create a fresh branch from development; never silently substitute fresh data. Without a record, disclose that absence is treated as first use. If creation succeeds but a later step fails, retain a non-secret record of that branch so a retry can find it, without changing the active selection. Do not delete it as rollback.
5. **Prepare privately.** Obtain credentials for the selected read-write endpoint. Capture provider stdout/stderr in a subprocess or SDK inside the local helper, never through a tool that displays raw output. Neon create responses and credential commands can contain passwords. Disable debug logging and shell tracing; sanitize errors before returning them. Keep credentials in memory or a restricted, ignored temporary file beside the destination. Use a serializer appropriate to the application's dotenv/JSON format; do not source dotenv files as shell code. On Windows restrict ACLs to the current user; on POSIX use mode 0600. Require TLS, check endpoint ownership using provider metadata, and ensure every candidate URL points to that selected branch. Never fall back to a production URL or the current environment.
6. **Probe before writing.** Using the candidate credentials, connect with the application's driver or Postgres client and a bounded timeout. Run a read-only `SELECT current_database(), current_user` and compare to the configured database/role. Also validate the hostname/endpoint against the provider response; a successful SQL query alone does not identify a branch. For pooled and direct URLs, probe both. Allow bounded readiness retries for new or sleeping compute (for example, at most two minutes); authentication, parent, identity, and persistent connection errors abort with the existing connection file byte-for-byte unchanged. Do not run migrations to make the check pass.
7. **Install and reload.** Quiesce the local app if not already stopped. Recheck the target path, Git branch, and original file contents to detect concurrent edits. Prepare one complete replacement that preserves unrelated keys, and atomically replace the connection file on the same filesystem. Keep the old contents privately in memory for rollback. Run the configured reload and verify commands with secrets available only through the local settings/process environment. Verification must use a newly opened application connection, not an old pool or just an HTTP liveness response. Update active-selection state only after success.
8. **Recover or report.** If installation, reload, verification, or state persistence fails, restore the original connection file atomically (or remove only the newly created file if none existed) and restore the previous active-selection record. Leave the application stopped when Git now targets a different branch or the old database is unavailable; otherwise reload and verify the restored configuration. Report partial Git/provider progress and a sanitized failure reason. Never report success while the app still uses the previous pool. On success report only Git branch, provider/project, database branch ID, lifetime, and pool verification status. Remove temporary secret files in all paths.

During Phase 5, stop the app before Git checkout and run this procedure for development after Git cleanup. Do not run a second checkout/cleanup if Phase 5 already did it. If the database step fails, the Git cleanup remains completed; clearly report the mismatch and keep the app stopped until `attieflow db development` succeeds.

## Resets, deletion, and schema promotion

Issue selection and Git cleanup never reset or delete a database branch. Require explicit authorization naming the provider, project, and exact branch before either operation, even after a PR merge. First switch away from that branch, check children/dependencies, and explain data loss. Do not remove branch protections or cascade deletion automatically. Production resources are outside this local switching workflow.

Git merge, database reset, and schema promotion are separate operations. Test versioned migrations on the issue branch, then use the repository's reviewed migration/deployment process to promote schema changes. Neither provider merges issue data into its parent when Git merges. Shared development can evolve after an issue branch was created; reusing that branch preserves its data rather than resetting it to catch up.

## Acceptance walkthroughs

When validating changes to this skill or adopting it in a repository, exercise these scenarios with mocked provider responses or an explicitly authorized disposable project. Do not use production for tests.

| Scenario | Expected result |
| --- | --- |
| No configuration | Original issue linkage, Git workflow, merge gate, and cleanup; no provider calls |
| Invalid provider or missing required field | Explicit configuration error; no connection changes |
| First issue start, each provider | Child of configured development base, configured lifetime/compute, successful probe and new application pool |
| Repeat issue selection | Same verified branch ID and existing data; no reset or duplicate creation |
| Wrong project, workspace, parent, or production ID | Refuse before local connection changes |
| Authentication failure or timed-out connection | Existing connection file unchanged; sanitized error |
| Recorded branch missing/expired, name now maps to another ID | Explain and request recovery choice; no silent recreation |
| Return to non-`main` default branch | Git checkout succeeds, shared development selected, pool reloaded |
| Reload/verification failure | Restore old settings; app stopped if Git/database would mismatch |
| Tracked file, symlink, duplicate keys, concurrent edit | Refuse replacement and preserve existing file |
| Merge and Git cleanup | Development connection restored; issue database retained |
| Credentials in provider responses or errors | Captured privately; no secret in transcripts, diffs, or status output |
