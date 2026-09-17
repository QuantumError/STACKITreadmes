# STACKIT Cloud Foundation – Feature List

Working document for the student project group delivering the Cloud Foundation on STACKIT. Builds on `landing-zone-background-and-features.md` and `original-idea.md`, and cross-references the locally cloned [STACKIT Landing Zone Accelerator](https://github.com/stackitcloud/stackit-landing-zone) (snapshot analyzed: commit `2aaa21d`, 2026-09-11).

## How to read this document

Each layer below has a table with these columns:

| Column | Meaning |
|---|---|
| **ID** | Stable identifier per feature: `L<layer>-<number>` for layered features (e.g. `L0-01`), or `CC-<number>` for the cross-cutting multi-customer requirements. Use these to reference features in discussions and future documents. |
| **Priority** | 1–5 scale, 5 = mandatory/essential. **Only features that are essential are pre-filled with a 5 at this stage.** Everything else is left blank — the group prioritizes those together. |
| **Feature** | The capability, in the same spirit as the layered list in `landing-zone-background-and-features.md`. |
| **Description** | What the feature means in practice. |
| **Status** | Whether the accelerator already delivers this: **Supported**, **Partial**, or **Missing**, plus a pointer to the module/file and a short note on gaps or caveats. |
| **STACKIT service(s)** | The underlying STACKIT service(s) the feature maps to. |

Layering follows the existing background document, with a few refinements based on what the accelerator actually looks like in practice.

## Cross-cutting — Multi-customer configurability

Not a feature layer on its own — a quality requirement across **all** layers below, since the landing zone must work as a repeatable pattern across customers rather than a one-off build for TUNI. All five items below are must-haves for this project.

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| CC-01 | 5 | All customer-identifying values are parameters | Name, code, address ranges, region, contacts never hardcoded | **Supported** — `company_code`, `company_name`, `owner_email`, ranges, region are all `.tfvars` inputs | n/a (Terraform design) |
| CC-02 | 5 | Repeatable onboarding process (not just repeatable code) | A documented, safe way to stand up a new customer instance | **Partial** — three reference `.tfvars` flavors exist, but `getting-started.md` is a manual, multi-step CLI bootstrap, not a scripted or self-service onboarding flow | STACKIT CLI (manual steps today) |
| CC-03 | 5 | No secrets committed to the IaC repository | Credentials never land in git, even by accident | **Supported** — secrets kept out of `.tfvars` in separate `TF_VAR_*` exports or gitignored files (e.g. `vpn_pre_shared_keys`, `firewall_api_credentials`) | Secrets Manager |
| CC-04 | 5 | Documented assumptions and defaults | Onboarding doesn't require re-reading all the code each time | **Supported** — extensive module READMEs, `architecture.md`, `getting-started.md` | n/a (documentation) |
| CC-05 | 5 | Minimal input set for onboarding a new customer | A small, well-scoped set of required variables | **Partial** — the three `.tfvars` flavors help, but the module surface is still large (dozens of variables); there's no thin onboarding wrapper on top | n/a |


## Layer 0 — Organization & governance basics

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L0-01 | 5 | Folder hierarchy | Create, update, and remove project folders that group resources by purpose (e.g. platform components, customer workloads, sandboxes) | **Supported** — `governance` module, `rm_folders` (4 default folders) | Resource Manager |
| L0-02 | | Consistent naming convention | `<company_code>-<layer>-<component>-<env>` applied to every resource | **Supported** — `naming_pattern` threaded through every module | Resource Manager (naming is a code convention, not a service) |
| L0-03 | 5 | Org-level roles: owners & auditors | Full-control owners, read-only auditors at the organization level | **Supported** — `governance` module, `organization_owners` / `organization_auditors` | Authorization (IAM) |
| L0-04 | 5 | Custom roles (narrower than built-in) | Fine-grained roles where owner/auditor/editor are too broad | **Supported** — org-level in `governance` (`custom_roles`) and project-level in `landing-zone` (`custom_roles`) | Authorization (IAM) |
| L0-05 | | Enforced tagging convention | Required tag keys (cost center, environment, owner) validated, not just applied | **Partial** — a generic `labels` map is applied everywhere, but nothing validates required keys or values | Resource Manager labels |

## Layer 1 — Identity & access baseline

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L1-01 | 5 | Human user role assignment | Per-project role grants for named users | **Supported** — `landing-zone` module `role_assignments` (subject = email) | Authorization |
| L1-02 | | Group-based role assignment | Assign a role to a group instead of listing individual emails | **Missing** — `role_assignments.subject` only accepts a user or service-account email; no group construct exists in the accelerator | Authorization (would need external group→role mapping, not modeled) |
| L1-03 | | Service accounts for automation | Pipelines authenticate as a dedicated SA, not personal credentials | **Supported** — `management` module provisions an automation SA with a 60-day rotating key (90-day TTL) | Service Accounts, Secrets Manager |
| L1-04 | | Workload-scoped service account | Each workload gets its own SA with a rotated key | **Supported** — same pattern repeated per landing zone in the `landing-zone` module | Service Accounts, Secrets Manager |
| L1-05 | | Pipeline-to-cloud authentication | STACKIT Git pipelines need a secure way to reach STACKIT Cloud/Terraform state without hardcoding credentials | **Supported** — documented pattern: a dedicated "pipelines" user is created in Secrets Manager (Vault-compatible API), its username/password is stored as a STACKIT Git repository/organization secret, and the `hashicorp/vault-action` (GitHub Actions-compatible) retrieves the actual service-account key at runtime for Terraform/OpenTofu to use.[^1] | Secrets Manager, STACKIT Git Pipeline secrets |
| L1-06 | | Human SSO / external IdP (e.g. Entra ID) | Organization login federated to a corporate identity provider | **Supported, platform-level** — STACKIT IdP supports SAML 2.0, generic OIDC, Google Workspace, and a dedicated Microsoft Entra ID Enterprise App integration, with SCIM-based provisioning for automated deprovisioning. Configured via a STACKIT support ticket, not self-service, and not modeled in the Terraform accelerator (it's an org-level platform setting, not a resource the provider manages).[^2] | STACKIT IdP |
| L1-07 | 5 | Per-project role scoping | A workload team sees only its own project | **Supported** — every landing zone is an isolated project with its own `role_assignments` | Resource Manager (project isolation) + Authorization |

## Layer 2 — Platform automation & state management

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L2-01 | 5 | Dedicated management project | Platform automation lives apart from customer workloads | **Supported** — `management` module (`<code>-pltfm-mgmt-prod`) | Resource Manager |
| L2-02 | 5 | Remote state storage for the IaC codebase | Versioned, access-controlled Terraform/OpenTofu state | **Supported** — Object Storage bucket in `management` module | Object Storage |
| L2-03 | 5 | Central secrets storage for platform credentials | Rotated keys and platform credentials never live in tfvars or CI variables | **Supported** — Secrets Manager instance in `management` module | Secrets Manager |
| L2-04 | | CI/CD pipeline applying the IaC (plan reviewed before apply) | Pull-request-driven change management | **Missing from the accelerator itself** — it ships no pipeline definitions (e.g. workflow files) at all; the delivery team has to author these, using the pipeline-to-cloud authentication pattern described in Layer 1.[^1] | STACKIT Git Pipelines (GitHub Actions-compatible) + Secrets Manager |

## Layer 3 — Networking foundation

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L3-01 | 5 | Shared private address space | All corporate landing zones (STACKIT projects) under the same customer account can reach each other over private IPs | **Supported** — `connectivity` module, STACKIT Network Area, configurable ranges/prefix lengths | Network Area (SNA) |
| L3-02 | 5 | WAN routing table for internet egress | Defined default route for outbound traffic | **Supported** — `connectivity` module `wan` routing table | Routing Tables |
| L3-03 | 5 | DNS zones with automatic per-workload delegation | Each workload gets a child zone without manual DNS work | **Supported** — hub zone in `connectivity`, auto-delegated child zone per workload in `landing-zone` | DNS |
| L3-04 | | Address plan decided upfront (range, subnet sizing rules) | Total range and min/max/default subnet size fixed before onboarding | **Supported** — `network_area` config block (`min_prefix_length`, `max_prefix_length`, `default_prefix_length`) | Network Area |
| L3-05 | 5 | Standalone/internet-facing networking (public LZ) | Workloads that don't need private connectivity get an independent network | **Supported** — `landing-zone` module, `corporate = false` path | Network |

## Layer 4 — Security guardrails: network & firewall

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L4-01 | | Firewall at the network boundary | Inspects traffic between workloads and the internet | **Supported, optional** — one of three deployment flavors (`connectivity` module OPNsense VM); the Standalone and Hub-Spoke flavors ship without it | Compute (VM) + Network |
| L4-02 | | Default-deny once configured | Explicit allow rules only | **Supported, with a caveat** — `firewall-config` module pushes the policy; the appliance is unconfigured and its GUI reachable from the internet between the two required applies (documented risk window) | firewall-config module / OPNsense |
| L4-03 | | Centralized egress with a stable public IP | Useful for customer-side allow-listing | **Supported** — static public IP on the firewall WAN interface | Public IP |
| L4-04 | | HA firewall pair (no single point of failure) | Active/passive CARP pair, ~1s failover | **Supported, optional** — `connectivity.firewall.ha` | Compute + Network |
| L4-05 | | Documented traffic paths (incl. VPN handling) | Clarity on what does/doesn't pass through the firewall | **Supported, as documentation** — `architecture.md` explicitly documents that both VPN directions bypass the appliance | n/a (design documentation) |
| L4-06 | 5 | Security Groups baseline (instance/subnet-level filtering) | Default-deny traffic filtering that works even without deploying the optional firewall VM | **Partial** — `stackit_security_group`/`_rule` resources exist and are used, but only to lock down the `debug-bastion` module's SSH access; no baseline Security Group is applied to landing zone VMs/networks in general | Security Groups |

## Layer 5 — Security guardrails: policy-as-code & compliance

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L5-01 | | General policy-as-code guardrail framework | Org-wide rules (allowed regions, SKUs, mandatory tags, etc.) enforced automatically | **Missing** — the accelerator has no broad guardrail engine; the only policy artifact is a single Kubernetes rule (see below) | Not modeled — would need e.g. OPA/Conftest in CI, or a policy product |
| L5-02 | | Region enforcement | Deployments blocked outside approved regions | **Partial** — `region` defaults to `eu01` and is threaded through, but nothing prevents targeting another region | n/a (convention only, not enforced) |
| L5-03 | 5 | CSPM / continuous compliance scanning | Ongoing posture scanning and drift detection against a baseline | **Supported, platform-level** — STACKIT Cloud Security Posture Management (CSPM) is a native, fully-managed dashboard offering automated misconfiguration/vulnerability detection and guided mitigation, with no agent or local deployment; it's a STACKIT Portal feature, not something the Terraform accelerator configures or wires up[^3] | STACKIT CSPM |
| L5-04 | | Secrets-in-Kubernetes guardrail | Workloads can't create raw `Secret` objects, forcing use of the managed secrets path | **Supported** — Kyverno policy blocks direct `Secret` creation in demo namespaces | Kyverno (self-hosted on SKE) + Secrets Manager |
| L5-05 | | Break-glass exception process | Documented, auditable way to bypass a guardrail under control | **Supported, firewall only** — `firewall_config.break_glass` block for the firewall's audit mode | firewall-config module |

## Layer 6 — Observability & audit

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L6-01 | 5 | Central audit log (org/project level) | Who changed what, retained for investigation | **Supported** — `management` module `audit_logs`: Telemetry Link → Telemetry Router → Logs instance + archive bucket, optional WORM (`s3_object_lock`) | Telemetry Router, Logs, Object Storage |
| L6-02 | 5 | Metrics, logs, traces for platform health | Baseline observability for the platform itself | **Supported** — `observability` block in `management` and `platform-kubernetes` modules | Observability |
| L6-03 | | Metrics, logs, traces per workload | Same baseline, scoped to each landing zone | **Supported, optional** — `observability` block in `landing-zone` module | Observability |
| L6-04 | 5 | Configurable retention for compliance | Retention tuned per data type, not one blanket setting | **Supported** — separate retention variables for logs/traces/metrics/downsampled metrics and archive retention | Observability, Object Storage |
| L6-05 | | Dashboards | Ready-made views instead of raw metrics | **Supported, demo only** — `namespace-service-demo` provisions a Grafana folder/dashboard | Grafana on Observability |
| L6-06 | | External telemetry export (e.g. New Relic) | Route audit logs/telemetry to a third-party observability tool the customer already uses, alongside STACKIT-native storage | **Missing from the accelerator, but available on STACKIT** — Telemetry Router destinations can forward in parallel to any OpenTelemetry-compatible endpoint (Basic or Token auth) in addition to the existing Logs/Object Storage destinations; New Relic ingests OTLP natively. The Terraform resource (`stackit_telemetryrouter_destination`) exists, but the accelerator's `audit_logs` only wires up STACKIT-internal destinations today[^5] | Telemetry Router (+ external OTLP endpoint, e.g. New Relic) |

## Layer 7 — Kubernetes platform (shared cluster)

Not represented as its own layer in the original background document, but the accelerator treats a shared SKE cluster as a parallel platform capability rather than "just a workload" — worth deciding explicitly whether it's in scope.

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L7-01 | | Managed Kubernetes cluster | Shared cluster with configurable node pools and maintenance windows | **Supported, optional** — `platform-kubernetes` module | STACKIT Kubernetes Engine (SKE) |
| L7-02 | | Per-tenant namespace isolation | Each workload gets its own namespace, scoped SA and Role | **Supported** — `namespace-service-demo` module | SKE |
| L7-03 | | Encrypted volumes (KMS-backed) | Storage class backed by a customer-managed key | **Supported, optional** | SKE + KMS |
| L7-04 | | Debug bastion | Controlled path for cluster troubleshooting | **Supported, optional** | Compute (VM) |
| L7-05 | | Blocks direct Kubernetes Secret management | Forces use of the managed secrets path (see Layer 5) | **Supported** — Kyverno policy | Kyverno + Secrets Manager |
| L7-06 | | Ingress/gateway + DNS record automation | A deployed service gets a working hostname automatically | **Supported, demo only** — `sample_load` demo proves the Gateway API route + DNS record path | SKE + DNS |

## Layer 8 — Workload landing zones (per team / per environment)

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L8-01 | 5 | Isolated project per workload/environment | e.g. `data-prod`, `api-staging`, each fully separate | **Supported** — `landing-zone` module, one instance per `for_each` entry | Resource Manager |
| L8-02 | 5 | Automatic network attachment | Joins the shared address space, or gets a standalone network | **Supported** — driven by the `corporate` flag | Network Area / Network |
| L8-03 | 5 | Isolated secrets storage and object storage per workload | No shared blast radius between workloads | **Supported** — Secrets Manager instance + object storage buckets per landing zone | Secrets Manager, Object Storage |

## Layer 9 — Running actual workloads

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L9-01 | | Managed database instance reachable from a workload project | A workload can provision and connect to a managed DB | **Missing from the accelerator, but available on STACKIT** — the STACKIT Terraform provider has an actively maintained `postgresflex` service (`stackit_postgresflex_instance`, `_user`, `_database`); it's just not wired into the `landing-zone` module today | STACKIT PostgreSQL Flex, Secrets Manager |
| L9-02 | | Small containerized application | A workload team can run a container without platform decisions left to make | **Partial** — only exists as a demo on the shared K8s platform (`sample_load`); there's no equivalent pattern for VM-based/non-K8s landing zones | SKE (demo only) |
| L9-03 | | VM-based application hosting (Linux Server) | A workload team can run an application directly on a VM, without needing the Kubernetes platform at all | **Missing from the accelerator** — `stackit_server` is only used for the firewall and debug bastion; there's no landing-zone pattern for provisioning a plain application VM | Linux Server (Compute Engine), `stackit_server` |
| L9-04 | | Automated OS patch management | Servers stay patched against known vulnerabilities on a schedule, without manual intervention | **Missing from the accelerator, but available on STACKIT** — the STACKIT Terraform provider has `stackit_server_update_enable` and `stackit_server_update_schedule` resources for scheduled OS updates (Windows & Linux) with fast-track emergency patching; not wired into any accelerator module[^4] | STACKIT Server Update Management |
| L9-05 | | Automated server backups | Scheduled, monitored backups of workload VM volumes with configurable retention and restore | **Missing from the accelerator, but available on STACKIT** — the STACKIT Terraform provider has `stackit_server_backup_enable` and `stackit_server_backup_schedule` resources for custom backup schedules, configurable retention, and restoring a volume backup to a new volume; not wired into any accelerator module[^4] | STACKIT Server Backup Management |
| L9-06 | | Public load balancer in front of a workload VM | Traffic distributed across one or more VMs with health checks, instead of pointing DNS at a single VM's IP | **Missing from the accelerator** — the STACKIT Terraform provider has `stackit_loadbalancer` (Network Load Balancer) and `stackit_application_load_balancer` resources; neither is used anywhere in the accelerator, and the only `LoadBalancer` reference is a Kubernetes-internal Service type inside the K8s demo[^4] | STACKIT Network/Application Load Balancer |

## Layer 10 — Optional extras

| ID | Priority | Feature | Description | Status | STACKIT service(s) |
|---|---|---|---|---|---|
| L10-01 | | Site-to-site VPN | Connects the shared address space to on-premises or another cloud | **Supported, optional** — `connectivity.vpn` block, HA dual-tunnel gateway, two-phase apply (gateway first, then connections once peer IPs are known) | IPsec VPN Gateway |

## Next step

Prioritize the blank cells with the project group: agree the remaining MVP scope (layers 0–6 and 9 are the strongest MVP candidates given accelerator coverage), what's a fast-follow, and what's explicitly out of scope — particularly the confirmed gaps (group-based RBAC, general policy-as-code guardrail framework) and the scope decision on the Kubernetes platform layer. A managed database (L9-01) is available on STACKIT (PostgreSQL Flex) but needs new module work to wire it into the accelerator — not a platform gap, just an accelerator gap. Human SSO (L1-06) and CSPM (L5-03) are both supported natively by STACKIT but sit outside the Terraform accelerator entirely (support-ticket setup for IdP, STACKIT Portal dashboard for CSPM) — worth their own scoping conversation. Security Groups (L4-06) are a quick win: the accelerator already uses them for the debug bastion, just not as a general landing-zone baseline. A VM-based workload path (L9-03–L9-06: Linux Server, OS patch management, backups, load balancer) is a real alternative to the Kubernetes platform layer (7) and worth weighing against it explicitly, since none of it is wired into the accelerator today either way. The cross-cutting multi-customer requirements (`CC-01`–`CC-05`) are already agreed as must-haves and don't need re-prioritizing.

## References

[^1]: [STACKIT Git — Pipelines - STACKIT Compatibility](https://docs.stackit.cloud/products/developer-platform/git/how-tos/pipeline-stackit-compatibility/), accessed 2026-09-15.
[^2]: [Getting Started with STACKIT IdP](https://docs.stackit.cloud/platform/access-and-identity/stackit-idp/getting-started/) and [Understand STACKIT IdP](https://docs.stackit.cloud/platform/access-and-identity/stackit-idp/), accessed 2026-09-15.
[^3]: [Cloud Security Posture Management (CSPM)](https://docs.stackit.cloud/products/security/cspm/) and [Security Hardening](https://docs.stackit.cloud/products/security/security-hardening/), accessed 2026-09-15.
[^4]: [Linux Server](https://docs.stackit.cloud/products/compute-engine/linux-server/), [Server Agent](https://docs.stackit.cloud/products/compute-engine/server-agent/), [Server Update Management](https://docs.stackit.cloud/products/compute-engine/server-update-management/), [Server Backup Management](https://docs.stackit.cloud/products/compute-engine/server-backup-management/), and [Load Balancing and Content Delivery](https://docs.stackit.cloud/products/network/load-balancing-and-content-delivery/), accessed 2026-09-15.
[^5]: [STACKIT Telemetry Router](https://docs.stackit.cloud/products/logging-and-monitoring/telemetry-router/) (Architecture, Destinations) and [New Relic — Introduction to OpenTelemetry](https://docs.newrelic.com/docs/opentelemetry/opentelemetry-introduction/), accessed 2026-09-16.

