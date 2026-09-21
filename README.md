# On-Prem to AWS Migration: Employee Directory App

An Employee Directory application migrated from a simulated on-premises server to AWS, using EC2 as the source, Amazon RDS as the target, and AWS DMS to move the data.

![Architecture diagram](./screenshots/architecture-diagram.png)

## What this demonstrates

- End-to-end database migration with AWS DMS: replication instance, source/target endpoints, full-load migration task
- EC2, RDS, and VPC security group configuration from the console
- Layered troubleshooting across OS, network, and database access-control issues (below)
- Cutover: repointing a running application from a local database to a managed cloud database

**Stack:** Node.js · MySQL · AWS EC2 · AWS RDS · AWS DMS

## The build

1. Ran the app locally against a local MySQL instance, as the "before" baseline
2. Rebuilt the app and database on an EC2 instance, simulating an on-premises server
3. Provisioned an RDS MySQL instance as the migration target
4. Configured DMS (replication instance, source endpoint, target endpoint, full-load task) and ran the migration
5. Verified the migrated data matched the source exactly
6. Cut the running application over to RDS

![App running against the migrated database](./screenshots/app-ui.png)
*Employee Directory running post-migration.*

![DMS migration task, load completed](./screenshots/dms-task-complete.png)
*Migration task at 100%, load completed.*

![Migrated data verified in RDS](./screenshots/rds-data-verified.png)
*Direct query against RDS confirming all rows migrated intact.*

## Troubleshooting

A clean run-through would have taken under an hour. Getting the networking and access control right took considerably longer — each issue below sits at a different layer, not a repeat of the same mistake.

**MySQL only listening on `127.0.0.1`, unreachable from any other machine.**
Ubuntu's MySQL package binds to localhost only by default. Invisible in local development, since app and database share a machine — it only becomes a problem once a separate machine (the DMS replication instance) needs to reach it. Fixed with `bind-address = 0.0.0.0`.

![Confirming the bind-address fix](./screenshots/bind-address-fix.png)
*`ss -tlnp` confirming MySQL now listening on all interfaces, not just localhost.*

**DMS timeouts traced to a security group that wasn't actually attached to the instance.**
I'd been editing a security group I assumed was on my EC2 instance. It wasn't — the instance was using a different, auto-generated group from the launch wizard. Fixed by confirming the actual attached security group for each resource before adding more rules.

**DMS authenticated but was rejected: "host not allowed to connect" (MySQL error 1130).**
The app's database user only had a `'localhost'` grant, which doesn't cover a TCP connection from a different host, even with the correct password. MySQL treats `'user'@'localhost'` and `'user'@'%'` as separate accounts. Fixed by adding an explicit `'user'@'%'` grant.

**The throughline:** every issue was a different layer independently deciding whether to allow a connection — OS-level bind address, AWS-level security group, MySQL-level host grant. None of them talk to each other, so a correct password and a running service weren't enough on their own. Diagnosing it meant isolating each layer in turn (localhost first, then the network path, then the exact grant) rather than guessing at a single fix.

## What I'd change for production

- Dedicated, narrowly-scoped security groups per resource, instead of one shared group (a deliberate simplification here, for speed)
- DMS full-load-and-CDC instead of a one-time load, for near-zero-downtime cutover
- RDS and the DMS replication instance on private subnets, reached only from within the VPC
- Provisioned with Terraform and deployed through CI/CD instead of manual console steps
