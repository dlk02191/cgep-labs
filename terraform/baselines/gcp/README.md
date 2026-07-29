# GCP Security Baseline (Lab 5.4)

Identity-first security baseline for a single GCP project. Where the AWS
baseline (Lab 5.2) detects and reports, this one prevents: Org Policy rejects
non-compliant API calls outright, so there is no finding to triage.

## What this deploys

| Component | Purpose | Controls |
|---|---|---|
| Org Policy x3 | Reject non-compliant API calls | CM-6, AC-2, AC-3 |
| Workload Identity Federation | Keyless CI auth via GitHub OIDC | AC-2, IA-5 |
| Data Access audit logs x3 | Record reads and writes | AU-2, AU-12 |

## Org Policy constraints enforced

All three at project scope with `enforce = "TRUE"` (rejection, not audit-only):

- **`storage.uniformBucketLevelAccess`** (CM-6) — forces IAM-only bucket
  access, eliminating legacy per-object ACLs.
- **`iam.disableServiceAccountKeyCreation`** (AC-2) — blocks downloadable
  service-account JSON keys, the most common GCP credential-leak vector.
- **`compute.requireOsLogin`** (AC-3) — VM access via IAM identity rather
  than SSH keys in instance metadata.

Verified enforcement: `gcloud iam service-accounts keys create` against the
CI service account returns `FAILED_PRECONDITION`, and
`keys list --managed-by=user` returns zero rows. The action did not happen;
there was nothing to remediate.

## Lesson: Data Access logs are off by default

GCP splits audit logging in two. **Admin Activity** logs — who changed the
configuration — are always on and free. **Data Access** logs — who read or
wrote the data — are **off by default** and must be enabled per service,
because volume is high and ingestion bills.

The practical consequence: a default GCP project can answer "who made this
bucket public" but cannot answer "who downloaded the contents." The second
question is the one that matters during an incident, and the gap is only
discovered when it is already too late. This is the most common GCP audit
finding for exactly that reason.

Enabled here for `storage.googleapis.com`, `cloudkms.googleapis.com`, and
`iam.googleapis.com` — data, keys, and permissions.

Note that `google_project_iam_audit_config` is authoritative per service:
Terraform overwrites rather than merges, so console changes revert on the
next apply. Terraform is the source of truth.

## WIF trust boundary

The provider's `attribute_condition` restricts token exchange to a single
repository:

    assertion.repository == "dlk02191/cgep-labs"

GitHub signs OIDC tokens for every repository on the platform, so a valid
signature proves only that the token came from GitHub. Without this
condition, any repository could impersonate the service account. Same role
as the scoped `sub` claim in the AWS OIDC trust from Lab 4.3.

The bound service account holds `roles/viewer` and nothing more.

**Known limitation:** the condition is repo-scoped, not branch-scoped. A
production configuration would typically add
`assertion.ref == "refs/heads/main"` or require a named environment.

## Reproduction requirements

Two things are required that are not captured in the Terraform code:

**1. Quota project on ADC.** User-based Application Default Credentials do
not attribute API calls to a project by default; calls arrive under Google's
shared gcloud project, and the Org Policy API rejects them with
`SERVICE_DISABLED`. The Terraform Google provider needs this explicitly:

    export USER_PROJECT_OVERRIDE=true
    export GOOGLE_BILLING_PROJECT=cgep-labs

Set as environment variables rather than provider arguments to keep the
Terraform configuration unmodified.

**2. Project must have an Organization parent.** Org Policy is a
resource-hierarchy feature. `roles/orgpolicy.policyAdmin` can only be
granted at organization scope — it is not grantable on a project, and
`roles/owner` does not include it. A project with no parent organization
cannot hold org policies at all.

This project was originally created outside any hierarchy and had to be
migrated:

    gcloud organizations add-iam-policy-binding ORG_ID \
      --member="user:USER" --role="roles/orgpolicy.policyAdmin"
    gcloud beta projects move cgep-labs --organization=ORG_ID

That carve-out is deliberate governance design: if a project owner could
grant themselves Organization Policy Administrator on their own project,
they could disable any constraint imposed on them, and org policy would be
meaningless. Separation of duties enforced structurally rather than by
policy document.

## Not applicable

**Security Command Center.** Requires org-level activation and the
`securitycenter.googleapis.com` API. Not provisioned by this lab. The Org
Policy enforcements above are the preventive equivalent.

## Evidence

- `evidence/lab-5-4/iam-policy.json` — Data Access log configuration per
  service (AU-2 / AU-12).

## Apply

    export USER_PROJECT_OVERRIDE=true
    export GOOGLE_BILLING_PROJECT=cgep-labs

    terraform init
    terraform apply \
      -var="gcp_project=cgep-labs" \
      -var="github_repo=dlk02191/cgep-labs"