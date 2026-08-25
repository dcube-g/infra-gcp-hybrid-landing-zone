# GCP lab: progress log and next steps (updated)

Session values confirmed working — reuse these for every command below:
```bash
export LZ_UNIQ="jd01"
export BILLING_ACCOUNT="01A629-7D8924-CDD7F4"
```

## Progress so far

- [x] **Step 1** — Authenticated as `deegee.ck@gmail.com`
- [x] **Step 2** — `LZ_UNIQ=jd01` set and persisted
- [x] **Step 3** — Org confirmed (`deegee-ck-org`, ID `43414815430`)
- [x] **Step 4** — `prj-lz-net-host-jd01` created
- [x] **Step 5** — Remaining 4 projects created (`app-prod`, `app-dev`, `onprem`, `shared`)
- [x] **Step 6** — Billing account confirmed (`01A629-7D8924-CDD7F4`, open, room for 5)
- [x] **Step 7** — All 5 projects linked to billing successfully
- [x] **Step 8** — Required APIs enabled on all 5 projects
- [x] **Phase 1.2** — IAM roles bound (user-binding path — see note below on why groups weren't used)
- [x] **Phase 1.3** — Shared VPC host/service projects (see note below on org-move + xpnAdmin needed)
- [x] **Phase 1.4** — Firewall rules (all 3 confirmed on `vpc-lz-shared`)

**Phase 1 complete.** Next: Phase 2 — Hybrid connectivity ← **you are here**

---

## Step 8 — Enable required APIs

```bash
for proj in net-host app-prod app-dev onprem shared; do
  echo "Enabling APIs on prj-lz-${proj}-${LZ_UNIQ}..."
  gcloud services enable \
    compute.googleapis.com \
    dns.googleapis.com \
    logging.googleapis.com \
    sqladmin.googleapis.com \
    --project="prj-lz-${proj}-${LZ_UNIQ}" \
    || echo "FAILED: prj-lz-${proj}-${LZ_UNIQ}"
done
```

**Verify:**
```bash
gcloud services list --enabled --project="prj-lz-net-host-${LZ_UNIQ}" \
  --filter="config.name:compute.googleapis.com OR config.name:dns.googleapis.com"
```
Expect both `compute.googleapis.com` and `dns.googleapis.com` listed. Repeat
the check for any other project if you want extra confidence:
```bash
gcloud services list --enabled --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --filter="config.name:compute.googleapis.com"
```

---

## Phase 1.2 — IAM roles (user-binding path — confirmed working)

**Decision made:** group-based bindings were attempted first and hit two
blockers, so this lab uses direct user bindings instead. Documented here so
the reasoning survives if you revisit this later:

- `grp-lz-prod-owners@example.com` failed with `SOLO_GROUP_OWNERS_DISALLOWED`
  — GCP refuses to let a group be the *sole* Owner on a project, as a
  safeguard against getting locked out if the group is later misconfigured.
- `grp-lz-dev-editors@example.com` failed with `Group ... does not exist` —
  `example.com` was always a placeholder domain in this guide, never meant
  to be typed literally. Real Cloud Identity groups have to be created
  under your org's actual verified domain (`deegee-ck-org`) via the
  Workspace/Admin console first — `gcloud` alone can't create them, and for
  a solo lab that detour isn't worth it.

**What's actually running — bind roles directly to your user:**

```bash
gcloud projects add-iam-policy-binding "prj-lz-app-prod-${LZ_UNIQ}" \
  --member="user:deegee.ck@gmail.com" \
  --role="roles/owner"

gcloud projects add-iam-policy-binding "prj-lz-app-dev-${LZ_UNIQ}" \
  --member="user:deegee.ck@gmail.com" \
  --role="roles/editor"
```

> If you want to talk through group-based IAM in an interview despite not
> running it live here, it's still fair to describe as the recommended
> pattern (assign to groups, not individuals) — just be upfront that this
> particular lab run used direct user bindings for a one-person project,
> and note *why* (the two errors above) if asked. That's a stronger answer
> than pretending groups were used.

**Custom least-privilege role:**
```bash
gcloud iam roles create custom.lz.deployer \
  --project="prj-lz-app-prod-${LZ_UNIQ}" \
  --title="LZ Deployer" \
  --permissions=compute.instances.create,compute.instances.delete,compute.disks.create
```

**Verify:**
```bash
gcloud projects get-iam-policy "prj-lz-app-prod-${LZ_UNIQ}" \
  --format="table(bindings.role,bindings.members)"
```

---

## Phase 1.3 — Shared VPC host and service projects (confirmed working)

**Two extra blockers hit and resolved beyond what the original guide
assumed** — documented here since both are strong "challenges faced and
resolved" material:

1. **Projects created outside the org.** `gcloud projects create` in Steps
   4/5 didn't specify `--organization`, so all 5 lab projects landed as
   standalone projects with no org parent. Shared VPC requires the host
   project to belong to an organization. Fixed with:
   ```bash
   for proj in net-host app-prod app-dev onprem shared; do
     gcloud beta projects move "prj-lz-${proj}-${LZ_UNIQ}" \
       --organization=43414815430
   done
   ```

2. **Missing `compute.organizations.enableXpnHost` /
   `enableXpnResource` permissions.** Project-level Owner isn't enough for
   Shared VPC operations — they're gated by org-level `roles/compute.xpnAdmin`
   specifically. Fixed with:
   ```bash
   gcloud organizations add-iam-policy-binding 43414815430 \
     --member="user:deegee.ck@gmail.com" \
     --role="roles/compute.xpnAdmin"
   ```
   (took ~60 seconds to propagate before the retry succeeded)

**Commands that ran successfully once both were resolved:**

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

---

## Phase 1.4 — Firewall rules (deny-by-default per tier)

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
  --format="table(name,direction,sourceRanges.list(),allowed[].map().firewall_rule().list())"
```
You should see all 3 rules, each scoped to the right port and source range.

---

## Once Phase 1 is fully verified, continue with:

Go back to **Phase 2 (Hybrid connectivity)** in
`gcp-hybrid-landing-zone-practice-project-v2.md` — `LZ_UNIQ` and the
billing account are already set correctly in this session, so every
command there will work as written.

## Running checklist to keep updating as you go

```
[x] Step 8   - APIs enabled on all 5 projects
[x] 1.2      - IAM roles bound (user-binding path)
[x] 1.3      - Shared VPC + subnets created, service projects attached
              (required: project org-move + compute.xpnAdmin grant)
[x] 1.4      - Firewall rules created (verified: all 3 on vpc-lz-shared)
[ ] Phase 2  - VPN + onprem simulation ← next
[ ] Phase 3  - Security baseline, labels, logging
[ ] Phase 4  - Backup & recovery
```
