☁️ Infra — GCP Hybrid Cloud Landing Zone
Enterprise-style Google Cloud infrastructure lab demonstrating hybrid networking, security governance, centralized observability, and disaster recovery.





🎯 Executive Summary

This project is a hands-on replication of a real-world hybrid-cloud landing zone on Google Cloud, built to demonstrate the engineering practices required for cloud infrastructure and platform roles.

The environment covers:

    Shared VPC networking across production and development projects
    HA VPN + Cloud Router/BGP hybrid connectivity
    Deny-by-default firewall architecture
    Organization Policy security controls
    Least-privilege IAM
    Centralized logging to BigQuery and Cloud Storage
    Cloud Monitoring detection for firewall-deny spikes
    Scheduled snapshots and Cloud SQL PITR
    Actual backup restoration with measured RTO
    Intentional network failure and troubleshooting
    Infrastructure naming, tagging, and governance

The objective was deliberately broader than provisioning resources: build → test → introduce failure → troubleshoot → recover → document.

    🔐 Repository Redaction

    Project IDs, billing account IDs, organization IDs, and personal email addresses shown in this repository are placeholders (<PROJECT_ID>, <ORG_ID>, etc.) or have been blurred in screenshots.

    Real values were used during the actual build but are not published here.

🏗️ Architecture

High-level topology

                         ┌──────────────────────────┐
                         │    GCP Organization       │
                         │                          │
                         │  Organization Policies   │
                         │  IAM / Governance        │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         ┌──────────────────────────┐
                         │    Shared VPC Host       │
                         │        Project           │
                         └────────────┬─────────────┘
                                      │
                ┌─────────────────────┼─────────────────────┐
                │                     │                     │
                ▼                     ▼                     ▼
          ┌───────────┐         ┌───────────┐         ┌───────────┐
          │ Web Tier  │         │ App Tier  │         │ Data Tier │
          │  Subnet   │         │  Subnet   │         │  Subnet   │
          └───────────┘         └───────────┘         └───────────┘
                │                     │                     │
                └─────────────────────┼─────────────────────┘
                                      │
                         ┌────────────┴────────────┐
                         │                         │
                         ▼                         ▼
                  ┌─────────────┐           ┌─────────────┐
                  │    PROD     │           │     DEV     │
                  │  Service    │           │  Service    │
                  │   Project   │           │   Project   │
                  └─────────────┘           └─────────────┘
                         │
                         │
                    HA VPN / BGP
                         │
                         ▼
                  ┌─────────────┐
                  │ Simulated   │
                  │  On-Prem    │
                  │  Network    │
                  └─────────────┘

🧩 What Was Built
01 — Identity & Core Networking

A Shared VPC host/service project topology was implemented to mirror an enterprise operating model where network infrastructure is centrally managed while workload teams operate from separate service projects.
Implemented

    Shared VPC host project
    Production service project
    Development service project
    Web / application / data subnet segmentation
    Centralized firewall configuration
    Centralized routing
    Least-privilege IAM
    Custom deployer role

custom.lz.deployer

Rather than granting broad primitive roles, the custom role was restricted to the permissions required by a deployer.
Evidence

docs/screenshots/01-shared-vpc.png
🌐 02 — Hybrid Connectivity

The environment establishes connectivity between Google Cloud and a simulated on-premises environment using:

HA VPN
   │
   ▼
Cloud Router
   │
   ▼
BGP
   │
   ▼
Simulated On-Prem Network

Real Failure Scenario

A deliberate overlapping IP-range conflict was introduced between the simulated on-premises environment and the production network.

This reproduced a realistic problem encountered during on-premises-to-cloud migrations.
Troubleshooting process

Overlapping CIDR ranges
          ↓
Routing conflict
          ↓
Connectivity failure
          ↓
Route / address-space investigation
          ↓
Conflict identified
          ↓
Address range corrected
          ↓
Connectivity verified

The issue was not merely simulated in documentation—the conflict was introduced into the environment, diagnosed, corrected, and verified.
Evidence

    docs/screenshots/02-vpn-tunnel-established.png
    docs/screenshots/03-ip-overlap-error.png
    docs/screenshots/04-ip-overlap-fixed.png

Detailed analysis:

troubleshooting.md
🛡️ 03 — Security & Governance

