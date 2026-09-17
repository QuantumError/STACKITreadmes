# STACKIT Cloud Foundation – Feature Success Criteria

Companion document to `feature-list-20260915.md`. Same feature IDs (`CC-`, `L0-`, `L1-`, …), one section per feature with concrete, checkable success criteria.

**Purpose:** remove ambiguity about what "done" means for each feature. Each feature gets 1–3 bullets that describe an observable outcome — something a reviewer could actually check — rather than a restatement of the feature name. If a criterion can't be checked, it gets rewritten.

**Status:** draft, in review. All layers (`CC-`, `L0-`–`L10-`) have been drafted; ready for a final pass with the project group.

## Cross-cutting — Multi-customer configurability

### CC-01 — All customer-identifying values are parameters
- Deploying a second, fictitious customer requires changing only variable values (`company_code`, `company_name`, `owner_email`, address ranges, region, contacts) — no `.tf` file is touched.
- A search for the TUNI-specific company code/name across `src/**/*.tf` (excluding `.tfvars`/config files) returns no matches.

### CC-02 — Repeatable onboarding process (not just repeatable code)
- A written, step-by-step onboarding runbook exists and has been followed successfully, end-to-end.
- The runbook states how long onboarding a new customer takes and which manual (non-`tofu apply`) steps are required.
- This might be hard to test, since we cannot necessarily produce multiple fresh customer accounts to repeat the process - you may need to tear down existing installations, but this is why we are using IaC.

### CC-03 — No secrets committed to the IaC repository
- Every required secret (service account keys, pre-shared keys, API tokens) is documented as either a `TF_VAR_*` environment variable or a Secrets Manager reference — never as a literal value in a committed file.
- [OPTIONAL] An automated secret scan (e.g. a pre-commit hook or CI job) runs on every push and fails the build if it detects a credential pattern.

### CC-04 — Documented assumptions and defaults
- Every module used in the deployment has a README listing its inputs, defaults, and assumptions, without needing to read the `.tf` source to understand them.
- The onboarding documentation has a dedicated "Assumptions & Defaults" (or equivalent) section that lists every default value a new customer's parameters could conflict with — e.g. default region, default IP ranges, naming pattern — so it can be checked before the first apply.

### CC-05 — Minimal input set for onboarding a new customer
- A single onboarding config file (e.g. a `.tfvars`) exists with a documented, bounded list of required fields, separate from optional/advanced configuration.
- Someone unfamiliar with the codebase can identify which variables are mandatory for onboarding within 5 minutes, using only the documentation.

## Layer 0 — Organization & governance basics

### L0-01 — Folder hierarchy
- The folder structure (names, count, owner/reader assignments) is driven entirely by configuration (`rm_folders` or equivalent) — a customer can rename, add, or remove folders without editing `.tf` files. The accelerator's four folders (Platform, Landing Zones – Corporate, Landing Zones – Public, Sandboxes) are its default, not a fixed structure.
- Folders are created, updated, and removed entirely through `tofu apply`/`destroy` — no manual folder changes in the STACKIT Portal.
- A new project (management, connectivity, or a landing zone) lands under the correct folder automatically based on its configuration, without manually setting its parent in the Portal.

### L0-02 — Consistent naming convention
- Every resource created by the deployment (projects, buckets, Secrets Manager instances, service accounts, etc.) follows the same documented naming pattern, checkable by listing resources in the STACKIT Portal/CLI after apply.
- Changing the customer identifier (e.g. `company_code`) in configuration and re-applying produces consistently renamed resources across every module — nobody has to hunt down and edit a hardcoded name in a specific module.
- The naming pattern (format, allowed characters, length limits) is documented well enough that a team member can predict a resource's name before running `tofu apply`.

### L0-03 — Org-level roles: owners & auditors
- Adding an email to the owners/auditors configuration and re-applying grants that person the correct organization-level role — owner gets full control, auditor gets read-only — with no manual role assignment in the Portal.
- Removing an email from the configuration and re-applying revokes that person's organization-level access.
- Logging in as an auditor account confirms read-only access at the organization level: resources and audit logs are visible, but create/update/delete actions are rejected.

