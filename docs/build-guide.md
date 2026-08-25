# Practice project: GCP hybrid landing zone (v3 — battle-tested)

Same lab as before — identity/networking, hybrid connectivity, security
baseline/tagging/logging, backup/recovery — rebuilt a third time with every
real blocker from the actual build session fixed at the source instead of
worked around after the fact. If you're running this fresh, this version
should go straight through Phase 1 without hitting the errors the earlier
versions did.

**What changed from v2, and why:**

| Fix | Why |
|---|---|
| Projects created with `--organization` from the start | v2 created standalone projects with no org parent, which blocked Shared VPC later with `The project has no organization` |
| `roles/compute.xpnAdmin` granted in Phase 1.1, before networking | v2 hit `Required 'compute.organizations.enableXpnHost' permission` because project Owner alone doesn't cover Shared VPC operations |
| IAM uses direct user bindings as the default path | v2's group-based bindings failed with `SOLO_GROUP_OWNERS_DISALLOWED` and `Group ... does not exist` — group creation needs a real verified domain and a Workspace admin-console step that's out of scope for a solo lab |
| VPN tunnel commands use the full resource URL for `--peer-gcp-gateway` | v2 omitted this, causing a 404 — the peer gateway lives in a different project than the one running the create command, and `--project` doesn't propagate to that flag |
| `gcloud dns managed-zones create` includes `--description` | v2 omitted this required flag, causing every zone creation to fail with `parameter is required but was missing` |
| Onprem-sim VM, Cloud NAT, and IAP SSH access added as explicit steps (2.0, new) | v2 referenced `vm-lz-onprem-us-central1-01` in the naming table but never gave the actual create command, the firewall rule for IAP, or a NAT gateway — all three were needed before DNS forwarding could actually be tested |
| Org Policy uses `gcloud org-policies set-policy` with the modern YAML schema for both constraints | v2's `resource-manager org-policies enable-enforce` / legacy `allValuesFromParent` schema failed outright — the modern `org-policies` command group doesn't have an `enable-enforce` subcommand at all; both boolean and list constraints go through `set-policy` with a `spec.rules` YAML |
| `roles/orgpolicy.policyAdmin` granted at the org level before applying constraints | Project Owner isn't enough for org policy operations, same pattern as the earlier `xpnAdmin` gap |
| Confirmed which of the two identically-named orgs (`deegee-ck-org`, IDs `43414815430` and `172919905395`) actually owns the lab projects, before granting any org-level roles | Two orgs sharing a display name meant an IAM grant could silently land on the wrong org and produce confusing downstream permission errors |
| Mandatory labels enforced directly via `gcloud alpha projects update --update-labels` on each project, not Terraform validation | The Terraform HCL block in v2's Phase 3.2 was never runnable in this session — this build is pure `gcloud` CLI with no Terraform project behind it, so it was pasted straight into bash and failed as invalid shell syntax; the stable-track `gcloud projects update` also doesn't support `--update-labels` at all, only the alpha track does |
| `roles/logging.admin` granted at the org level before creating sinks; BigQuery dataset and archive bucket created before the sinks that reference them; logging service account granted destination-level write access via `bq show`/`bq update` (BigQuery) and `gcloud storage buckets add-iam-policy-binding` (bucket) | Same permission-layering pattern as every prior org-level operation, plus: log sinks need their destination to already exist, and each sink writes as its own auto-created service account that needs explicit grants on that destination |
| Alert policy built via `--policy-from-file` JSON referencing a log-based metric with `resource.type="global"`, plus a real notification channel | `--condition-threshold-*` flags don't exist on the CLI; raw log fields aren't valid alert filter targets (need a metric first); and this particular metric registers under `global`, not the source log's resource type — confirmed via Metrics Explorer after two wrong guesses |
| Shared VPC instance creation uses `--subnet` with the full cross-project resource path — **and requires the active gcloud project to be explicitly set via `gcloud config set project`**, not just passed via `--project=` flags | The host project's Shared VPC "enabled" status and the service projects' "attached" status had both silently reverted at some point (confirmed via `get-host-project` returning `{}` and `associated-projects add` failing with `'...is not a shared VPC host project'`) — root cause traced to Cloud Shell's active project going `(unset)`, which broke `shared-vpc enable`'s internal state resolution even though every command still had an explicit `--project=` flag |
| Cloud SQL private IP requires `servicenetworking.googleapis.com` enabled on **both** the host project (for the peering range) and the service project (where the instance itself lives) | v2/v3 only enabled it on the host project; instance creation failed with `SERVICE_NETWORKING_NOT_ENABLED` until enabled on the service project too |
| Cloud SQL PITR uses `--enable-point-in-time-recovery`, not `--enable-bin-log` | `--enable-bin-log` is MySQL-specific; this instance is `POSTGRES_15`, so the original flag silently did nothing relevant, and PITR was never actually enabled until the correct flag was used — also needs a few minutes of transaction-log history to accumulate before a point-in-time clone will succeed |

---

## 0. Naming conventions (unchanged — still precise)

### The collision problem, and the fix

| Layer | Uniqueness scope | Reusable after delete? |
|---|---|---|
| Project **ID** (`--project-id`) | Global, across all GCP customers | No — reserved indefinitely / 30-day minimum |
| Project **display name** (`--name`) | Not unique at all | Yes, freely |
| GCS bucket name | Global, across all GCP customers | No — same problem as project ID |
| Everything else (VPC, subnet, VM, firewall rule, Cloud SQL instance, etc.) | Scoped to the project | Yes, immediately |

### Set once per session, persist it

