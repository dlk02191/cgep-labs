# Policy Library

| Policy | Control ID | Severity | Remediation |
|--------|------------|----------|-------------|
| sc28_encryption.rego | SC-28 | High | Add a customer-managed encryption key (CMEK) to every Google Cloud Storage bucket. |
| ac3_no_public.rego | AC-3 | High | Prevent public bucket access and do not expose management ports (22/3389) to the internet. |
| cm6_required_tags.rego | CM-6 | Medium | Add the required labels: project, environment, managed_by, and compliance_scope. |