### L0-04 — Custom roles (narrower than built-in)
- At least one custom role is actually defined and used in the deployment (not just a theoretical capability) for a case where a built-in role (owner/editor/reader) would have been too broad.
- Defining a custom role in configuration (name, description, permissions) and applying creates it in STACKIT Authorization, visible in the Portal's role list at the level (org or project) it was defined for.
- A user assigned only the custom role can perform exactly the listed permissions — an action outside that permission set is rejected when tested.

### L0-05 — Enforced tagging convention
- A required set of label keys (e.g. cost-center, environment, owner) is defined and documented for the deployment.
- [OPTIONAL] Whether a missing/empty required label key fails `tofu plan`/`apply` validation (e.g. a Terraform `validation` block or CI check) or just logs a warning is configurable per customer, instead of being hardcoded on or off.
- Every resource in the deployed environment can be listed/filtered by the required label keys via the STACKIT Portal or CLI, confirming they're consistently present, not just applied to some resources.

## Layer 1 — Identity & access baseline

### L1-01 — Human user role assignment
- Adding a user's email and role to a project's role-assignment configuration and applying grants that user exactly the specified role, visible in the STACKIT Portal's IAM view for that project.
- Removing the entry and re-applying revokes the user's access to that project. (Isolation between projects is covered separately under L1-07.)

### L1-02 — Group-based role assignment
- A role can be assigned to a named group rather than listing individual emails, and every current member of that group has the role's access.
- Adding or removing a member in the group changes their effective access without editing the landing zone's `role_assignments` list.
- [CAVEAT] Confirm with STACKIT (support/docs) whether Authorization supports group subjects at all before treating this as achievable — our research found none today. If it's genuinely unsupported, the fallback success criterion is: individual-email assignment (L1-01) plus a documented, recurring access-review process that keeps the email list in sync with the intended group membership.

---

### L1-03 — Service accounts for automation
- The management project has a dedicated service account (not a human user) that pipelines authenticate as to run Terraform/OpenTofu, verifiable in the STACKIT Portal's service account list.
- The service account's key rotates automatically on a defined schedule without manual intervention, and a pipeline run right after a rotation still succeeds.
- Checking pipeline configuration/logs confirms no human user's personal credentials are ever used to run the pipeline.

### L1-04 — Workload-scoped service account
- The number of service accounts in a landing zone project is configurable — zero, one, or several, each distinct from the platform automation SA and from other landing zones' service accounts — rather than the accelerator's current hardcoded one-per-project assumption, since STACKIT itself doesn't limit a project to a single service account.
- Each defined service account's key rotates automatically on a schedule, verifiable per workload the same way as L1-03.
- A workload's service account cannot read or write resources in a different landing zone project — tested by attempting a cross-project action and confirming it's rejected.

---

### L1-05 — Pipeline-to-cloud authentication
- A dedicated Secrets Manager "pipelines" user exists with the minimum role it actually needs (read-only unless write is required), separate from any human user's credentials.
- That user's username/password is stored only as a STACKIT Git repository/organization secret — never appearing in a workflow file or committed code.
- A pipeline run retrieves the required secret (e.g. a service account key) via `hashicorp/vault-action` and uses it to authenticate Terraform/OpenTofu against STACKIT Cloud, with no credential value appearing in pipeline logs.

---

### L1-06 — Human SSO / external IdP (e.g. Entra ID)
- STACKIT IdP federation is configured (e.g. via the Microsoft Entra ID Enterprise App, or generic OIDC/SAML) so users log in to the STACKIT organization with their existing corporate IdP credentials, without a separate STACKIT-only password.
- SCIM-based provisioning is enabled so that deactivating a user in the corporate IdP automatically deprovisions their STACKIT access, without a manual STACKIT-side step.
- The federation setup is documented as an onboarding step (support ticket + configuration), since STACKIT IdP federation is platform-level and not something the Terraform accelerator configures.

---

### L1-07 — Per-project role scoping
- A user with a role on one landing zone project does not see any other landing zone project in the STACKIT Portal/CLI project list, unless separately granted access there.
- A role scoped to a single project grants no implicit access at the folder or organization level — the user cannot view sibling projects under the same folder.