```bash
export LZ_UNIQ="jd01"                       # your short, stable suffix
export ORG_ID="43414815430"                 # your org ID
export BILLING_ACCOUNT="01A629-7D8924-CDD7F4"

echo "export LZ_UNIQ=\"${LZ_UNIQ}\""        >> ~/.bashrc
echo "export ORG_ID=\"${ORG_ID}\""          >> ~/.bashrc
echo "export BILLING_ACCOUNT=\"${BILLING_ACCOUNT}\"" >> ~/.bashrc
source ~/.bashrc

echo "LZ_UNIQ=$LZ_UNIQ | ORG_ID=$ORG_ID | BILLING_ACCOUNT=$BILLING_ACCOUNT"
```

If you're rebuilding this lab again after a teardown, pick a **new**
`LZ_UNIQ` (old project IDs stay reserved) — e.g. `export LZ_UNIQ=$(date +%m%d)`.

### Pattern

`<resource-type-abbr>-<app>-<component>-<env>-<region>-<instance>`

Project IDs get `-${LZ_UNIQ}` appended; everything else is project-scoped
and reusable as-is.

### Project naming (globally unique — includes suffix)

| Project | Project ID |
|---|---|
| Shared VPC host | `prj-lz-net-host-${LZ_UNIQ}` |
| Prod service project | `prj-lz-app-prod-${LZ_UNIQ}` |
| Dev service project | `prj-lz-app-dev-${LZ_UNIQ}` |
| Onprem simulation | `prj-lz-onprem-${LZ_UNIQ}` |
| Shared services (logging/backup) | `prj-lz-shared-${LZ_UNIQ}` |

### Resource name table (project-scoped — no suffix needed)

| Resource | Name |
|---|---|
| Shared VPC | `vpc-lz-shared` |
| Prod subnet — web | `snet-lz-web-prod-us-central1` |
| Prod subnet — app | `snet-lz-app-prod-us-central1` |
| Prod subnet — data | `snet-lz-data-prod-us-central1` |
| Dev subnet | `snet-lz-app-dev-us-central1` |
| Onprem sim VPC | `vpc-lz-onprem` |
| Onprem sim subnet | `snet-lz-onprem-us-central1` |
| Firewall rule — allow web ingress | `fw-lz-allow-web-ingress-prod` |
| Firewall rule — app from web | `fw-lz-allow-app-from-web-prod` |
| Firewall rule — data from app | `fw-lz-allow-data-from-app-prod` |
| Cloud Router (hub side) | `rtr-lz-host-us-central1` |
| Cloud Router (onprem sim side) | `rtr-lz-onprem-us-central1` |
| HA VPN gateway (hub) | `vpn-lz-host-us-central1` |
| HA VPN gateway (onprem sim) | `vpn-lz-onprem-us-central1` |
| VPN tunnel | `tun-lz-host-to-onprem-us-central1` |
| Compute instance — web | `vm-lz-web-prod-us-central1-01` |
| Compute instance — onprem sim | `vm-lz-onprem-us-central1-01` |
| Cloud SQL instance | `sql-lz-app-prod-us-central1` |
| BigQuery log dataset | `ds_lz_logs_prod` |
| Backup vault (Backup and DR) | `bkv-lz-shared-us-central1` |
| IAM custom role | `custom.lz.deployer` |

### Globally unique — needs the suffix too

| Resource | Name |
|---|---|
| Logging archive bucket | `bkt-lz-logs-archive-${LZ_UNIQ}` |

### Mandatory labels

| Label key | Example value |
|---|---|
| `environment` | `prod`, `dev` |
| `owner` | lowercase, hyphenated (e.g. `you-example-com`) — no `@`/`.`/uppercase allowed |
| `project` | `hybrid-landing-zone` |
| `cost-center` | `personal-lab` |
| `data-classification` | `internal` |

---

## Phase 1 — Identity & core networking

### 1.1 Projects — created inside the org from the start

```bash
for proj in net-host app-prod app-dev onprem shared; do
  gcloud projects create "prj-lz-${proj}-${LZ_UNIQ}" \
    --name="prj-lz-${proj}" \
    --organization="${ORG_ID}" \
    --labels=environment=lab,project=hybrid-landing-zone,cost-center=personal-lab \
    || echo "FAILED: prj-lz-${proj}-${LZ_UNIQ}"
done
```

**Verify every project has the org as its parent (this is what v2 got wrong):**
```bash
for proj in net-host app-prod app-dev onprem shared; do
  gcloud projects describe "prj-lz-${proj}-${LZ_UNIQ}" \
    --format="value(projectId,parent.type,parent.id)"
done
```
Every row should show `organization ${ORG_ID}`.

### 1.2 Billing

```bash
gcloud billing projects list --billing-account="${BILLING_ACCOUNT}"

for proj in net-host app-prod app-dev onprem shared; do
  gcloud billing projects link "prj-lz-${proj}-${LZ_UNIQ}" \
    --billing-account="${BILLING_ACCOUNT}" \
    || echo "FAILED: prj-lz-${proj}-${LZ_UNIQ}"
done
```

### 1.3 Enable required APIs

```bash
for proj in net-host app-prod app-dev onprem shared; do
  gcloud services enable \
    compute.googleapis.com \
    dns.googleapis.com \
    logging.googleapis.com \
    sqladmin.googleapis.com \
    --project="prj-lz-${proj}-${LZ_UNIQ}" \
    || echo "FAILED: prj-lz-${proj}-${LZ_UNIQ}"
done
```

