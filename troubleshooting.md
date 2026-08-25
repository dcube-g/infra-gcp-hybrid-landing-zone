# Troubleshooting log — issues hit and resolved during this build

A running record of every real problem encountered while building this
lab, in the order they occurred. Kept deliberately unpolished — this is
what actually happened, not a cleaned-up retrospective.

---

## Phase 1 — Identity & networking

**Billing quota exceeded on project link**
`FAILED_PRECONDITION: Cloud billing quota exceeded`. Root cause: default
quota of 5 projects per billing account. Fixed by checking current usage
(`gcloud billing projects list`) before linking, and unlinking/deleting
unused projects to free up room.

**Project ID already in use**
GCP project IDs are globally unique and stay reserved even after deletion
(minimum 30-day grace period). Fixed by adopting a session-scoped unique
suffix (`$LZ_UNIQ`) appended to every project ID.

**Projects created outside the org**
`gcloud projects create` without `--organization` created standalone
projects with no org parent, which later blocked Shared VPC with `The
project has no organization`. Fixed by using `--organization` at creation
time, and by moving already-created projects with `gcloud beta projects
move`.

**Missing org-level Shared VPC permission**
`Required 'compute.organizations.enableXpnHost' permission` — project
Owner isn't enough for Shared VPC operations; needs `roles/compute.xpnAdmin`
granted separately at the org level.

