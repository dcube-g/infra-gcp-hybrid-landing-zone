# Screenshot capture guide — hybrid landing zone portfolio

Matches the `docs/screenshots/` structure from `README-portfolio-template.md`.
For each item: **what to capture**, **exact navigation or command**, and
**what to redact** before it goes anywhere public.

> **Redact before saving, every time:** your project IDs (`prj-lz-*-jd01`),
> org ID (`43414815430`), billing account ID (`01A629-7D8924-CDD7F4`), and
> personal email — blur/black-box them in the image itself, don't just crop
> around them (metadata and adjacent UI elements can still reveal them).

---

## Console (graphical) screenshots

### 01 — Shared VPC topology
**Navigate:** Console → VPC network → **Shared VPC** (left sidebar, under
VPC network) → select `prj-lz-net-host-jd01` as host
**Capture:** the page showing the host project with `app-prod` and
`app-dev` listed as attached service projects, plus the subnet list
(`snet-lz-web-prod-us-central1`, etc.)
**Redact:** project ID in the URL bar and breadcrumb

### 02 — IAM bindings
**Navigate:** Console → IAM & Admin → IAM, on `prj-lz-app-prod-jd01`
**Capture:** the table showing your user with `Owner` and the custom
`LZ Deployer` role
**Redact:** your email address (blur it, keep the role names visible)

### 03 — Firewall rules
**Navigate:** Console → VPC network → Firewall, on `prj-lz-net-host-jd01`
**Capture:** all three `fw-lz-*` rules with their port/source columns visible
**Redact:** project ID in breadcrumb

### 04 — VPN tunnel established
**Navigate:** Console → Hybrid Connectivity → VPN → Cloud VPN tunnels
**Capture:** both tunnels (`tun-lz-host-to-onprem...`,
`tun-lz-onprem-to-host...`) showing status **Established** — this is one of
your strongest images, the green checkmark tells the whole story at a glance
**Redact:** project ID, peer IP addresses if visible

### 05 — Cloud Router BGP session
**Navigate:** Console → Hybrid Connectivity → VPN → click a tunnel → BGP
session details
**Capture:** the session status and route exchange count

### 06 — Org Policy constraints
**Navigate:** Console → IAM & Admin → Organization Policies, on
`prj-lz-app-prod-jd01`, filter to `compute.requireOsLogin` and
`compute.vmExternalIpAccess`
**Capture:** both showing as customized/enforced (not "Inherit from parent")

### 07 — Logging sinks
**Navigate:** Console → Logging → Log Router, at the organization level
**Capture:** both `sink-lz-logs-to-bq` and `sink-lz-logs-to-archive` listed
with their destinations
**Redact:** org ID and project IDs visible in the destination paths

### 08 — Metrics Explorer showing the custom metric
**Navigate:** Console → Monitoring → Metrics Explorer → search
`lz-firewall-deny-count`
**Capture:** the resource-type confirmation screen (`Global`) — this is the
one that shows you actually diagnosed the resource-type mismatch, good
evidence for the troubleshooting story

### 09 — Alert policy
**Navigate:** Console → Monitoring → Alerting → Policies
**Capture:** `alert-lz-firewall-deny-spike` listed as Enabled

### 10 — Instances list (3-tier app)
**Navigate:** Console → Compute Engine → VM instances, on
`prj-lz-app-prod-jd01`
**Capture:** `vm-lz-web-prod-us-central1-01` and
`vm-lz-app-prod-us-central1-01`, both running, both with no external IP
shown

### 11 — Cloud SQL private IP
**Navigate:** Console → SQL → `sql-lz-app-prod-us-central1` → Overview
**Capture:** the connection panel showing a private IP only, no public IP
**Redact:** the private IP itself is fine to show (it's internal-only and
meaningless outside your VPC)

### 12 — Backup/restore timing (once you get to 4.3)
**Navigate:** Console → SQL → instance → Backups, or Compute Engine →
Snapshots
**Capture:** timestamps of the backup and the restored resource — crop or
annotate to show elapsed time clearly, this is your RTO evidence

---

## CLI (terminal) screenshots or copy-pasted output

For CLI evidence, either screenshot the terminal directly or copy the
output into a code block in `troubleshooting.md` — text is often more
useful than an image here since it's searchable and copy-pasteable for
whoever reviews your portfolio.

### Verification commands worth capturing (all read-only, safe to run again anytime)

```bash
# Shared VPC attachment
gcloud compute shared-vpc get-host-project "prj-lz-app-prod-${LZ_UNIQ}"

# Firewall rules
gcloud compute firewall-rules list --project="prj-lz-net-host-${LZ_UNIQ}"

# VPN tunnel status
gcloud compute vpn-tunnels describe tun-lz-host-to-onprem-us-central1 \
  --project="prj-lz-net-host-${LZ_UNIQ}" --region=us-central1 \
  --format="value(status)"

# Org policies applied
gcloud org-policies list --project="prj-lz-app-prod-${LZ_UNIQ}"

# Labels on all projects
for proj in net-host app-prod app-dev onprem shared; do
  echo "prj-lz-${proj}-${LZ_UNIQ}:"
  gcloud projects describe "prj-lz-${proj}-${LZ_UNIQ}" --format="value(labels)"
done

# Logging sinks
gcloud logging sinks list --organization="${ORG_ID}"

# Alert policy
gcloud alpha monitoring policies list \
  --project="prj-lz-shared-${LZ_UNIQ}" \
  --format="table(displayName,enabled)"

# Running instances
gcloud compute instances list --project="prj-lz-app-prod-${LZ_UNIQ}"

# Cloud SQL private IP confirmation
gcloud sql instances describe sql-lz-app-prod-us-central1 \
  --project="prj-lz-app-prod-${LZ_UNIQ}" --format="value(ipAddresses)"
```

**Before screenshotting a terminal**, run this to swap the visible prompt
for something generic, so your Cloud Shell username/project number doesn't
leak into the image:
```bash
export PS1="lz-lab$ "
```
(only affects display, resets when you close the session)

**For the redaction-prone commands** (anything printing `prj-lz-*-jd01`
literally), pipe through `sed` to mask before you screenshot or paste:
```bash
gcloud compute instances list --project="prj-lz-app-prod-${LZ_UNIQ}" \
  | sed 's/jd01/****/g'
```

---

## Suggested capture order (roughly 30–40 minutes total)

1. Console: 01, 02, 03, 06, 10, 11 (Phase 1/3/4 static state — quick batch)
2. Console: 04, 05 (VPN — the strongest visual)
3. Console: 07, 08, 09 (logging/alerting chain)
4. CLI: run the full verification block above in one terminal session,
   screenshot or copy the output
5. Once Phase 4.3 (restore) is done: Console 12 for the RTO evidence

Save everything into `docs/screenshots/` using the numbered names above —
they'll sort correctly in a file browser and match the README references.
