# AWS Secure VPC --- Private Application Network with Zero Public Servers

A hands-on AWS networking and security project demonstrating a
three-tier architecture where only the Application Load Balancer is
internet-facing. The EC2 application servers and Amazon RDS MySQL
database are deployed in private subnets.

> **Lab note:** The tested listener is HTTP on port 80. HTTPS, WAF, VPC
> endpoints, and VPC Flow Logs are recommended enhancements and should
> not be considered deployed unless you configure them.

## Objectives

-   Build a custom VPC with public, private application, and private
    database subnets across two Availability Zones.
-   Expose the application through an internet-facing ALB without
    assigning public IPv4 addresses to EC2 application servers.
-   Use security-group references to restrict traffic between the ALB,
    EC2, and RDS tiers.
-   Administer private EC2 instances using AWS Systems Manager Session
    Manager instead of inbound SSH.
-   Connect the private application tier to private RDS MySQL.
-   Validate target health, HTTP delivery, and database connectivity.
-   Understand cost drivers and safe lab cleanup.

## Architecture

``` text
Internet users
    |
    | HTTP :80 (lab listener; add HTTPS for production)
    v
Internet-facing ALB: Omkar-Public-ALB
Public-AZ-A (10.0.1.0/24) + Public-AZ-B (10.0.2.0/24)
    |
    | Omkar-App-TG -> HTTP :80
    v
Private application tier (no public IPv4 addresses)
  +-- Omkar-Private-App-AZ-A | 10.0.11.0/24 | Amazon Linux 2023 + Apache
  +-- Omkar-Private-App-AZ-B | 10.0.12.0/24 | Amazon Linux 2023 + Apache
    |
    | TCP :3306 allowed by security-group reference
    v
Private Amazon RDS for MySQL: omkar-private-mysql
  +-- Private-DB-AZ-A | 10.0.21.0/24
  +-- Private-DB-AZ-B | 10.0.22.0/24

Private application outbound traffic (when needed):
Private-App-RT -> NAT Gateway in Public-AZ-A -> IGW -> Internet

Administration:
Authorized IAM principal -> Systems Manager Session Manager -> private EC2
```

## Network plan

  Subnet             CIDR             Purpose
  ------------------ ---------------- -------------------
  Public-AZ-A        `10.0.1.0/24`    ALB
  Public-AZ-B        `10.0.2.0/24`    ALB
  Private-App-AZ-A   `10.0.11.0/24`   Application EC2 A
  Private-App-AZ-B   `10.0.12.0/24`   Application EC2 B
  Private-DB-AZ-A    `10.0.21.0/24`   RDS subnet group
  Private-DB-AZ-B    `10.0.22.0/24`   RDS subnet group

VPC CIDR: `10.0.0.0/16`\
Region: Mumbai (`ap-south-1`)

## Main AWS resources

  -----------------------------------------------------------------------
  Resource                            Name
  ----------------------------------- -----------------------------------
  VPC                                 `Omkar-Secure-VPC`

  Internet Gateway                    `Omkar-Secure-IGW`

  Public route table                  `Omkar-Public-RT`

  Application route table             `Omkar-Private-App-RT`

  Database route table                `Omkar-Private-DB-RT`

  NAT Gateway                         `Omkar-NAT-AZ-A`

  ALB                                 `Omkar-Public-ALB`

  Target group                        `Omkar-App-TG`

  EC2 application servers             `Omkar-Private-App-AZ-A`,
                                      `Omkar-Private-App-AZ-B`

  RDS database                        `omkar-private-mysql`

  DB subnet group                     `omkar-private-db-subnet-group`

  EC2 IAM role                        `Omkar-EC2-SSM-Role`

  ALB security group                  `Omkar-ALB-SG`

  Application security group          `Omkar-Private-App-SG`

  RDS security group                  `Omkar-RDS-SG`
  -----------------------------------------------------------------------

## Routing and security

-   **Public route table:** `10.0.0.0/16 -> local`;
    `0.0.0.0/0 -> Internet Gateway`.
-   **Private application route table:** local route plus
    `0.0.0.0/0 -> NAT Gateway` for outbound internet access.
-   **Private database route table:** local route only; no default route
    to IGW or NAT Gateway.
-   **ALB security group:** allows HTTP :80 from the internet for lab
    testing; ALB forwards to targets on HTTP :80.
-   **Application security group:** inbound HTTP :80 only from
    `Omkar-ALB-SG`; no inbound SSH rule.
-   **RDS security group:** inbound MySQL :3306 only from
    `Omkar-Private-App-SG`.
-   **RDS:** Publicly accessible set to No.
-   **EC2 administration:** Session Manager with the
    `AmazonSSMManagedInstanceCore` policy attached to the instance role.

## Validation results

Based on the project operator's reported tests:

-   Session Manager terminal access succeeded.
-   Apache was installed and tested locally on both private EC2
    instances.
-   The ALB DNS name served the application.
-   Both target group instances reported Healthy.
-   Ncat connected to the RDS private address on TCP port 3306.
-   Database login and SQL tests succeeded.
-   Five final security checklist items were reported verified.

These are operator-reported results, not an independent query of the AWS
account.

## Why this project?

This pattern reduces direct exposure of application and database
servers, demonstrates defense in depth, and provides a common foundation
for internal business applications, customer portals, and multi-tier web
systems. A production deployment should additionally include HTTPS,
secrets management, backups/restore testing, monitoring, patching, and
an availability/cost decision for NAT and RDS.

## Cost considerations

No fixed price is stated because charges depend on region, usage,
instance size, storage, transfer, backup retention, and account
eligibility. Check **Billing and Cost Management → Bills** and **Cost
Explorer**.

-   EC2: compute hours and EBS storage.
-   NAT Gateway: hourly and data-processing charges, including when EC2
    is stopped.
-   ALB: hours and capacity usage, even if EC2 is stopped.
-   RDS: DB instance hours, storage, backups and related usage.
-   CloudWatch / Flow Logs: ingestion, storage and query charges.
-   Data transfer: internet and cross-AZ transfer where applicable.

For a lab, stop EC2 when idle, but understand that stopping EC2 does not
stop NAT Gateway, ALB, or RDS charges. Record settings and evidence
before deleting resources. A stopped RDS instance can automatically
restart after seven days.

## Limitations and future improvements

1.  Add HTTPS with AWS Certificate Manager and redirect HTTP to HTTPS.
2.  Decide whether the NAT Gateway should be deployed per AZ for
    production resilience, or whether appropriate VPC endpoints can
    reduce NAT dependence.
3.  Confirm whether RDS is Single-AZ or Multi-AZ; a DB subnet group
    spanning two AZs does not itself make the DB deployment Multi-AZ.
4.  Add VPC Flow Logs with a planned retention period if required.
5.  Add application-level health endpoints, secure secret storage,
    backup/restore tests, and infrastructure as code.
6.  Consider AWS WAF and least-privilege administrator policies for
    production.

## Repository structure suggestion

``` text
.
├── README.md
├── docs/
│   ├── architecture.md
│   ├── security-controls.md
│   ├── cost-and-cleanup.md
│   └── screenshots/
└── diagrams/
    └── architecture.png
```

## Evidence screenshots

The project owner supplied screenshots showing the VPC resource map, EC2
instances, Session Manager, RDS and DB subnet group, and test web pages.
Add sanitized screenshots to `docs/screenshots/` before publishing.
Remove account IDs, sensitive endpoints, public IPs where unnecessary,
and all credentials.

## Author

Omkar Pagade

## Disclaimer

This is a learning/portfolio project. Review actual AWS costs and
security settings before using a similar architecture for production
workloads.