## Layer 2 — Platform automation & state management

### L2-01 — Dedicated management project
- A dedicated project, distinct from every customer workload project, hosts the platform automation resources (state bucket, Secrets Manager, automation service account).
- No landing zone or customer workload resources are ever created inside the management project — checking the project's resource list shows only platform automation resources.
- Destroying and recreating a landing zone project does not affect the management project or its resources (state, secrets stay intact).

---

### L2-02 — Remote state storage for the IaC codebase
- Terraform/OpenTofu state for the deployment lives in a remote Object Storage bucket configured in `backend.tf` — not on a local disk or committed to git.
- The state bucket has object versioning enabled, so an accidentally deleted or overwritten state *file* can be restored to a prior version. (This recovers the state file itself, not the real infrastructure — if applied resources have already changed, reconciling actual drift after a restore is a separate, manual step.)
- Access to the state bucket is restricted to the automation service account (and explicitly authorized humans) — it is not publicly readable or writable.

---

### L2-03 — Central secrets storage for platform credentials
- A Secrets Manager instance exists in the management project, and every platform-level credential (automation SA key, firewall API credentials, VPN pre-shared keys, etc.) is stored there instead of in `.tfvars` or CI/CD variables.
- Retrieving a stored secret at runtime (e.g. via the STACKIT CLI/Portal, or a pipeline's vault-action step) returns the current value, confirming it's actually usable and not just written once and forgotten.
- Rotating a stored credential updates its value in Secrets Manager without anyone manually copying it elsewhere.

---

### L2-04 — CI/CD pipeline applying the IaC (plan reviewed before apply)

*(Branching/deployment convention is not yet decided — open questions: monorepo vs. one repo per customer account, and the branching/deployment model, e.g. whether a feature branch can deploy to a test environment. The criteria below are written to hold regardless of how those are resolved.)*

*(Bootstrap note: creating the Git repository the pipeline runs from is a one-time manual step — a pipeline can't apply the module that would create its own repository. Documented as such, not assumed to happen via `tofu apply`.)*

- A pipeline workflow runs `tofu plan` for a proposed change and makes the plan output visible for review before that change reaches any real environment — whatever the trigger (pull request, branch push, etc.) turns out to be.
- `tofu apply` against any environment only runs through the pipeline, with a trigger/approval gate appropriate to that environment (e.g. auto-apply to a test environment from a feature branch, gated approval for production) — not by a developer running `tofu apply` from their own machine in normal operation.
- Past plan/apply runs and their outputs are retained and viewable, so a reviewer can trace which pipeline run made which infrastructure change, regardless of which branch or environment it targeted.

---

## Layer 3 — Networking foundation

### L3-01 — Shared private address space

*(Design note: by default, STACKIT Network Area creates automatic system routes between every project attached to it, so corporate landing zones can reach each other unless something actively suppresses this — the accelerator only does so (`system_routes = false`) in the firewall flavor. Plain Hub-Spoke without a firewall is a flat, open network by default; if e.g. test and production must not reach each other, that isolation has to be a deliberate design choice, not assumed.)*

- A STACKIT Network Area is created with the configured total address range, and every corporate landing zone's network is created inside it rather than as a standalone network.
- Whether two landing zones (e.g. test and production) can reach each other over private IPs is a deliberate, reviewable configuration choice — not an accidental side effect of both being attached to the same Network Area.
- Adding a new corporate landing zone automatically gets a subnet allocated from the shared Network Area within the configured min/max/default prefix-length bounds, without manual IP planning.

---

### L3-02 — WAN routing table for internet egress
- A routing table with a single default route (`0.0.0.0/0 → internet`) is created in the connectivity project, verifiable via the STACKIT Portal/CLI.
- A VM in a corporate landing zone attached to this routing table can reach the public internet (a successful outbound connection test) when no firewall is deployed.
- The same routing table is reused by every corporate landing zone that needs direct internet egress, rather than each landing zone defining its own separate route.

---

### L3-03 — DNS zones with automatic per-workload delegation
- A hub DNS zone exists in the connectivity project, and applying a new landing zone automatically creates and delegates its child zone, without any manual DNS console work.
- A DNS record created inside a landing zone's child zone resolves correctly (e.g. via `dig`/`nslookup`), with no manual step needed in the parent hub zone.
- Removing a landing zone cleans up its delegated child zone — no orphaned delegation left pointing at a decommissioned project.

---

### L3-04 — Address plan decided upfront
- The total address range and min/max/default subnet prefix lengths are defined once, in configuration, before any landing zone is created.
- Requesting a landing zone subnet outside the configured min/max prefix-length bounds fails validation rather than silently succeeding with an unplanned size.
- The chosen total range and default prefix length are checked against a realistic estimate of how many landing zones the project will need, so the plan doesn't run out of address space partway through.

---

### L3-05 — Standalone/internet-facing networking (public LZ)
- Setting a landing zone to non-corporate creates it with its own independent network, not attached to the shared Network Area.
- A VM in a public landing zone reaches the internet directly, without depending on the connectivity project or the WAN routing table.
- A public landing zone cannot reach a corporate landing zone (or vice versa) over private IPs by default — confirmed by an actual connection test, not just by configuration intent.

---

## Layer 4 — Security guardrails: network & firewall

### L4-01 — Firewall at the network boundary
- Deploying the Hub-Spoke + Firewall flavor creates an OPNsense VM in the connectivity project, positioned between corporate landing zones and the internet (WAN/LAN interfaces as documented).
- Outbound traffic from a corporate landing zone to the internet passes through the firewall's LAN interface — confirmed by a traceroute or firewall traffic log showing the connection, not just by configuration.
- Whether a deployment includes a firewall at all (Standalone / Hub-Spoke / Hub-Spoke + Firewall) is a documented, explicit decision made per deployment, not an accidental default.

---

### L4-02 — Default-deny once configured
- Once the firewall policy has been pushed, traffic on a port/protocol with no matching allow rule is dropped — verified by attempting such a connection and confirming rejection.
- The risk window between the first apply (unconfigured appliance, GUI reachable from the internet) and the second apply (policy pushed) is actively mitigated — e.g. the GUI is bound to the LAN interface only, per the `getting-started.md` guidance, rather than left open on the public IP in between.
- Every allow rule in the pushed policy is traceable to a documented reason — no leftover "allow all" or unexplained rule survives initial setup.

---

### L4-03 — Centralized egress with a stable public IP
- Outbound internet traffic from every corporate landing zone routed through the firewall exits with the same public IP — verified by checking the source IP seen by an external service from more than one landing zone.
- The public IP stays the same across a routine firewall config change/apply — it doesn't shift unexpectedly and break a customer's allow-list.

---

### L4-04 — HA firewall pair (no single point of failure)
- Deploying with the HA option set creates two firewall appliances in separate availability zones, sharing a CARP virtual IP on the LAN side.
- Failing over from the primary to the backup node (e.g. by stopping the primary) keeps traffic flowing within roughly a second, without any landing zone route needing to change.
- The backup node's firewall policy stays in sync with the primary — a rule change made on the primary is confirmed present on the backup too, not left running a stale or empty ruleset.

---

### L4-05 — Documented traffic paths (incl. VPN handling)
- A written document lists each traffic direction relevant to the deployed flavor (LZ→internet, LZ→LZ, LZ→on-prem VPN, on-prem→LZ VPN) and states explicitly whether it passes through the firewall.
- For any traffic direction that bypasses the firewall, the document explains why and what compensating control, if any, exists.
- A new team member reading only this document — not the Terraform source — can correctly predict, before testing, whether a given connection is inspected.

---

### L4-06 — Security Groups baseline (instance/subnet-level filtering)
- Every landing zone VM/network has a Security Group applied by default — not just the debug bastion — with default-deny inbound and only the ports/protocols the workload actually needs allowed.
- This baseline holds in every deployment flavor, including Standalone (no firewall VM at all), confirming it doesn't depend on the optional OPNsense appliance.
- Security Group rules are defined in Terraform alongside the rest of the landing zone, so a rule change goes through the same plan/review process as any other infrastructure change.

---

## Layer 5 — Security guardrails: policy-as-code & compliance

### L5-01 — General policy-as-code guardrail framework
- A policy-as-code tool (e.g. OPA/Conftest, or an equivalent validation step) runs automatically in the pipeline against every proposed Terraform change.
- At least one concrete org-wide rule (e.g. "region must be eu01", "required labels present") is defined and demonstrably blocks a plan that violates it.
- A violation produces a clear, actionable error message in the pipeline output — not a silent pass or an opaque failure a reviewer has to reverse-engineer.

---

### L5-02 — Region enforcement
- A deployment attempt targeting a region outside the approved list fails validation (e.g. a Terraform `validation` block on the `region` variable, or the L5-01 policy engine) before any resource is created.
- The list of approved regions is defined in exactly one place, not scattered or duplicated across modules.
- Changing the approved region list is a single configuration change, not a search-and-replace across the codebase.

---

### L5-03 — CSPM / continuous compliance scanning
- STACKIT CSPM (the native, fully-managed dashboard) is enabled for the organization/project and shows visibility into the deployed resources without any agent or extra deployment.
- Deliberately introducing a known misconfiguration (e.g. a public bucket, an overly permissive Security Group) gets flagged by CSPM, with a guided mitigation step, within its normal scan interval.
- Findings are checked as part of a routine (e.g. a recurring calendar item or ticket), not left to accumulate unread in the dashboard.

---

### L5-04 — Secrets-in-Kubernetes guardrail
- Applying a plain Kubernetes `Secret` manifest in a workload namespace is rejected by the Kyverno policy's admission control, with a clear error message.
- A workload can still get secret values into its pods via the approved path (e.g. External Secrets synced from STACKIT Secrets Manager), confirming the guardrail blocks the anti-pattern without blocking the legitimate need.
- The policy applies to every workload namespace, not just the demo namespace it currently ships with.

---

### L5-05 — Break-glass exception process
- Enabling the firewall's break-glass/audit mode lets traffic that would otherwise be blocked pass through while logging it, rather than silently dropping it — verifiable by testing a blocked rule in audit mode.
- Using the break-glass path is auditable: who enabled it, when, and for how long is recorded, not just a silent toggle.
- Break-glass mode is off by default and requires a deliberate action to enable — it isn't the steady-state configuration.

---

## Layer 6 — Observability & audit

### L6-01 — Central audit log (org/project level)
- Making a change via the Portal or API (e.g. a role assignment, a resource creation) at the organization or project level produces an audit log entry in the Logs instance, including who made the change and when.
- Audit logs are archived to object storage in addition to the Logs instance, so they survive beyond the Logs instance's own retention window.
- If WORM/object-lock is enabled, an attempt to delete or modify an archived audit log entry before its retention period expires is rejected.

---

### L6-02 — Metrics, logs, traces for platform health
- The management project's Observability instance receives metrics/logs from the platform's own resources (e.g. the firewall VM), viewable in a dashboard without extra per-resource setup.
- If a platform Kubernetes cluster is deployed, its Observability instance similarly receives cluster-level metrics/logs (node health, control plane) out of the box.
- An operator can tell, using only the Observability instance, whether the platform's core components are currently healthy — without needing to log into anything directly.

---

### L6-03 — Metrics, logs, traces per workload
- Enabling the observability option for a landing zone provisions a dedicated Observability instance scoped to that project, separate from the platform's own instance.
- Metrics/logs from a workload running in that landing zone appear in its own Observability instance, without leaking into or requiring access to another landing zone's instance.
- A workload team member with access only to their landing zone project can view their own metrics/logs without needing platform-level access.

---

### L6-04 — Configurable retention for compliance
- Logs, traces, and metrics (including downsampled metrics) each have independently configurable retention periods, confirmed by setting different values for each and observing that data expiration follows each setting.
- Changing the retention value for one data type does not affect the others' retention.

---

### L6-05 — Dashboards
- At least one dashboard is provisioned automatically via IaC for a deployed workload, showing its key metrics without manual creation in the Grafana UI.
- The dashboard is scoped to the correct project/namespace — a viewer only sees their own workload's data, not another workload's.
- Recreating (destroy + apply) the workload also recreates its dashboard, confirming it's managed as code, not a one-off manual artifact that would be lost or orphaned.

---

### L6-06 — External telemetry export (e.g. New Relic)
- A Telemetry Router destination forwards data to an external OTLP endpoint (e.g. New Relic) in parallel with the existing STACKIT-native destinations (Logs, archive bucket) — the internal path keeps working unchanged.
- Data sent to the external destination is visible and queryable in that tool (e.g. logs/traces show up in New Relic) within a reasonable delay, confirming the pipe actually works end-to-end, not just that the destination was created.
- The credentials used to authenticate to the external endpoint (e.g. a New Relic license key) are stored in Secrets Manager and referenced by the Telemetry Router destination config, never committed as a literal value.

---

## Layer 7 — Kubernetes platform

### L7-01 — Managed Kubernetes cluster
- Deploying the platform Kubernetes module creates an SKE cluster with the configured node pools (machine types, min/max sizes, availability zones), verifiable via `kubectl get nodes`.
- Maintenance windows (Kubernetes version updates, machine image updates) are configured and take effect during the specified window, not at arbitrary times.
- Adding or resizing a node pool via a configuration change and apply updates the cluster accordingly, without manual console intervention.

---

### L7-02 — Per-tenant namespace isolation
- Applying the namespace-service module for a workload creates a dedicated namespace, a scoped service account, and a Role limited to that namespace.
- A pod running with the workload's service account cannot read or write resources in another workload's namespace — tested by attempting a cross-namespace action and confirming it's denied.
- Adding a new workload's namespace-service entry doesn't require any manual RBAC edits in the cluster — it's fully driven by the module's configuration.

---

### L7-03 — Encrypted volumes (KMS-backed)
- Enabling the encrypted volumes option creates a storage class backed by a customer-managed KMS key, not a platform-default key — verifiable by inspecting the storage class's key reference.
- A persistent volume created with this storage class is actually encrypted with that key, confirmed via the KMS keyring/key resource in the STACKIT Portal/CLI, not just assumed from the storage class name.
- Rotating or revoking the KMS key is a deliberate, documented action for the team — not something that could happen accidentally and lock out data.

---

### L7-04 — Debug bastion
- Enabling the debug bastion creates a VM with SSH access restricted to the configured allowed CIDRs, not open to the whole internet in real use.
- The bastion has `kubectl` pre-configured and can reach the cluster's API for troubleshooting, without extra manual setup.
- The bastion can be disabled or destroyed when not needed, without affecting the cluster or workloads — confirming it's a genuinely optional, bolt-on aid, not a hidden dependency.

---

### L7-05 — Blocks direct Kubernetes Secret management
- Same capability and criteria as L5-04 — see that section. Listed here too because the feature list tracks it at both the guardrail layer and the Kubernetes-platform layer, but there's only one implementation and one test.

---

### L7-06 — Ingress/gateway + DNS record automation
- Deploying a workload with a Gateway API route automatically creates a matching DNS record pointing at the ingress, without a manual DNS step.
- The resulting hostname resolves and successfully reaches the deployed service from outside the cluster.
- This works for any workload's namespace-service entry, not only the specific `sample_load` demo resource it currently ships with.

---

## Layer 8 — Workload landing zones

### L8-01 — Isolated project per workload/environment
- Adding a new landing zone entry to configuration and applying creates a fully separate STACKIT project, distinct from every other landing zone's project.
- Removing a landing zone entry and applying tears down its project and all attached resources (network, RBAC, secrets, storage) cleanly, without leaving orphaned resources behind.
- Two landing zones for the same workload but different environments (e.g. prod and staging) are fully independent projects — a change to one doesn't affect the other.

---

### L8-02 — Automatic network attachment
- Setting a landing zone's networking mode to "corporate" attaches its network to the shared Network Area automatically, without manual network configuration.
- Setting it to "public"/non-corporate instead creates a standalone network, with the mode switch being the only configuration change needed.
- Flipping the mode on an existing landing zone (corporate ↔ public) is either cleanly supported or explicitly documented as unsupported/destructive — not left ambiguous.

---

### L8-03 — Isolated secrets storage and object storage per workload
- Each landing zone gets its own Secrets Manager instance and object storage bucket(s), distinct from every other landing zone's.
- A secret or object stored in one landing zone is not accessible using another landing zone's credentials/service account — tested by attempting a cross-landing-zone read and confirming it's denied.
- Deleting a landing zone's Secrets Manager instance/buckets doesn't affect any other landing zone's secrets or storage.

---

## Layer 9 — Running actual workloads

### L9-01 — Managed database instance reachable from a workload project
- A workload can provision a STACKIT PostgreSQL Flex instance (`stackit_postgresflex_instance`) scoped to its own landing zone project via IaC, with connection credentials created as a `stackit_postgresflex_user` and delivered through Secrets Manager rather than a plaintext Terraform output.
- The instance's ACL is set to only the specific source ranges that need access (e.g. the landing zone's own network) — never `0.0.0.0/0` — confirmed by inspecting the applied ACL entries.
- A workload pod/VM in the same landing zone can connect to the database using the Secrets-Manager-delivered credentials, without the credentials ever appearing in Terraform state output or pipeline logs.

---

### L9-02 — Small containerized application
- A workload team can deploy a small containerized application into their landing zone using only their own configuration, with networking/RBAC/secrets already wired by the platform — no platform-level decisions left for them to make.
- The deployed container can reach its own workload-scoped secrets/storage (L8-03) and, if applicable, its own database (L9-01) without additional platform-level setup.
- For landing zones that don't use the shared Kubernetes platform, there's a documented, equally simple path to run a container — not "this only works if you also opt into Kubernetes."

---

### L9-03 — VM-based application hosting (Linux Server)
- A workload team can provision an application VM (`stackit_server`) inside their own landing zone project, with networking, RBAC, and Security Groups (L4-06) already wired — no platform-level networking decisions left to make.
- The VM boots with only the Security Group rules it actually needs open (default-deny otherwise), not a wide-open default.
- The pattern is documented and reusable — a second workload can get its own application VM by following the same steps/module, not a one-off hand-built VM.

---

### L9-04 — Automated OS patch management
- Enabling Server Update Management for a workload VM applies OS updates automatically on a defined schedule (outside working hours), without someone manually SSHing in to patch.
- A critical/emergency patch can be triggered ahead of the normal schedule, confirming the fast-track path actually works, not just the routine one.
- The update schedule and scope (which servers are covered) are set through Terraform/CLI/API, not a manual per-server Portal click each time.

---

### L9-05 — Automated server backups
- Enabling Server Backup Management for a workload VM creates backups automatically on a custom schedule, without manual intervention.
- Restoring a backup (a full volume, or a partial file recovery via restoring the backup to a new volume) actually succeeds and produces usable data — tested at least once, not just assumed to work.
- The backup retention period is explicitly configured (not left at whatever the default happens to be), and old backups are confirmed to age out per that setting.

---

### L9-06 — Public load balancer in front of a workload VM
- Traffic to a workload is distributed across more than one backend VM via a STACKIT Load Balancer, with an unhealthy backend automatically removed from rotation by the health check.
- The workload is reachable through the load balancer's address, not by pointing DNS directly at a single VM's IP.
- Adding or removing a backend VM (e.g. scaling out) is a configuration change to the load balancer's target list, not a manual DNS or routing change.

---

## Layer 10 — Optional extras

### L10-01 — Site-to-site VPN
- Deploying the VPN option creates an HA IPsec gateway with two tunnels in separate availability zones, each with its own public IP.
- After the two-phase apply (gateway first, then the connection once the remote peer IP is known), traffic from a corporate landing zone reaches a resource on the remote/on-premises network over the tunnel, and vice versa.
- Pre-shared keys are supplied via a separate variable, kept out of the main config object, so they never appear in a committed `.tfvars` file.

---

All features from `feature-list-20260915.md` are now covered.