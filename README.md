# github-actions

Shared GitHub Actions components for infrastructure automation.

| Component | Purpose |
| --- | --- |
| `.github/workflows/terraform-quality.yml` | Reusable Terraform formatting, linting, validation, and security checks |
| `actions/terraform/action.yml` | Composite action for Terraform initialization, planning, and optional application |

## Terraform execution action

The action runs five separately named steps:

1. Validate inputs
2. Terraform Init
3. Terraform Plan
4. Terraform Apply, when requested
5. Remove saved plan

Separate steps expose each operation's status, duration, and logs. A failure
in initialization is distinguishable from a planning or application failure.

### Caller responsibilities

Before invoking the action, the calling job must:

- Check out the repository containing the Terraform root.
- Install a pinned Terraform version compatible with that root.
- Provide backend and provider authentication.
- Supply required Terraform variables, for example through `TF_VAR_*`.
- Commit a compatible `.terraform.lock.hcl` with checksums suitable for
  the runner platform.
- Select the intended Terraform workspace if using multiple workspaces.

Use a Linux runner with Bash and `mktemp`, such as a GitHub-hosted Ubuntu
runner. Run the action from trusted workflow configuration.

The caller owns triggers, permissions, quality-check dependencies,
deployment authorization, and concurrency. Serialize operations targeting
the same state and avoid cancelling an active apply. State locking also
requires a backend configured to support it.

Authentication stays in the caller because infrastructure and backend
credentials can differ. For example, Hetzner infrastructure with an S3
backend requires both Hetzner and AWS authentication.

### Inputs

Composite action inputs are strings. Accepted values are case-sensitive.

| Input | Required | Default | Accepted values |
| --- | --- | --- | --- |
| `working_directory` | Yes | None | Existing, non-empty repository-relative directory |
| `action` | No | `Provision` | `Provision`, `Destroy` |
| `should_apply` | No | `'false'` | `'true'`, `'false'` |

`Provision` produces a normal Terraform plan. Such a plan can include
creation, updates, replacement, or deletion according to the configuration.
It does not mean "create only."

`Destroy` produces a plan to destroy the resources managed by the selected
state. Neither operation is applied unless `should_apply` is `'true'`.

### Usage

After checkout, Terraform setup, and authentication, invoke the action
as a step:

```yaml
- name: Plan Hetzner infrastructure
  uses: pashkulev-devops-projects/github-actions/actions/terraform@<COMMIT_SHA>
  with:
    working_directory: terraform/hetzner
```

Replace `<COMMIT_SHA>` with the full commit SHA of a reviewed release
containing this action.

The default is plan only. An authorized deployment job can explicitly set:

```yaml
with:
  working_directory: terraform/hetzner
  action: Provision
  should_apply: 'true'
```

A playground caller can instead pass a root such as
`aws/projects/ansible-clients`.

### Design decisions

#### Shared execution, local orchestration

A composite action reuses steps inside the caller's job. This allows
authentication and deployment policy to remain explicit in each repository.

A single `working_directory` input replaces the original cloud-specific
directory construction. The action therefore supports different repository
layouts without knowing provider names or project naming conventions.

The existing quality workflow remains a separate reusable workflow because
its checks own independent jobs.

#### Validate the action's own input contract

A manual workflow may constrain inputs through dropdowns and checkboxes,
but another workflow can call this action directly with arbitrary strings.

Validation prevents a misspelled `Destroy` from silently becoming a normal
plan, or an invalid boolean string from silently skipping apply. Invalid
inputs fail before Terraform runs.

Directory validation rejects empty, absolute, and nonexistent paths. It is
a configuration check, not a filesystem sandbox: it does not enforce
containment against parent-directory traversal or symlinks.

#### Pass shell inputs through environment variables

GitHub expressions are resolved before Bash executes a script. Substituting
an input directly into shell source can allow its contents to alter the code.

Step-level environment variables keep input values separate from shell
source. Quoted expansions preserve each value as data.

Environment variables declared on a step do not carry over to another step.
Validation and planning therefore each declare `TERRAFORM_ACTION`.

#### Use the runner's Bash error handling

Explicit `shell: bash` makes GitHub invoke Bash with `-e -o pipefail`:

- `-e` stops execution on command failures, subject to Bash control-flow rules.
- `pipefail` preserves failures from commands earlier in a pipeline.

