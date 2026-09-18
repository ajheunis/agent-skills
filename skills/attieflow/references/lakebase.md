# Databricks Lakebase selection

Use this with the shared [selection procedure](database-branching.md). This supports **Lakebase Autoscaling projects** through `databricks postgres`. A provisioned `databricks database` instance is not an interchangeable branching target; report it as unsupported rather than migrating it.

Read `databricks postgres --help` and each required subcommand's help. Pass `--profile <profile>` on every call and confirm the profile resolves to the configured workspace host. Verify authenticated identity and access to the exact project and development branch. Capture structured responses and check full resource names, `spec.source_branch`, state, and expiry. Never choose resources solely by display name.

```bash
databricks current-user me --profile <profile> --output json
databricks postgres get-project <project-resource> --profile <profile> --output json
databricks postgres get-branch <development-branch-resource> --profile <profile> --output json
databricks postgres list-branches <project-resource> --profile <profile> --output json
databricks postgres create-branch <project-resource> <issue-name> --json @<branch-spec-file> --profile <profile> --output json
```

Create the non-secret branch request with a JSON serializer. Set `spec.source_branch` to the full configured development resource and specify exactly one expiration policy: `spec.no_expiry: true` or `spec.ttl: "<seconds>s"` for an authorized, supported TTL. Validate supported duration limits with the current API. Wait for the operation to complete with a bounded timeout; re-fetch the branch before obtaining a connection. An operation timeout is not proof creation failed: read the resource/operation before retrying.

List endpoints under the selected branch. Do not assume creation always supplies an endpoint or that its ID is `primary`. Select its unique read-write endpoint; if creation did not supply one, create one within that new issue branch. Use `spec.endpoint_type: "ENDPOINT_TYPE_READ_WRITE"`, configured `autoscaling_limit_min_cu`/`autoscaling_limit_max_cu`, and the current API's scale-to-zero fields. Discover and validate those fields from installed help/current API before applying the configured idle duration. If an automatically supplied endpoint needs configuration, update only that newly created endpoint with an explicit field mask. Verify actual settings; never resize shared development or a reused issue endpoint as a side effect.

```bash
databricks postgres list-endpoints <selected-branch-resource> --profile <profile> --output json
databricks postgres get-endpoint <selected-endpoint-resource> --profile <profile> --output json
```

Obtain the hostname from the verified endpoint response and check its resource ancestry. Verify the database and OAuth role exist on that branch. Do not grant privileges or create roles just to bypass a failed probe.

Generate credentials **inside a helper that captures output privately**:

```bash
databricks postgres generate-database-credential <selected-endpoint-resource> --profile <profile> --output json
```

Build the candidate Postgres connection with a URL/connection-parameter library, TLS, the selected hostname, configured database, and configured role. Never print the token or assemble unescaped credentials in a shell. OAuth credentials expire; use the application's token refresh mechanism before new pool connections, or arrange a fresh selection/reload before expiration. A stored token alone is not a durable authentication setup. Verify refreshed credentials remain scoped to the selected endpoint and complete the shared probe/install/reload procedure.

Sources: [Lakebase CLI guide](https://docs.databricks.com/aws/en/oltp/projects/cli), [postgres command reference](https://docs.databricks.com/aws/en/dev-tools/cli/reference/postgres-commands), [branch semantics](https://docs.databricks.com/aws/en/oltp/projects/branches).
