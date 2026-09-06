# workloads-poc

GitOps proof of concept for provisioning stateful compute from one JSON definition
per server. It models an internal compute platform where an IDP writes definitions
and a pipeline reconciles them with a provider adapter.

The released Terraform roots live in
[`amirasyraf/workloads-templates-poc`](https://github.com/amirasyraf/workloads-templates-poc).
Only the AWS provider adapter is implemented in this POC.

## Architecture

```text
IDP or manual workflow
        |
        v
workloads/<workload>/aws/<os>/<server>/terraform.tfvars.json
        |
        v
apply-workload.yml on the definitions branch
        |
        +--> released provider root from workloads-templates-poc
        +--> one S3 state object and lock file per server
        +--> EC2 + no-ingress security group + SSM role
```

The reconciliation workflow parses the provider directory and checks out the
matching released root as `.workload-template/<provider>`. The Terraform
`init`, `validate`, `plan`, `apply`, and `output` steps are shared across
providers. Provider authentication is an adapter step; AWS is the first adapter
implemented. The selected S3 state backend is a separate state-service concern.

`main` is the default branch and holds the platform code. `definitions` is the
mutable branch used by the IDP simulation and by reconciliation. The workflows
exist on both branches so pushes to `definitions` are actionable.

The OS family and version are part of immutable server identity. Changing OS
means requesting a separate server definition and state, never modifying an
existing instance in place.

## Repository Layout

```text
.github/workflows/
  apply-workload.yml       # reconciles changed definitions
  bootstrap-state.yml      # creates the central S3 state bucket
  ci.yml                   # validates workflow syntax
  request-server.yml       # manual IDP simulation
mise.toml                  # pinned local and CI tool versions
workloads/
  <workload>/<provider>/<os>/<server>/terraform.tfvars.json
```

## Repository Variables

| Variable | Example | Purpose |
| --- | --- | --- |
| `AWS_OIDC_ROLE_ARN` | `arn:aws:iam::134584031874:role/github-aws-role` | GitHub OIDC role in the state account |
| `AWS_STATE_ACCOUNT_ID` | `134584031874` | Account allowed to own the backend bucket |
| `AWS_STATE_REGION` | `ap-southeast-1` | Region containing the state bucket |
| `AWS_TARGET_ACCOUNT_ID` | `134584031874` | Only account in which workloads may be created |
| `TF_STATE_BUCKET` | `amirasyraf-workloads-poc-tfstate-134584031874` | Globally unique S3 bucket name |
| `TEMPLATE_VERSION` | `v0.3.0` | Template used for new manual requests |

AWS authentication uses OIDC only. No static AWS access keys are required or
supported. Both state and workloads are restricted to the `amirasyraf` account
(`134584031874`). Terraform's AWS provider rejects credentials for any other
account; no cross-account role assumption is used.

## Local Tools

The repository and every pipeline install their pinned tools through mise.

```bash
mise install
mise run check
```

## Bootstrap

1. Publish the required template release in `workloads-templates-poc`.
2. Configure the repository variables above.
3. Run **Actions > Bootstrap Terraform state > Run workflow** once.

The bootstrap is idempotent. It creates the S3 bucket if needed, enables
versioning and default encryption, blocks public access, and enforces TLS.
Terraform uses S3 native lock files, so no DynamoDB table is needed.

## Test A Request

Run **Actions > Request AWS server > Run workflow** from `main`. The workflow
validates the request, resolves the current vendor AMI through AWS's public SSM
parameter, pins that AMI ID in the JSON definition, commits the file to
`definitions`, and explicitly dispatches reconciliation.

The current request harness exposes the AWS adapter and defaults to public subnet
`subnet-0a1d48f2ff2bd0332` in the existing
`amirasyraf-amirasyraf-ap-southeast-1` VPC. There is no cross-account option. The
managed security group has no ingress. Human access is through AWS Systems
Manager Session Manager.

The resulting path is:

```text
workloads/<workload>/<provider>/<os>/<server_name>/terraform.tfvars.json
```

Direct pushes that add or modify definitions on `definitions` also trigger
reconciliation. Multiple changed definitions run independently in a matrix, and
concurrency is serialized per server.

## Definition Contract

| Field | Required | Description |
| --- | --- | --- |
| `template_version` | yes | Immutable `workloads-templates-poc` release tag |
| `aws_account_id` | yes | Must be `134584031874` |
| `aws_region` | yes | Target AWS region |
| `workload` | yes | Stable owning workload identifier |
| `server_name` | yes | Stable server identifier within the workload |
| `desired_state` | yes | `present` or `absent` |
| `os` | yes | `ubuntu-24.04` or `windows-2025` |
| `ami_id` | yes | Pinned vendor-owned AMI ID |
| `subnet_id` | yes | Existing target-account subnet |
| `instance_type` | yes | EC2 instance type compatible with the AMI architecture |
| `associate_public_ip_address` | yes | Whether EC2 assigns a public IPv4 address |
| `root_volume_size` | yes | Encrypted `gp3` root volume size in GiB |
| `additional_security_group_ids` | no | Extra existing security groups; defaults to `[]` |
| `additional_tags` | no | Extra tags; platform-reserved tags take precedence |
| `user_data` | no | Optional cloud-init or PowerShell data; do not put secrets here |

The AMI is pinned at request time so an unrelated definition update cannot
silently replace a stateful server after a vendor publishes a new image.

The supported provider directory is currently `aws`. The supported OS directories
are `ubuntu24` and `win2025`. The pipeline verifies that the provider and OS
directories agree with the definition and selects the matching provider root from
the released template repository.

## Update Or Destroy

Re-run the request workflow with the same workload, OS, and server name to
replace the definition and reconcile changes.

To destroy a server, run it with `desired_state` set to `absent`. The workflow
only changes that field on the existing definition, preserving the original
configuration and state identity. Deleting a definition is intentionally ignored
because deletion would discard the desired-state instruction needed for a safe
Terraform destroy.

Changing `template_version` on an existing definition is the explicit template
upgrade mechanism. Review release notes before doing so.
