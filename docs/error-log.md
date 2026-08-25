# Complete error log — GCP hybrid landing zone build

Every error actually encountered during this build, in chronological order,
with the exact error text, root cause, and fix applied. Unlike
`troubleshooting.md` (which summarizes the *story*), this is the raw
incident log — useful as a personal reference and as evidence of real,
independent troubleshooting.

---

## 1. Billing quota exceeded

```
ERROR: (gcloud.billing.projects.link) FAILED_PRECONDITION: Precondition check failed.
- '@type': type.googleapis.com/google.rpc.QuotaFailure
  violations:
  - description: 'Cloud billing quota exceeded: https://support.google.com/code/contact/billing_quota_increase'
    subject: billingAccounts/01A629-7D8924-CDD7F4
```
**Cause:** default quota of 5 projects per billing account.
**Fix:** checked usage with `gcloud billing projects list --billing-account=...`
before linking; freed up room by unlinking/deleting unused projects.

---

## 2. Project ID already in use

```
ERROR: (gcloud.projects.create) Project creation failed. The project ID you specified is already in use by another project. Please try an alternative ID.
```
(repeated 5x across a batch create attempt)
**Cause:** GCP project IDs are globally unique and stay reserved even after
deletion (30-day minimum grace period).
**Fix:** adopted a session-scoped unique suffix (`$LZ_UNIQ`) appended to
every project ID going forward.

---

## 3. Invalid project ID characters

```
ERROR: (gcloud.projects.create) INVALID_ARGUMENT: field [project_id] has issue [project_id contains invalid characters]
```
**Cause:** `$LZ_UNIQ` was unset in that shell session (set in a different
terminal/tab), so the project ID resolved to `prj-lz-net-host-` with a
trailing hyphen — invalid syntax.
**Fix:** re-exported `LZ_UNIQ` in the correct session and persisted it to
`~/.bashrc`.

---

## 4. Billing link permission error (downstream of #3)

```
ERROR: (gcloud.billing.projects.link) [deegee.ck@gmail.com] does not have permission to access projects instance [prj-lz-net-host-] (or it may not exist)
```
**Cause:** symptom of #3 — the project was never created, so nothing
existed to link billing to.
**Fix:** resolved once #3 was fixed.

---

## 5. Bash syntax errors from pasted placeholder/Terraform text