**Group-based IAM binding failures**
`SOLO_GROUP_OWNERS_DISALLOWED` (a group can't be the sole Owner) and
`Group ... does not exist` (the guide's `@example.com` placeholder domain
was never meant to be typed literally — real Cloud Identity groups need a
verified domain and a Workspace admin-console step). Resolved by using
direct least-privilege user bindings for this solo lab, with group-based
IAM documented as the production-recommended pattern instead.

**Two organizations sharing the same display name**
`gcloud organizations list` returned two entries both named
`deegee-ck-org` with different IDs. An IAM grant on the wrong one would
have silently failed downstream permission checks. Resolved by confirming
which org actually owned the lab projects (`gcloud projects describe
... --format="value(parent.id)"`) before granting anything.

## Phase 2 — Hybrid connectivity

**Cross-project VPN peer gateway 404**
`--peer-gcp-gateway` needs the full resource URL when the peer gateway
lives in a different project — a bare name only resolves within the same
project as the command.

**Missing reverse VPN tunnel**
HA VPN requires both directions established; only creating the tunnel from
host → onprem left the connection incomplete.

**DNS zone creation missing required field**
`--description` is required on `gcloud dns managed-zones create` but
wasn't in the original command — every zone creation failed until added.

**On-prem simulation VM was never actually specified**
The naming convention referenced `vm-lz-onprem-us-central1-01` but no
create command existed for it. Had to add it, plus discover and fix two
more gaps once it existed:
- **No outbound internet access** — a private VM (`--no-address`) needs
  Cloud NAT to reach `apt-get` repositories; without it, package installs
  fail with `Network is unreachable`.
- **SSH access needs three separate things**, not just a firewall rule —
  a firewall rule allowing IAP's range (`35.235.240.0/20`), the
  `roles/iap.tunnelResourceAccessor` IAM role, and the IAP API enabled.

## Phase 3 — Security baseline, logging, alerting

**Legacy Org Policy command/schema doesn't work**
`gcloud resource-manager org-policies enable-enforce` and the legacy
`allValuesFromParent` YAML schema are both rejected by the current API.
The modern `gcloud org-policies set-policy` command with a `spec.rules`
YAML is the only thing that works — and there's no `enable-enforce`
subcommand in the modern command group at all, so boolean constraints go
through `set-policy` too.

**Missing org-level Org Policy permission**
Same permission-layering pattern as Shared VPC — needed
`roles/orgpolicy.policyAdmin` at the org level, separate from project
Owner.

**Terraform label-validation block pasted into bash**
A Terraform HCL variable block isn't shell syntax — it errored as invalid
bash when pasted directly. This build has no live Terraform project behind
it, so labels were applied directly via `gcloud alpha projects update
--update-labels` instead (the `--update-labels` flag is alpha-track only
on the stable `projects update` command).

**Logging sink permission and destination gaps**
- Needed `roles/logging.admin` at the org level to create sinks.
- The BigQuery dataset and GCS bucket destinations have to exist *before*
  the sinks that reference them.
- Each sink writes as its own auto-created service account
  (`service-org-<ORG_ID>@gcp-sa-logging.iam.gserviceaccount.com`), which
  needs explicit write access granted on the destination.
  `bq add-iam-policy-binding` was allowlist-gated in this org
  (`This feature requires allowlisting`) — worked around with a
  `bq show`/`bq update` JSON patch instead.

**Alert policy command doesn't support threshold flags**
`--condition-threshold-value` / `--condition-threshold-comparison` don't
exist on `gcloud alpha monitoring policies create` — alert conditions are
nested objects, only settable via `--policy-from-file`.

**Alert filter can't reference raw log fields**
`jsonPayload.disposition` isn't a valid alert-condition filter target —
Monitoring conditions only accept `metric`/`resource`/`group`/`project`
prefixed filters. Had to create a log-based metric
(`gcloud logging metrics create`) first, then alert on that metric.

**Metric's resource type didn't match the source log's resource type**
Assumed the log-based metric would register under `gce_firewall_rule`
(matching the source logs) — it actually registered under `global`.
Confirmed via Monitoring Console → Metrics Explorer after two failed
guesses, since the error (`The resource name does not represent a known
descriptor`) didn't say which resource type was actually valid.

## Phase 4 — Backup & recovery

**Shared VPC silently reverted mid-build**
The most significant issue in the whole session. VM creation failed
repeatedly with `Cross-project references for this resource are not
allowed` despite using textbook-correct syntax. Diagnosis process:
1. `gcloud compute networks subnets list-usable` showed only the
   auto-created `default` subnets — none of the Shared VPC subnets at all.
2. `gcloud compute shared-vpc get-host-project` on the service project
   returned an empty `{}` — the attachment wasn't real.
3. Attempting to re-attach failed with `'...is not a shared VPC host
   project'` — the **host project's Shared VPC-enabled status itself**
   had reverted, not just the attachment.
4. Re-running `gcloud compute shared-vpc enable` produced a truncated,
   easy-to-miss error about an unset `core/project` config value.
5. **Root cause:** Cloud Shell's active project (`gcloud config get-value
   project`) had gone `(unset)` at some point in the session. Despite every
   command carrying an explicit `--project=` flag, `shared-vpc enable`'s
   internal state resolution depended on the active config project too,
   and silently failed to persist without it.
6. **Fix:** `gcloud config set project "prj-lz-net-host-${LZ_UNIQ}"`
   before re-running `enable`, then re-attaching both service projects.
   Confirmed via `list-usable` showing the real subnets before retrying
   VM creation, which then succeeded immediately.

**Cloud SQL private IP needs servicenetworking enabled on both projects**
Enabling `servicenetworking.googleapis.com` on the host project (for the
peering range) wasn't sufficient — the **service project** where the
instance actually lives also needs it enabled, or instance creation fails
with `SERVICE_NETWORKING_NOT_ENABLED`.

**Wrong PITR flag for the database engine**
`--enable-bin-log` is MySQL-specific and silently did nothing on a
`POSTGRES_15` instance — no error, but point-in-time recovery stayed
disabled. The correct flag, `--enable-point-in-time-recovery`, worked
once identified. Also needed a few minutes of transaction-log history to
accumulate before a point-in-time clone would succeed.

**Restore verification — disk status isn't enough**
A `READY` disk and `RUNNING` instance status don't actually prove
recoverability on their own. Verified properly by SSHing into the
restored VM via IAP and confirming a command actually executed —
that's the point at which "the backup succeeded" became "the service is
genuinely recoverable."

---

## Takeaways worth stating explicitly in an interview

- **Permission layering is the recurring theme**: project-level Owner
  repeatedly wasn't sufficient for org-scoped operations (Shared VPC,
  Org Policy, Logging) — each needed its own specific org-level role
  granted separately. Worth calling out as a deliberate GCP design pattern,
  not a series of unrelated bugs.
- **State can silently revert** — the Shared VPC issue wasn't a one-time
  setup mistake, it broke *after* working correctly earlier in the same
  session. Always re-verify state with a read command before debugging
  syntax, rather than assuming a step that worked once is still true.
- **Read the actual API error, not just the wrapper message** — several
  of these (the alert filter, the PITR flag, the DNS description field)
  were solved by the specific wording of a 400 error rather than by
  guessing at fixes.