Security controls were implemented at the organization level rather than relying exclusively on individual project configuration.
Organization Policy

The security baseline enforces:

    🚫 No external IP addresses
    🔐 OS Login required

This demonstrates how centralized governance can prevent insecure configurations from being introduced into workload projects.
Evidence

docs/screenshots/05-org-policy.png
🔑 Least-Privilege IAM

The lab intentionally avoids treating project ownership as equivalent to organization-level administrative authority.

One of the actual implementation issues demonstrated this distinction:

    Project Owner ≠ organization-level Shared VPC administrator

The required organization-level permission was granted using:

roles/compute.xpnAdmin

This was necessary to successfully administer the Shared VPC configuration.
🏷️ 04 — Resource Governance & Tagging

Every resource follows a standardized metadata model.
Mandatory labels
Label	Purpose
environment	Environment identification
owner	Resource ownership
project	Workload/project association
cost-center	Cost attribution
data-classification	Data sensitivity classification

Terraform variable validation enforces the required label set.

This turns resource metadata from a manual convention into an infrastructure deployment requirement.

Naming standards are documented in:

naming-conventions.md
📊 05 — Centralized Logging & Detection

Centralized logging was implemented using an aggregated logging sink.

                  Cloud Logging
                       │
                       ▼
              Aggregated Log Sink
                  │          │
                  ▼          ▼
              BigQuery       GCS
                  │
                  ▼
           Security Analysis
                  │
                  ▼
        Firewall-Deny Spike Alert

The project deliberately includes an actual alert rather than stopping at log collection.
Detection implemented

Firewall-deny spike alert

This demonstrates the operational progression from:

collect → centralize → analyze → detect → alert
💾 06 — Backup & Disaster Recovery

The recovery architecture covers both compute and database workloads.
Compute

Scheduled disk snapshots provide recurring backup protection for persistent disks.
Cloud SQL

Cloud SQL Point-in-Time Recovery (PITR) provides database recovery capability.
Recovery Was Actually Tested

The project does not simply demonstrate that backup jobs completed.

A real restoration was performed into an isolated project, and the recovery process was timed end-to-end.

This produced a measured Recovery Time Objective (RTO) based on an actual restoration workflow.
Evidence

docs/screenshots/06-backup-restore-timing.png

    Key principle: A successful backup job is not equivalent to a tested recovery strategy. Recovery was explicitly performed and measured.

🔥 Real Problems Encountered

One of the primary goals of this project was to document real implementation failures and their resolutions, rather than presenting an idealized deployment.
Problem	Root Cause	Resolution
Billing quota exceeded on project link	Default 5-projects-per-billing-account quota	Checked usage, freed capacity, and planned around the quota
Project ID already in use	GCP project IDs are globally unique and remain reserved after deletion	Adopted a session-scoped unique suffix convention
Shared VPC: no organization	Projects were created without an organization parent	Created projects with --organization; moved existing projects with gcloud beta projects move
Shared VPC: permission denied	Project Owner did not provide organization-level Shared VPC administration	Granted roles/compute.xpnAdmin at the organization level
Group IAM binding failed	SOLO_GROUP_OWNERS_DISALLOWED plus nonexistent placeholder-domain group	Used direct least-privilege user bindings for the solo lab; documented group-based IAM as the production pattern
Hybrid routing conflict	Overlapping IP ranges between on-prem and production networks	Identified and corrected the conflicting address range
Why this matters

These failures provided practical experience with:

    GCP quota management
    Project lifecycle behavior
    Organization hierarchy
    Organization-level IAM
    Shared VPC administration
    IAM group constraints
    CIDR/IP planning
    Hybrid routing troubleshooting

Full details:

troubleshooting.md
🔐 Security Model

The overall security approach follows defense in depth:

             Organization Policy
                     │
                     ▼
              Least-Privilege IAM
                     │
                     ▼
              Shared VPC Segmentation
                     │
                     ▼
             Deny-by-Default Firewall
                     │
                     ▼
               Private Networking
                     │
                     ▼
              Centralized Logging
                     │
                     ▼
               Security Detection

The design separates:

    Organization-level governance
    Network administration
    Workload projects
    Deployment identities
    Monitoring and audit infrastructure

