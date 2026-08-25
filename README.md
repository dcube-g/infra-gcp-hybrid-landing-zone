# infra-gcp-hybrid-landing-zone

A hands-on lab replicating a real hybrid-cloud landing zone: Shared VPC
networking, simulated on-prem connectivity over Cloud VPN, security
baselines enforced via Organization Policy, centralized logging, and a
tested backup/restore cycle with measured RTO.

Built to practice (and demonstrate) identity & network design, hybrid
connectivity troubleshooting, security/compliance controls, and disaster
recovery — the core competencies for a cloud infrastructure/platform role.

> **Note on redaction:** project IDs, billing account IDs, org IDs, and any
> personal email addresses shown below are placeholders
> (`<PROJECT_ID>`, `<ORG_ID>`, etc.) or have been blurred in screenshots.
> Real values were used during the actual build but are not published here.

---

## Architecture

![Architecture diagram](docs/architecture.png)

- **Shared VPC** host project with per-tier subnets (web / app / data)
  attached to prod and dev service projects
- **HA VPN + Cloud Router (BGP)** connecting to a simulated on-prem network
- **Deny-by-default firewall rules**, scoped per tier
- **Centralized logging** via aggregated sinks to BigQuery + GCS
- **Org Policy** enforcing a security baseline (no external IPs, OS Login
  required)
- **Scheduled snapshots + Cloud SQL PITR**, with a verified restore

---

## What I built — by phase

### 1. Identity & core networking
- Shared VPC host/service project topology, mirroring how enterprises
  segment prod/dev/shared networking without duplicating firewall and
  routing config per team.
- Least-privilege IAM: least-privilege custom role
  (`custom.lz.deployer`) scoped to only the permissions a deployer needs,
  rather than broad primitive roles.
- Screenshot: `docs/screenshots/01-shared-vpc.png`

### 2. Hybrid connectivity
- HA VPN + Cloud Router BGP session to a simulated on-prem network.
- **Real problem solved:** deliberately created overlapping IP ranges
  between the "on-prem" and prod networks to reproduce a routing conflict
  that comes up often in real on-prem-to-cloud migrations, then diagnosed
  and resolved it. Full writeup: [`troubleshooting.md`](troubleshooting.md).
- Screenshots: `docs/screenshots/02-vpn-tunnel-established.png`,
  `03-ip-overlap-error.png`, `04-ip-overlap-fixed.png`

### 3. Security baseline, tagging, logging
- Organization Policy constraints enforcing a security baseline (blocked
  external IPs, required OS Login).
- Mandatory labels (`environment`, `owner`, `project`, `cost-center`,
  `data-classification`) enforced via Terraform variable validation.
- Centralized logging with an aggregated sink and one working alert
  (firewall-deny spike) — chosen deliberately to demonstrate detection,
  not just log collection.
- Screenshot: `docs/screenshots/05-org-policy.png`

### 4. Backup & recovery
- Scheduled disk snapshots and Cloud SQL point-in-time recovery.
- Performed an actual timed restore into an isolated project rather than
  just confirming backup jobs succeeded — measured real RTO end to end.
- Screenshot: `docs/screenshots/06-backup-restore-timing.png`

---

## Real issues hit during the build (and how I fixed them)

See [`troubleshooting.md`](troubleshooting.md) for full detail. Summary:

| Issue | Root cause | Fix |
|---|---|---|
| Billing quota exceeded on project link | Default 5-projects-per-billing-account quota | Checked usage first, freed up/planned around the limit |
| Project ID already in use | GCP project IDs are globally unique and reserved even after deletion | Adopted a session-scoped unique suffix convention for all project IDs |
| Shared VPC enable failed — "no organization" | Projects were created without an org parent | Created projects with `--organization` from the start; moved existing ones with `gcloud beta projects move` |
| Shared VPC enable failed — permission denied | Project Owner ≠ org-level Shared VPC admin | Granted `roles/compute.xpnAdmin` at the org level |
| Group-based IAM binding failed | `SOLO_GROUP_OWNERS_DISALLOWED` + nonexistent placeholder-domain group | Used direct least-privilege user bindings for this solo lab; documented the group-based pattern as the production-recommended approach |

---

## Tooling

`gcloud` CLI, Terraform (validation), Cloud Logging, Cloud Monitoring,
Organization Policy, Cloud DNS, Cloud Router/HA VPN, Cloud SQL.

## Naming & tagging conventions

Documented in [`naming-conventions.md`](naming-conventions.md) —
resource-type prefixes, environment/region suffixes, and the mandatory
label set applied to every resource.