```
-bash: variable: command not found
-bash: syntax error near unexpected token `('
-bash: validation: command not found
```
**Cause (x2, different occasions):** first, a literal `<ORG_OR_FOLDER_ID>`
placeholder was pasted into a command unmodified — bash interpreted `<` and
`>` as redirection operators. Second, a Terraform HCL block was pasted
directly into bash, which isn't valid shell syntax at all.
**Fix:** replaced placeholders with real values before running; for the
Terraform block, wrote it to a `.tf` file via heredoc instead of executing
it as a command, since this build has no live Terraform project behind it.

---

## 6. Org policy propagation / wrong command family

```
ERROR: (gcloud.resource-manager.org-policies.enable-enforce) [deegee.ck@gmail.com] does not have permission to access projects instance [prj-lz-app-prod-jd01:setOrgPolicy]
ERROR: (gcloud.resource-manager.org-policies.set-policy) INVALID_ARGUMENT: Invalid JSON payload received. Unknown name "allValuesFromParent" at 'policy.list_policy'
```
**Cause:** two separate issues — missing `roles/orgpolicy.policyAdmin` at
the org level, and use of the legacy `resource-manager org-policies`
command with an outdated policy schema.
**Fix:** granted the role; switched entirely to the modern `gcloud
org-policies set-policy` command with the current `spec.rules` YAML schema.

```
ERROR: (gcloud.org-policies.describe) NOT_FOUND: A Policy of constraint constraints/compute.requireOsLogin ... does not exist
```
**Cause:** the `enable-enforce` command silently never took effect due to
the permission gap above.
**Fix:** re-ran after confirming the role grant, using `set-policy` instead.

```
ERROR: (gcloud.org-policies) Invalid choice: 'enable-enforce'.
```
**Cause:** `enable-enforce` doesn't exist at all in the modern `org-policies`
command group — leftover syntax from the legacy command family.
**Fix:** used `set-policy` with a YAML file for both boolean and list
constraints.

---

## 7. Invalid org ID argument

```
ERROR: (gcloud.organizations.add-iam-policy-binding) INVALID_ARGUMENT: Request contains an invalid argument.
```
**Cause:** `$ORG_ID` was empty when the command ran.
**Fix:** confirmed via `echo`, then set explicitly.

```
ORG_ID is: [<the-id-from-step-1>]
```
**Cause:** the literal placeholder text `<the-id-from-step-1>` was assigned
to the variable instead of a real value — a copy-paste-without-substitution
mistake.
**Fix:** ran the `describe` command to get the real org ID, then set
`ORG_ID` explicitly to that literal number.

---

## 8. Two organizations with identical display names

```
DISPLAY_NAME: deegee-ck-org
ID: 172919905395
DISPLAY_NAME: deegee-ck-org
ID: 43414815430
```
**Not an error, but a real risk** — an IAM grant could silently land on
the wrong org. Confirmed which org actually owned the lab projects before
proceeding with any further org-level grants.

---

## 9. IAM group binding failures

```
ERROR: (gcloud.projects.add-iam-policy-binding) INVALID_ARGUMENT: Request contains an invalid argument.
- type: SOLO_GROUP_OWNERS_DISALLOWED
  member: group:grp-lz-prod-owners@example.com
```
**Cause:** GCP disallows a group being the *sole* Owner on a project.

```
ERROR: (gcloud.projects.add-iam-policy-binding) INVALID_ARGUMENT: Group grp-lz-dev-editors@example.com does not exist.
```
**Cause:** `@example.com` was a placeholder domain, never meant to be typed
literally; real Cloud Identity groups need a verified domain and
Workspace admin-console creation.
**Fix (both):** switched to direct user-bindings as the default IAM path
for this solo lab.

---

## 10. Shared VPC organization/permission errors

```
ERROR: (gcloud.compute.shared-vpc.enable) Could not enable [prj-lz-net-host-jd01] as XPN host:
 - Invalid resource usage: 'The project has no organization.'
ERROR: (gcloud.compute.shared-vpc.associated-projects.add) Could not enable resource [...]:
 - Invalid resource usage: 'The project has no organization.'
```
**Cause:** projects were created without `--organization`, landing as
standalone projects.
**Fix:** moved all 5 projects into the org with `gcloud beta projects move`.

```
ERROR: (gcloud.compute.shared-vpc.enable) Could not enable [prj-lz-net-host-jd01] as XPN host:
 - Required 'compute.organizations.enableXpnHost' permission for 'projects/prj-lz-net-host-jd01'
ERROR: (gcloud.compute.shared-vpc.associated-projects.add) Could not enable resource [...]:
 - Required 'compute.organizations.enableXpnResource' permission for 'projects/prj-lz-net-host-jd01'
```
**Cause:** project Owner isn't sufficient for Shared VPC operations —
needs `roles/compute.xpnAdmin` at the org level specifically.
**Fix:** granted the role, waited ~60s for propagation, retried.

---

## 11. VPN tunnel cross-project reference (first occurrence)

```
ERROR: (gcloud.compute.vpn-tunnels.create) HTTPError 404: The resource 'projects/prj-lz-net-host-jd01/regions/us-central1/vpnGateways/vpn-lz-onprem-us-central1' was not found.
```
**Cause:** `--peer-gcp-gateway` was passed as a bare name, which gcloud
resolved against the wrong (local) project instead of the project where
the peer gateway actually lives.
**Fix:** used the full resource URL for `--peer-gcp-gateway`; also added
the missing reverse tunnel (host→onprem alone isn't sufficient for HA VPN).

---

## 12. Cloud DNS missing required field

```
ERROR: (gcloud.dns.managed-zones.create) HTTPError 400: The 'entity.managedZone.description' parameter is required but was missing.
```
(occurred twice, once per zone)
**Fix:** added `--description` to both zone-creation commands.

```
ERROR: (gcloud.dns.managed-zones.create) HTTPError 400: Invalid value for 'entity.managedZone.forwardingConfig.targetNameServers[0]': '<onprem-sim-dns-server-ip>'
```
**Cause:** the literal placeholder text was left in the command instead of
a real IP.
**Fix:** created the actual on-prem-sim VM (which hadn't been built yet at
all — a gap in the guide), retrieved its real internal IP, used that.

---

## 13. On-prem VM didn't exist, then couldn't reach the internet, then couldn't SSH

```
ERROR: (gcloud.compute.instances.describe) Could not fetch resource:
 - The resource 'projects/prj-lz-onprem-jd01/zones/us-central1-a/instances/vm-lz-onprem-us-central1-01' was not found
```
**Cause:** the create command for this VM was never actually run — it only
existed in the naming table, not as an executed step.
**Fix:** created it explicitly.

```
Cannot initiate the connection to deb.debian.org:443 ... Network is unreachable
Cannot initiate the connection to packages.cloud.google.com:443 ... connection timed out
```
**Cause:** VM had `--no-address` (private-only) and no NAT gateway, so no
outbound internet path for `apt-get`.
**Fix:** created a Cloud NAT gateway attached to the existing Cloud Router.

```
ERROR: (gcloud.compute.ssh) [/usr/bin/ssh] exited with return code [255].
```
**Cause:** three separate missing pieces for IAP-tunneled SSH — firewall
rule allowing IAP's range (`35.235.240.0/20`), the
`roles/iap.tunnelResourceAccessor` IAM role, and the IAP API not enabled.
**Fix:** added all three.

---

## 14. Alert policy command/schema errors (longest chain in the build)

```
ERROR: (gcloud.alpha.monitoring.policies.create) unrecognized arguments:
  --condition-threshold-value=50
  --condition-threshold-comparison=COMPARISON_GT
```
**Cause:** these flags don't exist on this command — alert conditions are
nested objects, only settable via `--policy-from-file`.

```
ERROR: (gcloud.alpha.monitoring.policies.create) INVALID_ARGUMENT: Name must begin with 'projects/{project_id}/notificationChannels/{channel_id}', got: <paste-the-full-channel-name-from-step-1>
```
**Cause:** literal placeholder text left in the JSON file.
**Fix:** substituted the real channel name via `sed`.

```
ERROR: (gcloud.alpha.monitoring.policies.create) INVALID_ARGUMENT: Field alert_policy.conditions[0].condition_threshold.filter had an invalid value of "resource.type="gce_firewall_rule" AND jsonPayload.disposition="DENIED"": The lefthand side of each expression must be prefixed with one of {group, metadata, metric, project, resource}
```
**Cause:** raw log fields (`jsonPayload.disposition`) aren't valid alert
filter targets — only metrics are.
**Fix:** created a log-based metric first (`gcloud logging metrics create`).

```
ERROR: (gcloud.alpha.monitoring.policies.create) INVALID_ARGUMENT: ... "metric.type="logging.googleapis.com/user/lz-firewall-deny-count" AND resource.type="gce_firewall_rule"": The resource name does not represent a known descriptor.
```
**Cause:** assumed the metric would register under the source log's
resource type (`gce_firewall_rule`) — it actually registered under `global`.

```
ERROR: (gcloud.alpha.monitoring.policies.create) INVALID_ARGUMENT: ... "metric.type="logging.googleapis.com/user/lz-firewall-deny-count"": must specify a restriction on "resource.type" in the filter
```
**Cause:** removing `resource.type` entirely also isn't valid — one is
required, just not the guessed value.
**Fix:** confirmed the correct resource type (`global`) via Monitoring
Console → Metrics Explorer, used that in the filter. **Policy created
successfully on this attempt.**

---

## 15. Logging sink permission and destination errors

```
ERROR: (gcloud.logging.sinks.create) [deegee.ck@gmail.com] does not have permission to access organizations instance [43414815430] ... Permission 'logging.sinks.create' denied
```
**Fix:** granted `roles/logging.admin` at the org level.

```
BigQuery error in add-iam-policy-binding operation: This feature requires allowlisting.
```
**Cause:** `bq add-iam-policy-binding` is allowlist-gated on this org for
dataset-level bindings.
**Fix:** used `bq show --format=prettyjson` → edited the `access` array →
`bq update --source=...` instead.

---

## 16. Shared VPC silently broke mid-build (most significant issue overall)

```
ERROR: (gcloud.compute.instances.create) Could not fetch resource:
 - Invalid value for field 'resource.networkInterfaces[0].network': '...'. The referenced network resource cannot be found.
```
then, after switching to a fully-qualified network path:
```
ERROR: (gcloud.compute.instances.create) Could not fetch resource:
 - Invalid value for field 'resource.networkInterfaces[0]': '{ "network": "..." }'. Cross-project references for this resource are not allowed.
```
then, after switching to `--subnet` only:
```
ERROR: (gcloud.compute.instances.create) Could not fetch resource:
 - Invalid value for field 'resource.networkInterfaces[0]': '{ "subnetwork": "..." }'. Cross-project references for this resource are not allowed.
```
then, after switching to `--network-interface=subnet=...,no-address`:
```
(same "Cross-project references for this resource are not allowed" error)
```
Four different syntactically-correct approaches, same error each time —
this ruled out a syntax problem and pointed at actual broken state.

**Diagnostic step 1:**
```bash
gcloud compute networks subnets list-usable --project="prj-lz-app-prod-${LZ_UNIQ}"
```
Result: only the auto-created `default` subnets in every region appeared —
none of the `vpc-lz-shared` subnets at all.

**Diagnostic step 2:**
```bash
gcloud compute shared-vpc get-host-project "prj-lz-app-prod-${LZ_UNIQ}"
```
Result: `{}` — empty. The attachment that had reportedly succeeded back in
Phase 1.6 was gone.

**Diagnostic step 3 — attempted re-attach:**
```
ERROR: (gcloud.compute.shared-vpc.associated-projects.add) Could not enable resource [prj-lz-app-prod-jd01] as an associated resource for project [prj-lz-net-host-jd01]:
 - Invalid resource usage: ''projects/prj-lz-net-host-jd01' is not a shared VPC host project.'
```
This revealed the real scope: not just the attachment, but the **host
project's Shared VPC-enabled status itself** had reverted.

**Diagnostic step 4 — attempted re-enable, error was truncated in terminal
output but referenced:**
```
...or it can be set temporarily by the environment variable [CLOUDSDK_CORE_PROJECT]
```

**Root cause, found via:**
```bash
gcloud config get-value project
```
Result: `(unset)`.

**Fix:**
```bash
gcloud config set project "prj-lz-net-host-${LZ_UNIQ}"
gcloud compute shared-vpc enable "prj-lz-net-host-${LZ_UNIQ}"
# succeeded this time
gcloud compute shared-vpc associated-projects add "prj-lz-app-prod-${LZ_UNIQ}" \
  --host-project="prj-lz-net-host-${LZ_UNIQ}"
gcloud compute shared-vpc associated-projects add "prj-lz-app-dev-${LZ_UNIQ}" \
  --host-project="prj-lz-net-host-${LZ_UNIQ}"
```
Confirmed via `list-usable` showing real subnets, then VM creation
succeeded on the next attempt with no further changes to the create
command itself.

**Key lesson:** explicit `--project=` flags on individual commands do not
fully substitute for a properly set active gcloud config project — some
multi-step operations (like `shared-vpc enable`) depend on the ambient
config state too.

---

## 17. Cloud SQL private IP — service networking not enabled on service project

```
ERROR: (gcloud.sql.instances.create) HTTPError 400: Invalid request: Incorrect Service Networking config for instance: ...SERVICE_NETWORKING_NOT_ENABLED.
```
**Cause:** `servicenetworking.googleapis.com` had only been enabled on the
host project (for the VPC peering range) — the service project where the
instance itself lives also needed it enabled.
**Fix:** enabled it on `prj-lz-app-prod` as well.

---

## 18. Cloud SQL PITR — wrong flag for database engine

```
ERROR: (gcloud.sql.instances.clone) HTTPError 400: Invalid request: Instance being cloned should have point in time recovery enabled or it was a primary instance with point in time recovery enabled before it switched to a replica.
```
**Cause:** `--enable-bin-log` (used in Phase 4.2) is MySQL-specific and
silently did nothing on a `POSTGRES_15` instance — no error at the time,
but PITR was never actually turned on.
**Fix:** used `--enable-point-in-time-recovery` instead, waited a few
minutes for transaction-log history to accumulate, then retried the clone
successfully.

---

## 19. Verification commands run from inside the wrong shell context

Multiple instances of `$LZ_UNIQ` resolving empty mid-session (producing
`prj-lz-app-prod-` with a trailing hyphen and downstream "invalid project
ID" errors), traced to:
- Cloud Shell sessions losing exported-but-not-persisted variables between
  reconnects
- one instance of a `gcloud` verification command being typed *inside* an
  active SSH session on a remote VM instead of back in Cloud Shell,
  producing a hang with no useful output
**Fix (recurring):** `echo "LZ_UNIQ is: [$LZ_UNIQ]"` as a standard first
check before any command block; `exit` to confirm returning to the correct
shell context (`whoami && hostname`) before running `gcloud` commands
against local infrastructure state.

---

## Error frequency by category

| Category | Count |
|---|---|
| Missing org-level IAM role | 4 (xpnAdmin, orgpolicy.policyAdmin, logging.admin, iap.tunnelResourceAccessor) |
| Empty/placeholder shell variable not substituted | 5 |
| Wrong/legacy gcloud command or flag syntax | 6 |
| Missing prerequisite resource (API, NAT, firewall rule, VM never created) | 5 |
| Cross-project resource reference handling | 4 (culminating in the Shared VPC root-cause chain) |
| Wrong config value for the specific database/service engine | 1 |
| Destination/service-account permission not granted | 3 |
