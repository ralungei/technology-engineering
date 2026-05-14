# Programmatic Access to OCI for CI/CD Pipelines

Last reviewed: 2026-05-14

---

## Overview

When a CI/CD pipeline calls OCI APIs it needs to authenticate. The safest way to do this is to avoid storing credentials at all. A key sitting in a secret store can leak through a build log, get committed to a repo by mistake, or expire without anyone noticing.

OCI makes credential-free authentication possible through native identity: the pipeline resource proves who it is using its own OCI identity, with no keys involved. When the runner runs outside OCI, a short-lived token exchange can replace the static key. Static API keys are the last option, used only when nothing else is available.

The sections below help you pick the right approach for your setup. Configuration steps are in the implementation guides linked at the bottom.

## Options

```mermaid
%%{init: {'theme': 'dark', 'themeVariables': {'fontFamily': 'Oracle Sans, Helvetica Neue, Arial, sans-serif', 'fontSize': '14px', 'lineColor': '#64748B', 'edgeLabelBackground': '#1E293B', 'primaryColor': '#334155', 'primaryTextColor': '#E2E8F0', 'primaryBorderColor': '#4B5563'}}}%%
flowchart TD
    classDef decision fill:#334155,color:#E2E8F0,stroke:#64748B,stroke-width:2px
    classDef native   fill:#0891B2,color:#ffffff,stroke:#0E7490,stroke-width:2px
    classDef external fill:#059669,color:#ffffff,stroke:#047857,stroke-width:2px
    classDef fallback fill:#DC2626,color:#ffffff,stroke:#B91C1C,stroke-width:2px

    A{Runner inside OCI?}:::decision
    A -->|Yes| B{OCI resource type?}:::decision
    A -->|No| E{Platform supports OIDC?}:::decision
    B -->|Compute VM| C[Instance Principals]:::native
    B -->|OCI DevOps Pipeline| D[Resource Principals]:::native
    B -->|OKE Pod| F[OKE Workload Identity]:::native
    E -->|Yes — GitHub Actions / GitLab CI| G[Workload Identity Federation]:::external
    E -->|No — Legacy Jenkins / Scripts| H[API Signing Keys]:::fallback
```

| Method | Where the runner runs | Secrets to manage | Rotation |
| --- | --- | --- | --- |
| **Instance Principals** | OCI Compute VM | None | Automatic |
| **Resource Principals** | OCI DevOps Pipeline or Functions | None | Automatic |
| **OKE Workload Identity** | OKE pod (Enhanced Cluster only) | None | Automatic |
| **Workload Identity Federation** | Any external runner with OIDC | OAuth client only | Automatic |
| **API Signing Keys** | Anywhere | Private key, rotated every 90 days | Manual |

## What We Recommend

### Runner inside OCI

Pick the option that matches what runs the pipeline:

| Runner type | Method |
| --- | --- |
| Self-hosted agent on an OCI Compute VM | **Instance Principals** |
| OCI DevOps Build or Deploy Pipeline | **Resource Principals** — already configured inside the pipeline, nothing extra needed |
| Pod on an OKE Enhanced Cluster | **OKE Workload Identity** — scoped to the pod, not the node |

No keys. No rotation process. No secrets in your pipeline configuration.

### Runner outside OCI

If your platform supports OIDC (GitHub Actions, GitLab CI, and most modern CI tools do), use **Workload Identity Federation**. The only thing stored in your CI/CD system is an OAuth `client_id` and `client_secret` that can only be used to exchange tokens — it gives no access to OCI resources on its own.

### When nothing else works

If your tooling does not support OIDC and does not run inside OCI, use **API Signing Keys**. Follow the hardening steps in the implementation guide — they are not optional.

> Oracle's IAM security documentation lists API Signing Keys last in the preference order, after native principals and session-based methods.

## Security Basics

These apply regardless of which method you use:

- **Scope policies to a compartment.** Never use `manage all-resources in tenancy` for a CI/CD identity.
- **One identity per pipeline.** Do not share keys or dynamic group rules across unrelated teams or projects.
- **Check the audit logs.** OCI logs every API call with the full principal identity. Filter by principal OCID to confirm only expected actions are happening.
- **No credentials in code.** Keys must not appear in source files, Dockerfiles, or build specs — ever.

## License

Copyright (c) 2026 Oracle and/or its affiliates.
Licensed under the Universal Permissive License (UPL), Version 1.0.
See [LICENSE](LICENSE) for more details.
