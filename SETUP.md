# Pushing this to GitHub

This folder is ready to become a repo. From this directory:

```bash
git init
git add .
git commit -m "Initial commit: GCP hybrid landing zone build"
git branch -M main
git remote add origin https://github.com/dcube-g/infra-gcp-hybrid-landing-zone.git
git push -u origin main
```

On GitHub, create the repo first with:
- **Name:** `infra-gcp-hybrid-landing-zone`
- **Description:** Hands-on GCP hybrid cloud landing zone — Shared VPC, HA VPN, security baseline, and tested backup/recovery, with real troubleshooting documented
- **Visibility:** Public
- **Do not** initialize with a README — this repo already has one ready to push

Before pushing, double check no real project IDs, org IDs, billing account
IDs, or personal emails remain in any file — search for them explicitly:

```bash
grep -rn "jd01\|43414815430\|172919905395\|01A629\|deegee" .
```

Replace any matches with placeholders before committing. Screenshots
should be added to `docs/screenshots/` following the naming convention in
`docs/screenshot-checklist.md`, with sensitive values blurred in the image
itself before saving.
