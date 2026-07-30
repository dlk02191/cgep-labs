# OSCAL Documents

Machine-readable control documentation for this repository.

## `components/compliant-s3.json`

Component Definition for the Terraform module at `terraform/primitives/compliant-s3`.
Declares four NIST 800-53 Rev 5 controls, each naming the Terraform resource that
enforces it:

| Control | Terraform resource |
|---|---|
| `sc-28` Protection of Information at Rest | `aws_s3_bucket_server_side_encryption_configuration.primary` |
| `ac-3` Access Enforcement | `aws_s3_bucket_public_access_block.primary` |
| `au-3` Content of Audit Records | `aws_s3_bucket_logging.primary` |
| `cm-6` Configuration Settings | `aws_s3_bucket_versioning.primary` |

## `profiles/cge-p-minimum.json`

Selects those four controls from NIST's published 800-53 Rev 5 catalog.

## Evidence

Every `implemented-requirements` entry carries a `links[rel=evidence]` href pointing to a
signed pipeline bundle in the S3 Object Lock vault, produced by the Lab 4.3/4.4 workflow.
Verify with:

```bash
bash scripts/verify-evidence.sh <run_id> --vault <bucket> --profile default
```

Authored with `compliance-trestle` 4.2.0 (OSCAL 1.2.1). Both documents pass
`trestle validate`; output captured in `evidence/lab-6-1/trestle-validate.txt`.