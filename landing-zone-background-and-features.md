# STACKIT Cloud Foundation – Background & Feature List

Working document for the project group. 

Purpose: shared understanding of how a managed service partner operates customer cloud environments on top of an IaC landing zone, and a feature list to prioritize together.

## 1. How a managed service partner (MSP) uses a landing zone

A landing zone is the repeatable baseline a partner deploys once per customer and then operates. Everything a customer's workloads run on — accounts, network, security guardrails — comes from the same code, not from manual clicking. This is what makes the operating model possible:

- **Onboarding a customer** means running the landing zone code with that customer's parameters (name, address ranges, region, contacts). The result is a working, secured environment on day one, not a blank cloud account.
- **Change management** happens through the code, not the console. Every change to firewall rules, network layout, or access is a pull request: reviewed, tested, and recorded. This gives the partner an audit trail for every change made to a customer's environment, which is exactly what change management processes require.
- **Access management** is handled through roles defined in code. When someone joins or leaves a project, or needs temporary elevated access, it's a role assignment change — reviewable and revocable, not a support ticket asking someone to "add a user" by hand.
- **Incident management** relies on the observability and audit logging the landing zone sets up automatically for every customer. When something breaks, the partner's service desk already has logs, metrics, and audit trails to work from, without asking the customer to enable anything first.
- **Service desk operations** sit on top of this baseline: tickets reference specific projects, roles, or network segments that all exist as named, documented resources in the IaC codebase — not tribal knowledge.
- **Multi-customer scale** works because the landing zone is a template, not a one-off build. The same module set is deployed repeatedly with different parameters per customer, so the partner's team doesn't relearn the environment each time — it looks and behaves the same everywhere.

In short: the landing zone is what turns "we manage your cloud" into a standardized, auditable, repeatable service instead of ad-hoc administration.

## 2. Feature list — organization basics to workloads on firewalled networks

Grouped in the order a deployment is typically built up. Priority (MVP / later) to be agreed with the project group — this is a working draft, not a decision.

### Layer 0 — Organization & governance basics
- Organization structure: a folder hierarchy separating platform resources, customer workload projects, and sandboxes
- Consistent naming convention for all resources (customer/company code, environment, component)
- Organization-level roles: owners (full control) and auditors (read-only)
- Custom roles where the built-in ones are too broad

### Layer 1 — Identity & access baseline
- Human user accounts and group-based role assignments
- Service accounts for automation (CI/CD pipelines), not personal credentials
- Federated authentication for pipelines (no long-lived keys checked into anywhere)
- Per-project role assignments so a workload team only sees its own project

### Layer 2 — Platform automation & state management
- A dedicated management project separate from customer workloads
- Remote state storage for the IaC codebase itself (versioned, access-controlled)
- Central secrets storage for platform credentials and rotated keys
- CI/CD pipeline that applies the IaC code (plan reviewed before apply)

### Layer 3 — Networking foundation
- A shared private address space that customer workload projects join
- A defined routing table for internet-bound (WAN) traffic
- DNS zones for the customer, with per-workload subdomains delegated automatically
- Address plan decisions made once, upfront (total range, subnet sizing rules)

### Layer 4 — Security guardrails (firewall)
- A firewall placed at the network boundary, inspecting traffic between workloads and the internet
- Default-deny posture once configured, with explicit allow rules
- Centralized egress with a stable public IP (useful for customer allow-listing on their side)
- High-availability firewall pair, so the network boundary isn't a single point of failure
- Documented traffic paths: what does and doesn't pass through the firewall (e.g. site-to-site VPN traffic needs special handling)

### Layer 5 — Observability & audit
- Central audit log of who changed what, at the organization and project level
- Metrics, logs and alerting for platform and workload health
- Retention settings appropriate for incident investigation and compliance needs

### Layer 6 — Workload landing zones (per team / per environment)
- One isolated project per workload and environment (e.g. `data-prod`, `api-staging`)
- Automatic network attachment to the shared address space, or a standalone network for internet-facing workloads
- Per-project role assignments for the application team
- Isolated secrets storage and object storage per workload
- A delegated DNS subdomain per workload
- A workload-scoped service account with a rotated key

### Layer 7 — Running actual workloads
- A managed database instance reachable from a workload project
- A small containerized application running inside a workload project
- DNS record automation so a running service gets a working hostname without manual DNS edits

### Layer 8 — Optional extras (candidates, scope TBD)
- Site-to-site VPN connecting the customer's private network to on-premises or another cloud
- Container platform DNS automation for services exposed via a gateway/ingress
- Using Entra ID as a Identity Provider

### Layer 9 — Multi-customer configurability (reusability requirement)
This layer isn't a feature on its own — it's a quality requirement that cuts across all layers above, since the landing zone must work as a repeatable pattern across multiple customers, not just once for TUNI Project:
- All customer-identifying values (name, code, address ranges, region, contacts) are parameters, not hardcoded
- A clear, minimal set of inputs needed to onboard a new customer
- Documented assumptions and defaults so onboarding doesn't require re-reading all the code each time
- A repeatable process (not just repeatable code) for standing up a new customer instance safely
- No secrets committed to the IaC repository — customer parameters live in config files, actual credentials stay in the secrets manager

## 3. Minimum Viable Product (Solita perspective)

TO BE DISCUSSED

A working, pre-configured package built from the STACKIT Landing Zone Accelerator, deployed using its most complete reference setup (shared network, firewall included) — proven end-to-end once, then usable as the template for future customers. Everything in layers 0–6 above is the core of that package; layers 7–9 extend it toward "ready to run real workloads" and "ready to repeat for another customer."

## 4. Next step

Prioritize the feature list above with the project group: agree what's in the MVP, what's a fast-follow, and what's explicitly out of scope for this project.