🧰 Technology Stack
Category	Technologies
Cloud	Google Cloud Platform
IaC / Validation	Terraform
CLI	gcloud
Networking	VPC, Shared VPC, HA VPN, Cloud Router, BGP
Security	IAM, Organization Policy, Firewall
DNS	Cloud DNS
Logging	Cloud Logging
Monitoring	Cloud Monitoring
Analytics	BigQuery
Archive	Cloud Storage
Compute DR	Scheduled disk snapshots
Database DR	Cloud SQL PITR
📁 Repository Structure

Infra-gcp-hybrid-landing-zone/
│
├── README.md
├── troubleshooting.md
├── naming-conventions.md
│
├── docs/
│   ├── architecture.png
│   │
│   └── screenshots/
│       ├── 01-shared-vpc.png
│       ├── 02-vpn-tunnel-established.png
│       ├── 03-ip-overlap-error.png
│       ├── 04-ip-overlap-fixed.png
│       ├── 05-org-policy.png
│       └── 06-backup-restore-timing.png
│
└── ...

📸 Implementation Evidence
Capability	Evidence
Shared VPC	01-shared-vpc.png
HA VPN established	02-vpn-tunnel-established.png
IP overlap failure	03-ip-overlap-error.png
IP overlap resolved	04-ip-overlap-fixed.png
Organization Policy	05-org-policy.png
Backup/restore RTO	06-backup-restore-timing.png
🎓 Skills Demonstrated
☁️ Cloud Infrastructure

    GCP project architecture
    Shared VPC
    Multi-environment design
    Cloud networking
    Cloud SQL
    Compute Engine

🌐 Network Engineering

    VPC subnet design
    HA VPN
    Cloud Router
    BGP
    CIDR planning
    Route troubleshooting
    Hybrid connectivity

🔐 Security

    IAM
    Custom roles
    Least privilege
    Organization Policy
    OS Login
    Firewall segmentation
    External-IP restrictions

📈 Observability

    Centralized logging
    Aggregated sinks
    BigQuery log storage
    Cloud Storage log archival
    Monitoring alerts
    Firewall-deny detection

💾 Resilience

    Scheduled snapshots
    Cloud SQL PITR
    Restore testing
    Isolated recovery environment
    RTO measurement

🛠️ Troubleshooting

    Quota investigation
    Project lifecycle issues
    Organization hierarchy
    IAM permission analysis
    Shared VPC administration
    IP overlap diagnosis
    Hybrid routing recovery

💡 Engineering Takeaways

This project was intentionally built to demonstrate that cloud infrastructure engineering is more than provisioning resources.

The implementation covers the complete operational lifecycle:

        DESIGN
          ↓
      IMPLEMENT
          ↓
        TEST
          ↓
   INTRODUCE FAILURE
          ↓
     TROUBLESHOOT
          ↓
       RECOVER
          ↓
       MEASURE
          ↓
      DOCUMENT

The resulting experience demonstrates practical capability in:

    Designing a multi-project GCP landing zone.
    Centralizing network administration with Shared VPC.
    Implementing least-privilege IAM.
    Building hybrid connectivity with HA VPN, Cloud Router, and BGP.
    Diagnosing and resolving real IP-address overlap.
    Enforcing organization-wide security controls.
    Implementing centralized logging and actionable detection.
    Establishing infrastructure naming and metadata standards.
    Implementing backup and database recovery.
    Performing and measuring an actual restore.
    Troubleshooting real GCP platform, IAM, quota, and networking failures.
    Documenting technical decisions and operational issues clearly.

🚀 Project Outcome

The completed lab provides an enterprise-style hybrid-cloud landing-zone reference implementation demonstrating the intersection of:

Infrastructure + Networking + Security + Governance + Observability + Disaster Recovery

Rather than focusing only on successful deployment, the project emphasizes operational realism through intentional failure injection, troubleshooting, recovery validation, and measured results.
📌 Documentation

    Architecture: docs/architecture.png
    Troubleshooting: troubleshooting.md
    Naming & tagging standards: naming-conventions.md
    Implementation evidence: docs/screenshots/

Conclusion

Thank you for exploring this GCP project. This project demonstrates the use of Google Cloud services to build, deploy, and manage a scalable and reliable cloud-based solution. We hope this README provides everything needed to understand, set up, and work with the project.

For questions, improvements, or issues, please feel free to contribute or raise an issue.

Thank you for using this project, and happy cloud computing! ☁️🚀




		
