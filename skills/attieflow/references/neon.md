# Neon selection

Use this with the shared [selection procedure](database-branching.md). Read the installed CLI's `neon --help`, `neon branches create --help`, and `neon connection-string --help` before executing; older installations may expose `neonctl` instead. Never install or upgrade solely to make an assumed flag work.

Always pass the configured project ID. Verify access by reading that project and its development branch. Resolve branches from structured JSON, following pagination if using the API. Check `project_id`, `id`, `parent_id`, state, expiry, and exact name. Do not let CLI context choose a project or parent.

Command templates, with non-secret placeholders:

```bash
neon projects get <project-id> --output json
neon branches get <development-branch-id> --project-id <project-id> --output json
neon branches list --project-id <project-id> --output json
neon branches create --project-id <project-id> --parent <development-branch-id> --name <issue-name> --cu <min-max> --suspend-timeout <seconds> --no-secrets --output json
```

For persistent branches omit `--expires-at`. For an authorized TTL policy calculate an absolute UTC expiration on first creation and pass `--expires-at <RFC3339-time>`. Re-selection must not recalculate the expiry. Neon expiration deletes the branch and its compute; do not enable it without the authorization described in the configuration contract. Use the configured CU range (or a single value for fixed size). Verify the resulting read-write endpoint and its settings. If a required option is unsupported, stop or consult the current API rather than dropping the setting.

Get the candidate connection string for the verified branch ID, with explicit role and database. **Execute this inside a helper that captures output privately; the result contains a password:**

```bash
neon connection-string <branch-id> --project-id <project-id> --database-name <database> --role-name <role> --endpoint-type read_write
```

Add `--pooled` only for a configured pooled connection. If the app also needs a direct URL, obtain and validate it separately. Confirm both endpoints belong to the selected branch; never construct a pooled hostname by string replacement or use a default connection string. Require TLS and complete the shared probe/install/reload procedure. Do not use `--psql` or print credentials into the transcript.

Sources: [branch CLI](https://neon.com/docs/cli/branches), [connection CLI](https://neon.com/docs/cli/connection-string), [branching](https://neon.com/docs/introduction/branching).