### 1.4 Grant yourself org-level Shared VPC admin now (this is what v2 missed until it broke Phase 1.3)

```bash
gcloud organizations add-iam-policy-binding "${ORG_ID}" \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/compute.xpnAdmin"
```

**Verify:**
```bash
gcloud organizations get-iam-policy "${ORG_ID}" \
  --flatten="bindings[].members" \
  --filter="bindings.members:$(gcloud config get-value account) AND bindings.role:xpnAdmin" \
  --format="table(bindings.role)"
```

### 1.5 IAM roles — direct user binding (default path for a solo lab)

```bash
gcloud projects add-iam-policy-binding "prj-lz-app-prod-${LZ_UNIQ}" \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/owner"

gcloud projects add-iam-policy-binding "prj-lz-app-dev-${LZ_UNIQ}" \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/editor"

gcloud iam roles create custom.lz.deployer \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --title="LZ Deployer" \
  --permissions=compute.instances.create,compute.instances.delete,compute.disks.create
```

> **If you want to practice group-based IAM instead** (the pattern worth
> describing in interviews even if you don't run it here): create real
> Cloud Identity groups under your org's verified domain first, at
> `admin.google.com` → Directory → Groups — `gcloud` alone can't create
> them. Note also that GCP disallows a group being the *sole* Owner on a
> project (`SOLO_GROUP_OWNERS_DISALLOWED`), so keep at least one individual
> user bound as Owner alongside any group.

### 1.6 Shared VPC — host and service projects

```bash
gcloud compute networks create vpc-lz-shared \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --subnet-mode=custom

gcloud compute networks subnets create snet-lz-web-prod-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --network=vpc-lz-shared --region=us-central1 --range=10.1.1.0/24

gcloud compute networks subnets create snet-lz-app-prod-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --network=vpc-lz-shared --region=us-central1 --range=10.1.2.0/24

gcloud compute networks subnets create snet-lz-data-prod-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --network=vpc-lz-shared --region=us-central1 --range=10.1.3.0/24

gcloud compute networks subnets create snet-lz-app-dev-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --network=vpc-lz-shared --region=us-central1 --range=10.2.1.0/24

gcloud compute shared-vpc enable "prj-lz-net-host-${LZ_UNIQ}"

gcloud compute shared-vpc associated-projects add "prj-lz-app-prod-${LZ_UNIQ}" \
  --host-project="prj-lz-net-host-${LZ_UNIQ}"

gcloud compute shared-vpc associated-projects add "prj-lz-app-dev-${LZ_UNIQ}" \
  --host-project="prj-lz-net-host-${LZ_UNIQ}"
```

**Verify:**
```bash
gcloud compute shared-vpc get-host-project "prj-lz-app-prod-${LZ_UNIQ}"
gcloud compute networks subnets list --project="prj-lz-net-host-${LZ_UNIQ}"
```

### 1.7 Firewall rules — deny-by-default per tier

```bash
gcloud compute firewall-rules create fw-lz-allow-web-ingress-prod \
  --project="prj-lz-net-host-${LZ_UNIQ}" --network=vpc-lz-shared \
  --direction=INGRESS --action=ALLOW --rules=tcp:443 \
  --source-ranges=0.0.0.0/0 --target-tags=lz-web-prod

gcloud compute firewall-rules create fw-lz-allow-app-from-web-prod \
  --project="prj-lz-net-host-${LZ_UNIQ}" --network=vpc-lz-shared \
  --direction=INGRESS --action=ALLOW --rules=tcp:8080 \
  --source-ranges=10.1.1.0/24 --target-tags=lz-app-prod

gcloud compute firewall-rules create fw-lz-allow-data-from-app-prod \
  --project="prj-lz-net-host-${LZ_UNIQ}" --network=vpc-lz-shared \
  --direction=INGRESS --action=ALLOW --rules=tcp:5432 \
  --source-ranges=10.1.2.0/24 --target-tags=lz-data-prod
```

**Verify:**
```bash
gcloud compute firewall-rules list --project="prj-lz-net-host-${LZ_UNIQ}" \
  --format="table(name,network,direction,sourceRanges.list(),allowed[].map().firewall_rule().list())"
```

---

## Phase 2 — Hybrid connectivity (simulated on-prem via Cloud VPN)

### 2.1 "On-prem" VPC — overlapping range on purpose

```bash
gcloud compute networks create vpc-lz-onprem \
  --project="prj-lz-onprem-${LZ_UNIQ}" --subnet-mode=custom

gcloud compute networks subnets create snet-lz-onprem-us-central1 \
  --project="prj-lz-onprem-${LZ_UNIQ}" \
  --network=vpc-lz-onprem --region=us-central1 --range=10.1.0.0/16
# Deliberately overlaps 10.1.0.0/16 used by the prod subnets — see 2.4.
```

### 2.1a The onprem-sim VM — actually create it (this step was missing entirely before)

The naming table lists `vm-lz-onprem-us-central1-01` as the DNS server for
the forwarding zone in 2.3, but the create command for it was never given
until it was needed. Do it here, before the VPN gateways, so it's ready
when DNS forwarding is configured.

```bash
gcloud compute instances create vm-lz-onprem-us-central1-01 \
  --project="prj-lz-onprem-${LZ_UNIQ}" \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --network=vpc-lz-onprem \
  --subnet=snet-lz-onprem-us-central1 \
  --no-address \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

`--no-address` keeps it private (no public IP), matching how a real
on-prem host would only be reachable internally — which is also why the
next two steps (NAT and IAP) are both required, not optional extras.

**Cloud NAT — required for outbound internet access (package installs, updates).**
Without this the VM has no route out and `apt-get` will fail with
`Network is unreachable`:

```bash
gcloud compute routers nats create nat-lz-onprem-us-central1 \
  --project="prj-lz-onprem-${LZ_UNIQ}" \
  --router=rtr-lz-onprem-us-central1 \
  --region=us-central1 \
  --auto-allocate-nat-external-ips \
  --nat-all-subnet-ip-ranges
```
> Note: this attaches to `rtr-lz-onprem-us-central1`, created in 2.2 below
> — if you're following this guide in order for the first time, create the
> Cloud Router in 2.2 first, then come back and run this NAT command.

**IAP-tunneled SSH — required since the VM has no public IP.** Three
separate things are needed (all missed in the original guide):

```bash
# a) Firewall rule allowing IAP's range to reach SSH
gcloud compute firewall-rules create fw-lz-allow-iap-ssh-onprem \
  --project="prj-lz-onprem-${LZ_UNIQ}" --network=vpc-lz-onprem \
  --direction=INGRESS --action=ALLOW --rules=tcp:22 \
  --source-ranges=35.235.240.0/20

# b) IAM role for IAP tunneling specifically (separate from SSH access)
gcloud projects add-iam-policy-binding "prj-lz-onprem-${LZ_UNIQ}" \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/iap.tunnelResourceAccessor"

# c) IAP API enabled on this project
gcloud services enable iap.googleapis.com --project="prj-lz-onprem-${LZ_UNIQ}"
```
Wait ~60 seconds after the IAM binding for it to propagate, then connect:
```bash
gcloud compute ssh vm-lz-onprem-us-central1-01 \
  --project="prj-lz-onprem-${LZ_UNIQ}" --zone=us-central1-a --tunnel-through-iap
```

**Install dnsmasq once connected:**
```bash
sudo apt-get update && sudo apt-get install -y dnsmasq
sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
exit
```

**Grab the internal IP — you'll need it for the DNS forwarding zone in 2.3:**
```bash
gcloud compute instances describe vm-lz-onprem-us-central1-01 \
  --project="prj-lz-onprem-${LZ_UNIQ}" --zone=us-central1-a \
  --format="value(networkInterfaces[0].networkIP)"
```

### 2.2 HA VPN gateways + Cloud Router (BGP) on both sides

```bash
gcloud compute routers create rtr-lz-host-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" --network=vpc-lz-shared \
  --region=us-central1 --asn=65001

gcloud compute vpn-gateways create vpn-lz-host-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" --network=vpc-lz-shared --region=us-central1

gcloud compute routers create rtr-lz-onprem-us-central1 \
  --project="prj-lz-onprem-${LZ_UNIQ}" --network=vpc-lz-onprem \
  --region=us-central1 --asn=65002

gcloud compute vpn-gateways create vpn-lz-onprem-us-central1 \
  --project="prj-lz-onprem-${LZ_UNIQ}" --network=vpc-lz-onprem --region=us-central1

# --peer-gcp-gateway needs the FULL resource URL when the peer gateway lives
# in a different project — plain names only work within the same project,
# and a bare name here fails with a 404 "resource not found" even though
# the gateway exists.
gcloud compute vpn-tunnels create tun-lz-host-to-onprem-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" --region=us-central1 \
  --vpn-gateway=vpn-lz-host-us-central1 \
  --peer-gcp-gateway="https://compute.googleapis.com/compute/v1/projects/prj-lz-onprem-${LZ_UNIQ}/regions/us-central1/vpnGateways/vpn-lz-onprem-us-central1" \
  --shared-secret="<psk>" --router=rtr-lz-host-us-central1 --interface=0

# The reverse tunnel is also required — HA VPN needs both directions
# established, and this was missing from earlier versions of this guide.
# Use the SAME <psk> value as above; it must match on both ends.
gcloud compute vpn-tunnels create tun-lz-onprem-to-host-us-central1 \
  --project="prj-lz-onprem-${LZ_UNIQ}" --region=us-central1 \
  --vpn-gateway=vpn-lz-onprem-us-central1 \
  --peer-gcp-gateway="https://compute.googleapis.com/compute/v1/projects/prj-lz-net-host-${LZ_UNIQ}/regions/us-central1/vpnGateways/vpn-lz-host-us-central1" \
  --shared-secret="<psk>" --router=rtr-lz-onprem-us-central1 --interface=0
```

**Verify both tunnels establish:**
```bash
gcloud compute vpn-tunnels describe tun-lz-host-to-onprem-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" --region=us-central1 \
  --format="value(status)"
```
Look for `ESTABLISHED`.

### 2.3 Cloud DNS — conditional forwarding

```bash
# --description is REQUIRED — omitting it fails with
# "entity.managedZone.description parameter is required but was missing"
gcloud dns managed-zones create lz-internal \
  --project="prj-lz-net-host-${LZ_UNIQ}" --dns-name="lz.internal." \
  --networks=vpc-lz-shared --visibility=private \
  --description="Private zone for internal landing zone namespace"

# Use the real internal IP captured at the end of 2.1a, not a placeholder —
# Cloud DNS validates this is a real IP and will reject literal text.
gcloud dns managed-zones create lz-onprem-forward \
  --project="prj-lz-net-host-${LZ_UNIQ}" --dns-name="onprem.lz.internal." \
  --networks=vpc-lz-shared --visibility=private \
  --forwarding-targets="<paste-onprem-vm-internal-ip-from-2.1a>" \
  --description="Forwarding zone to simulated on-prem DNS server"
```

**Verify both zones exist:**
```bash
gcloud dns managed-zones list --project="prj-lz-net-host-${LZ_UNIQ}"
```
You should see `lz-internal` and `lz-onprem-forward`, both `PRIVATE`.

### 2.4 The overlap problem — how to resolve it

1. Bring up the tunnel/BGP session — it connects fine, but routes from the
   overlapping `10.1.0.0/16` onprem range collide with the existing prod
   subnet routes in the host VPC, causing intermittent misrouting.
2. Fix, in order of preference:
   - **Re-IP** the onprem-sim subnet to a non-overlapping range (e.g.
     `10.99.0.0/16`) — cleanest for a lab.
   - If re-IP isn't possible, use **Cloud NAT** or a Compute Engine NAT
     instance to translate the overlapping range, or set custom route
     priorities on the Cloud Router.
3. Verify with:
   ```bash
   gcloud compute routers get-status rtr-lz-host-us-central1 \
     --project="prj-lz-net-host-${LZ_UNIQ}" --region=us-central1
   ```

---

## Phase 3 — Security baseline, labeling, logging

### 3.1 Organization Policy constraints

**Prerequisite — confirm you're granting roles on the correct org.** If your
org has more than one entry with the same display name (check with
`gcloud organizations list`), confirm which one actually owns your
projects before granting anything:
```bash
gcloud projects describe "prj-lz-app-prod-${LZ_UNIQ}" \
  --format="value(parent.type,parent.id)"
```
Set `ORG_ID` to that literal value, then:
```bash
gcloud organizations add-iam-policy-binding "${ORG_ID}" \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/orgpolicy.policyAdmin"
```
Wait ~60 seconds for propagation.

**Apply both constraints via `set-policy`** — this is the only working
command in the modern `org-policies` group; there is no `enable-enforce`
subcommand, and the legacy `resource-manager org-policies` schema
(`allValuesFromParent`) is rejected by the current API:

```bash
echo "prj-lz-app-prod-${LZ_UNIQ}"   # confirm exact project ID string first
```

```bash
cat > policy-require-oslogin.yaml << 'EOF'
name: projects/<paste-exact-project-id>/policies/compute.requireOsLogin
spec:
  rules:
  - enforce: true
EOF

gcloud org-policies set-policy policy-require-oslogin.yaml \
  --project="prj-lz-app-prod-${LZ_UNIQ}"
```

```bash
cat > policy-no-external-ip.yaml << 'EOF'
name: projects/<paste-exact-project-id>/policies/compute.vmExternalIpAccess
spec:
  rules:
  - denyAll: true
EOF

gcloud org-policies set-policy policy-no-external-ip.yaml \
  --project="prj-lz-app-prod-${LZ_UNIQ}"
```

**Verify both:**
```bash
gcloud org-policies describe compute.requireOsLogin \
  --project="prj-lz-app-prod-${LZ_UNIQ}"

gcloud org-policies describe compute.vmExternalIpAccess \
  --project="prj-lz-app-prod-${LZ_UNIQ}"
```
Confirm `rules: - enforce: true` and `rules: - denyAll: true` respectively.

### 3.2 Enforce mandatory labels — applied directly via gcloud

This build is pure `gcloud` CLI with no Terraform project behind it, so
labels are set directly on each project rather than validated through a
Terraform variable block (that approach only applies if you're actually
running `terraform apply` — see the note at the end of this section if you
want to add that separately for portfolio purposes).

**Note:** `--update-labels` is alpha-track only on `gcloud projects update`
— accept the alpha component install prompt if asked, it's safe.

```bash
for proj in net-host app-prod app-dev onprem shared; do
  gcloud alpha projects update "prj-lz-${proj}-${LZ_UNIQ}" \
    --update-labels=environment=lab,owner=deegee-ck,project=hybrid-landing-zone,cost-center=personal-lab,data-classification=internal
done
```

**Verify all 5:**
```bash
for proj in net-host app-prod app-dev onprem shared; do
  echo "prj-lz-${proj}-${LZ_UNIQ}:"
  gcloud projects describe "prj-lz-${proj}-${LZ_UNIQ}" --format="value(labels)"
done
```
Each should list all 5 keys: `cost-center`, `data-classification`,
`environment`, `owner`, `project`.

> **If you want a Terraform validation example for your portfolio anyway**
> (worth having as IaC evidence even without a live Terraform project
> behind this build), write it to a file rather than pasting into bash:
> ```bash
> cat > variables.tf << 'EOF'
> variable "labels" {
>   type = map(string)
>   validation {
>     condition = alltrue([
>       for k in ["environment", "owner", "project", "cost-center", "data-classification"] :
>       contains(keys(var.labels), k)
>     ])
>     error_message = "All resources must include environment, owner, project, cost-center, data-classification labels."
>   }
> }
> EOF
> ```
> This just creates the file for reference/repo purposes — it doesn't
> validate anything unless it's part of an actual Terraform project.

### 3.3 Centralized logging

**Prerequisite — org-level role:**
```bash
gcloud organizations add-iam-policy-binding "${ORG_ID}" \
  --member="user:$(gcloud config get-value account)" \
  --role="roles/logging.admin"
```
Wait ~60 seconds for propagation.

**Create the destinations before the sinks that reference them** — the
original guide assumed these already existed:
```bash
bq mk --project_id="prj-lz-shared-${LZ_UNIQ}" --dataset ds_lz_logs_prod

gcloud storage buckets create "gs://bkt-lz-logs-archive-${LZ_UNIQ}" \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --location=us-central1
```

**Create the sinks** (use `--quiet` to skip the interactive filter-warning
prompt, which is misleading here since a real filter is set):
```bash
gcloud logging sinks create sink-lz-logs-to-bq \
  "bigquery.googleapis.com/projects/prj-lz-shared-${LZ_UNIQ}/datasets/ds_lz_logs_prod" \
  --organization="${ORG_ID}" --include-children \
  --log-filter='resource.type="gce_instance" OR resource.type="gce_firewall_rule"' \
  --quiet

gcloud logging sinks create sink-lz-logs-to-archive \
  "storage.googleapis.com/bkt-lz-logs-archive-${LZ_UNIQ}" \
  --organization="${ORG_ID}" --include-children --quiet
```

Each `create` command prints the exact service account it needs granted on
the destination — read that output rather than assuming; it follows the
pattern `service-org-<ORG_ID>@gcp-sa-logging.iam.gserviceaccount.com`.

**Grant that service account write access on the archive bucket** (this one
works via the standard command):
```bash
gcloud storage buckets add-iam-policy-binding "gs://bkt-lz-logs-archive-${LZ_UNIQ}" \
  --member="serviceAccount:service-org-${ORG_ID}@gcp-sa-logging.iam.gserviceaccount.com" \
  --role="roles/storage.objectCreator"
```

**Grant it write access on the BigQuery dataset** — `bq add-iam-policy-binding`
is allowlist-gated on some orgs and fails with `This feature requires
allowlisting`. Use a `bq show`/`bq update` JSON patch instead:
```bash
bq show --format=prettyjson "prj-lz-shared-${LZ_UNIQ}:ds_lz_logs_prod" > dataset-access.json
```
Edit `dataset-access.json` and add this object into the `"access"` array:
```json
{
  "role": "WRITER",
  "userByEmail": "service-org-<ORG_ID>@gcp-sa-logging.iam.gserviceaccount.com"
}
```
(substitute your literal `${ORG_ID}` value)
```bash
bq update --source=dataset-access.json "prj-lz-shared-${LZ_UNIQ}:ds_lz_logs_prod"
```
> If editing JSON in Cloud Shell is awkward, the Console UI is faster here:
> BigQuery → project `prj-lz-shared-${LZ_UNIQ}` → dataset
> `ds_lz_logs_prod` → **Sharing → Permissions → Add Principal** → paste the
> service account → role **BigQuery Data Editor** → Save.

**Verify everything:**
```bash
gcloud logging sinks list --organization="${ORG_ID}"

bq show --format=prettyjson "prj-lz-shared-${LZ_UNIQ}:ds_lz_logs_prod" \
  | grep -A2 "gcp-sa-logging"
```

### 3.4 One alert on top of the logs

The flag-based command in earlier versions of this guide doesn't exist —
`gcloud alpha monitoring policies create` only accepts a full policy
definition via `--policy-from-file`, and alert conditions can only filter
on metrics (not raw log fields), so a log-based metric has to exist first.

**1. Create a notification channel (email, simplest for a lab):**
```bash
gcloud alpha monitoring channels create \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --display-name="lz-lab-email-alert" \
  --type=email \
  --channel-labels=email_address=<your-real-email>

CHANNEL_NAME=$(gcloud alpha monitoring channels list \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --format="value(name)")
echo "$CHANNEL_NAME"
```

**2. Create a log-based metric** — alert conditions can't filter on raw log
fields like `jsonPayload.disposition` directly (rejected with `The
lefthand side of each expression must be prefixed with one of {group,
metadata, metric, project, resource}`), so turn the log filter into a
counter metric first:
```bash
gcloud logging metrics create lz-firewall-deny-count \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --description="Counts firewall deny events across the landing zone" \
  --log-filter='resource.type="gce_firewall_rule" AND jsonPayload.disposition="DENIED"'
```
Newly created metrics take a few minutes to register as a valid
descriptor — wait before referencing it in a policy.

**3. Confirm which resource type the metric actually registers under.**
Don't assume it matches the source log's resource type (it often doesn't)
— check via the Console: Monitoring → Metrics Explorer → search for
`lz-firewall-deny-count` → note the resource type shown (in this build it
came back `global`, not `gce_firewall_rule`).

**4. Write the alert policy JSON**, using the real channel name from step 1
and the confirmed resource type from step 3:
```bash
cat > alert-policy.json << 'EOF'
{
  "displayName": "alert-lz-firewall-deny-spike",
  "combiner": "OR",
  "conditions": [
    {
      "displayName": "Firewall deny spike",
      "conditionThreshold": {
        "filter": "metric.type=\"logging.googleapis.com/user/lz-firewall-deny-count\" AND resource.type=\"global\"",
        "comparison": "COMPARISON_GT",
        "thresholdValue": 50,
        "duration": "300s",
        "aggregations": [
          {
            "alignmentPeriod": "300s",
            "perSeriesAligner": "ALIGN_SUM"
          }
        ]
      }
    }
  ],
  "notificationChannels": [
    "<paste-real-channel-name-from-step-1>"
  ]
}
EOF

sed -i "s|<paste-real-channel-name-from-step-1>|${CHANNEL_NAME}|" alert-policy.json
```

**5. Create the policy:**
```bash
gcloud alpha monitoring policies create \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --policy-from-file=alert-policy.json
```

**Verify:**
```bash
gcloud alpha monitoring policies list \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --format="table(displayName,enabled)"
```

Note: since this is a brand-new custom metric, it won't have data points
until at least one matching firewall-deny event actually occurs — the
policy still creates and enables successfully either way.

### 3.5 Compliance verification

```bash
gcloud org-policies list --project="prj-lz-app-prod-${LZ_UNIQ}"
```
Confirms both constraints from 3.1 are set (`compute.requireOsLogin` as
`BOOLEAN_POLICY: SET`, `compute.vmExternalIpAccess` as `LIST_POLICY: SET`).

```bash
gcloud scc findings list "<scc-source-id>" --filter='state="ACTIVE"'
```
Only relevant if Security Command Center is enabled on this org — optional
for a lab, skip if you don't have SCC Premium.

---

## Phase 4 — Backup & recovery

### 4.0 Prerequisite — active project must be set explicitly

Given the Shared VPC breakage traced to Cloud Shell's active project going
`(unset)` earlier in this build, make a habit of confirming this at the
start of any new session before running anything in Phase 4:
```bash
gcloud config set project "prj-lz-app-prod-${LZ_UNIQ}"
gcloud config get-value project   # confirm it's not (unset)
```

### 4.1 Deploy the minimal 3-tier app

**Web and app tier VMs** — Shared VPC subnets need the full cross-project
resource path via `--subnet`, and no separate `--network` flag (combining
both causes `Cross-project references for this resource are not allowed`):

```bash
gcloud compute instances create vm-lz-web-prod-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --subnet="projects/prj-lz-net-host-${LZ_UNIQ}/regions/us-central1/subnetworks/snet-lz-web-prod-us-central1" \
  --tags=lz-web-prod \
  --no-address \
  --image-family=debian-12 \
  --image-project=debian-cloud

gcloud compute instances create vm-lz-app-prod-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --zone=us-central1-a \
  --machine-type=e2-small \
  --subnet="projects/prj-lz-net-host-${LZ_UNIQ}/regions/us-central1/subnetworks/snet-lz-app-prod-us-central1" \
  --tags=lz-app-prod \
  --no-address \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

> **If this fails with the cross-project error even using this exact
> syntax**, don't keep tweaking flags — check the underlying Shared VPC
> attachment first:
> ```bash
> gcloud compute shared-vpc get-host-project "prj-lz-app-prod-${LZ_UNIQ}"
> ```
> If this returns `{}` instead of real host project details, the
> attachment (or the host's Shared VPC "enabled" status itself) has
> reverted. Re-enable and re-attach:
> ```bash
> gcloud config set project "prj-lz-net-host-${LZ_UNIQ}"
> gcloud compute shared-vpc enable "prj-lz-net-host-${LZ_UNIQ}"
> gcloud compute shared-vpc associated-projects add "prj-lz-app-prod-${LZ_UNIQ}" \
>   --host-project="prj-lz-net-host-${LZ_UNIQ}"
> gcloud compute shared-vpc associated-projects add "prj-lz-app-dev-${LZ_UNIQ}" \
>   --host-project="prj-lz-net-host-${LZ_UNIQ}"
> ```
> Confirm subnets are visible before retrying VM creation:
> ```bash
> gcloud compute networks subnets list-usable --project="prj-lz-app-prod-${LZ_UNIQ}"
> ```
> You should see `vpc-lz-shared` subnets alongside the auto-created
> `default` ones — if only `default` appears, the attachment still isn't
> real.

**Cloud SQL** — private IP requires `servicenetworking.googleapis.com`
enabled on **both** the host project (done in the Private Services Access
step below) and the **service project** where the instance itself lives —
missing the second one fails with `SERVICE_NETWORKING_NOT_ENABLED`:

```bash
# Private Services Access — host project side (prerequisite for private IP)
gcloud services enable servicenetworking.googleapis.com \
  --project="prj-lz-net-host-${LZ_UNIQ}"

gcloud compute addresses create ps-connect-lz-shared \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --global --purpose=VPC_PEERING --prefix-length=16 \
  --network=vpc-lz-shared

gcloud services vpc-peerings connect \
  --project="prj-lz-net-host-${LZ_UNIQ}" \
  --service=servicenetworking.googleapis.com \
  --ranges=ps-connect-lz-shared \
  --network=vpc-lz-shared

# ALSO required — the service project, not just the host
gcloud services enable servicenetworking.googleapis.com \
  --project="prj-lz-app-prod-${LZ_UNIQ}"

gcloud sql instances create sql-lz-app-prod-us-central1 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1 \
  --network="projects/prj-lz-net-host-${LZ_UNIQ}/global/networks/vpc-lz-shared" \
  --no-assign-ip
```

**Verify:**
```bash
gcloud compute instances list --project="prj-lz-app-prod-${LZ_UNIQ}"

gcloud sql instances describe sql-lz-app-prod-us-central1 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --format="value(state,ipAddresses)"
```

### 4.2 Scheduled snapshots + Cloud SQL PITR

```bash
gcloud compute resource-policies create snapshot-schedule snap-lz-web-daily \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --region=us-central1 \
  --max-retention-days=7 --daily-schedule --start-time=03:00

gcloud compute disks add-resource-policies vm-lz-web-prod-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --resource-policies=snap-lz-web-daily
```

**Point-in-time recovery — use the correct flag for your database engine.**
`--enable-bin-log` is MySQL-only; this instance is Postgres, which needs
`--enable-point-in-time-recovery` instead. Using the wrong flag doesn't
error — it just silently does nothing, and PITR stays disabled until the
correct flag is used:
```bash
gcloud sql instances patch sql-lz-app-prod-us-central1 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --enable-point-in-time-recovery
```

**Verify:**
```bash
gcloud compute disks describe vm-lz-web-prod-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --format="value(resourcePolicies)"
# expect: snap-lz-web-daily

gcloud sql instances describe sql-lz-app-prod-us-central1 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --format="value(settings.backupConfiguration.pointInTimeRecoveryEnabled)"
# expect: True
```
PITR needs a few minutes of transaction-log history to accumulate before a
point-in-time clone will succeed — don't attempt 4.3's SQL restore
immediately after enabling this.

### 4.3 Do a real restore

**VM disk — snapshot, restore, verify it actually boots and is reachable:**
```bash
gcloud compute disks snapshot vm-lz-web-prod-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --snapshot-names="snap-lz-web-manual-$(date +%Y%m%d)"

gcloud compute disks create vm-lz-web-restore-test-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --source-snapshot="snap-lz-web-manual-$(date +%Y%m%d)"

gcloud compute instances create vm-lz-restore-test-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --disk=name=vm-lz-web-restore-test-01,boot=yes
```

**Confirm it's genuinely reachable, not just that the disk exists** — SSH
in via IAP and check a basic command runs:
```bash
gcloud compute ssh vm-lz-restore-test-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a --tunnel-through-iap \
  --command="echo restored VM is reachable and SSH works"
```
This is the piece that actually proves recoverability — a `READY` disk
status alone doesn't confirm the service comes back up.

**Get real elapsed-time evidence for your RTO answer:**
```bash
echo "Snapshot created:" && gcloud compute snapshots describe \
  "snap-lz-web-manual-$(date +%Y%m%d)" \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --format="value(creationTimestamp)"

echo "Restored disk created:" && gcloud compute disks describe \
  vm-lz-web-restore-test-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --format="value(creationTimestamp)"

echo "Restored instance created:" && gcloud compute instances describe \
  vm-lz-restore-test-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a \
  --format="value(creationTimestamp)"
```

**Cloud SQL point-in-time restore** — wait at least a few minutes after
enabling PITR in 4.2 before running this, and use a timestamp after PITR
was enabled:
```bash
gcloud sql instances clone sql-lz-app-prod-us-central1 sql-lz-app-restore-test \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --point-in-time="$(date -u -d '2 minutes ago' +%Y-%m-%dT%H:%M:%SZ)"
```

### 4.4 Clean up between attempts

```bash
gcloud compute instances delete vm-lz-restore-test-us-central1-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a --quiet

gcloud compute disks delete vm-lz-web-restore-test-01 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --zone=us-central1-a --quiet

gcloud sql instances delete sql-lz-app-restore-test \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --quiet
```

---

## Mapping back to the four interview questions

| Question | What in this project answers it |
|---|---|
| Complex workload — identity & networking | Phase 1: Shared VPC host/service projects, per-tier firewall rules, direct/least-privilege IAM bindings |
| Hybrid connectivity + challenges | Phase 2: HA VPN + Cloud Router BGP to simulated on-prem, Cloud DNS forwarding, the deliberate overlap you diagnosed and fixed — **plus the real org-parent/xpnAdmin issues from Phase 1, the cross-project peer-gateway 404, the missing reverse tunnel, and the private-VM-needs-NAT-and-IAP chain you actually hit and resolved**, which is stronger material precisely because it's unscripted |
| Security baselines, tagging, logging + compliance | Phase 3: Org Policy constraints (fixed twice — first for the wrong command syntax, then for a dual-org permission mix-up), label-enforcement via Terraform validation, aggregated log sinks, Org Policy/SCC compliance review |
| Backup/recovery + RTO/RPO | Phase 4: scheduled snapshots + Cloud SQL PITR (with the MySQL/Postgres flag mismatch caught and fixed), a real timed restore into an isolated resource verified via actual SSH reachability — not just a "disk exists" check — **plus the Shared VPC attachment silently reverting mid-build, traced back to Cloud Shell's active project going unset**, which is arguably the single best troubleshooting story from this whole project |

---

## Teardown

```bash
for proj in net-host app-prod app-dev onprem shared; do
  gcloud billing projects unlink "prj-lz-${proj}-${LZ_UNIQ}"
done

for proj in net-host app-prod app-dev onprem shared; do
  gcloud projects delete "prj-lz-${proj}-${LZ_UNIQ}" --quiet
done
```

If you rebuild again, pick a **new** `LZ_UNIQ` — old project IDs stay
reserved through at least the 30-day grace period.

---

## Azure ↔ GCP concept map (quick reference)

| Concept | Azure | GCP |
|---|---|---|
| Network isolation boundary | Subscription / Resource Group | Project |
| Shared network topology | Hub-spoke VNet peering | Shared VPC (host/service projects) |
| Site-to-site tunnel | VPN Gateway + Local Network Gateway | HA VPN + Cloud Router (BGP) |
| Private connectivity to on-prem (dedicated circuit) | ExpressRoute | Dedicated/Partner Interconnect |
| Segmentation firewalling | NSG | VPC Firewall Rules |
| Metadata for cost/ownership | Tags | Labels |
| Guardrail policy engine | Azure Policy | Organization Policy / Custom Constraints |
| Centralized log store | Log Analytics Workspace | Cloud Logging sinks → BigQuery/GCS |
| Backup/DR | Recovery Services Vault | Backup and DR Service / disk snapshots |
| Identity | Entra ID (Azure AD) | Cloud Identity / Google Workspace + IAM |
| Resource ID uniqueness | Names unique per resource group only | Project IDs & bucket names unique **globally, forever** |
| Cross-project network admin permission | Owner/Contributor at subscription scope is usually enough | Needs **`roles/compute.xpnAdmin`** at the org level separately from project Owner |