The multiline scripts add `set -u` to reject expansion of unset variables.
Repeating `set -euo pipefail` is unnecessary in this execution environment.

#### Make Terraform noninteractive

`-input=false` disables prompts. Missing configuration causes a failure
instead of requesting interactive input.

`TF_IN_AUTOMATION` suppresses suggestions about which commands to run next.
It is cosmetic and does not disable prompts or authorize application.

#### Preserve reviewed provider dependencies

`terraform init -lockfile=readonly` prevents initialization from modifying
the committed provider dependency lock file while still installing
dependencies and checking recorded checksums.

Update provider selections and required checksums deliberately, review the
lock-file changes, and commit them before deployment.

This flag concerns provider dependencies, not Terraform state locking.
It does not pin remote module versions; module references belong in the
Terraform configuration.

#### Save the plan outside the source checkout

Saved Terraform plans can contain sensitive values even when terminal output
redacts them. The action keeps this generated file outside the checkout to
reduce accidental inclusion in commits or workspace artifacts.

`mktemp "$RUNNER_TEMP/terraform-plan.XXXXXX"` creates a unique file with
owner-only read/write permissions and returns its absolute path.
`$(...)` captures that path in `plan_file`. An additional `umask 077`
is unnecessary for this file because `mktemp` already restricts access.

The plan file's path is published through:

```bash
printf 'plan_file=%s\n' "$plan_file" >> "$GITHUB_OUTPUT"
```

`printf` substitutes the path for `%s` and appends a newline. `>>` appends
to GitHub's step-output file without overwriting other outputs.

The step's `id: plan` makes the path available internally as
`steps.plan.outputs.plan_file`. Only the path is published, not file contents.
It is recorded before Terraform planning so cleanup can also run after
a planning failure.

The action does not export a public plan output or upload a plan artifact.

#### Construct command arguments as a Bash array

`plan_args=(...)` stores each command-line argument separately.
Expanding `"${plan_args[@]}"` preserves argument boundaries, including
paths containing spaces. No `eval` or shell command string is needed.

`-destroy` is appended only when the validated operation is `Destroy`.

#### Apply the exact saved plan

Apply receives the plan generated during the same invocation. It does not
generate a replacement plan.

Supplying a saved plan already authorizes Terraform to execute it, so
`-auto-approve` is redundant. The action defaults to plan only; the caller
must explicitly enable apply.

There is no human approval gate between Plan and Apply inside this action.
A process requiring review of the exact saved plan before execution needs
separate planning and application jobs with controlled artifact transfer.

A later invocation creates a fresh plan; it does not reuse an earlier
plan-only run.

#### Wait a bounded time for state locks

Plan and Apply use `-lock-timeout=5m` to tolerate brief contention without
waiting indefinitely. This does not enable locking on an unsupported or
unconfigured backend and does not replace caller-level concurrency controls.

#### Clean up after success or failure

The cleanup condition uses `always()` so earlier failures do not skip it.
The non-empty output check skips cleanup when no plan path was published.

`rm -f -- "$PLAN_FILE"` deletes the generated file:

- `-f` avoids prompting and tolerates an already-missing file.
- `--` prevents the filename from being interpreted as an option.
- Quoting preserves the path as one argument.

Cleanup does not delete Terraform state or infrastructure, and it does not
hide an earlier failure. It is best-effort during cancellation or runner
failure; a terminated runner cannot execute cleanup.

### Verification before release

Verify the action with a stub Terraform executable and a disposable
local-state configuration before migrating callers:

- Valid default inputs run Init and Plan but skip Apply.
- Provision and Destroy pass the intended planning arguments.
- Invalid inputs fail before Terraform executes.
- Initialization or planning failure prevents application.
- Apply receives the exact generated plan path.
- Temporary plans are removed after success and planning/application failures.
- Paths containing spaces remain single arguments.
- The temporary plan file has owner-only permissions.

These are release checks, not claims that verification has already passed.

### References

- [GitHub workflow shell behaviour](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idstepsshell)
- [Terraform environment variables](https://developer.hashicorp.com/terraform/cli/config/environment-variables)
- [Terraform init](https://developer.hashicorp.com/terraform/cli/commands/init)
- [Terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
- [Terraform apply](https://developer.hashicorp.com/terraform/cli/commands/apply)
