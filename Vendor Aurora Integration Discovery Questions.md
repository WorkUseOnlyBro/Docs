# Vendor Aurora Integration: Discovery Questions

Sep 23, 2026 · @Someone

## Purpose

We're in the discovery phase of setting up a recurring feed of your data into our environment, at least twice a day. No approach or platform has been chosen yet. Your answers will help both sides decide what fits best.

As we understand it, the data lives in two places: the current database on Amazon Aurora, which has an existing read replica, and an archive on MariaDB hosted on AWS. The sections below outline the options under consideration, the questions that will shape the decision, and the technical details we'd need for each option.

## Options at a glance

We haven't settled on an approach. These are the integration patterns under consideration, and any of them could work depending on your answers below.

**Integration patterns**

| Pattern | How it works | Network requirement | Effort on your side | Effort on our side |
| --- | --- | --- | --- | --- |
| Scheduled pull over Twingate | We query your existing read replica on a schedule and load our database | A Twingate service account; outbound-only connections on both sides | Read-only database user, Twingate resource and service key | Extraction job and monitoring |
| Scheduled pull over site-to-site VPN | Same, over an IPsec tunnel into your network | VPN on both ends; non-overlapping IP ranges | VPN setup and routing | VPN plus extraction job |
| Scheduled file delivery | You export files on a schedule to agreed storage (S3, Azure Blob or SFTP), and we load them | Storage access only; no path into your network | Export job, manifest and file retention | Loader and monitoring |
| Continuous replication (CDC) | Changes replicate in near real time | A persistent connection; binlog or logical replication enabled | Replication configuration and monitoring | Always-on pipeline |

The target platform on our side is also open. It could be Snowflake, Azure SQL, or an on-prem SQL Server, MySQL or PostgreSQL database.

## Questions that shape the approach

Your answers to these will do the most to narrow the options, so they're the best place to start.

**Integration pattern**

| Question | Why we're asking |
| --- | --- |
| What's behind your preference for scheduled file dumps: a security policy, existing tooling, staffing, or something else? | Helps us understand which constraints are fixed. |
| Would your security team approve a Twingate service account for an unattended, read-only connection to the read replica? | Tells us whether a direct pull is possible over your existing access layer. |
| Would a site-to-site VPN be acceptable as another network path? | Identifies the connectivity options available. |
| Is the read replica dedicated to integrations and reporting, or shared with your application? What replica lag is typical? | Tells us what load a pull would add and how current the replica's data is. |
| Is twice a day a fixed schedule or a starting point? | Some patterns handle higher frequency better than others. |
| Who on your side would own an export or integration job long term, and how are failures handled? | Clarifies ongoing support on both sides. |
| Do you already export this data anywhere, for backups or other customers? | An existing process might be reusable. |
| If data is delivered as files, what formats can you produce (SQL dump, CSV, Parquet), and can each delivery include a manifest? | Determines how files would be loaded and validated. |

**Target platform**

| Question | Why we're asking |
| --- | --- |
| Do you already use a cloud data platform, such as Snowflake? | Helps us understand which delivery methods are possible. |
| Have you delivered this data to other customers? Onto which platforms, and using what method? | A proven path might be reusable. |
| Do you have a preference or existing tooling for delivering to Snowflake, Azure SQL, or on-prem SQL Server, MySQL or PostgreSQL? | Narrows the platforms to what both sides can support. |
| Is the current database Aurora MySQL or Aurora PostgreSQL? | Affects type conversion and tooling on our side. |
| Roughly how large are the current database and the archive, and how fast do they grow? (Details in Data volume and growth.) | Drives sizing on our side. |

## Source database

These are the basics we need to connect and to model the data on our side.

**Current database (Aurora)**

- [ ] Engine and version: Aurora MySQL or Aurora PostgreSQL, and which version
- [ ] AWS region of the cluster
- [ ] Hostname and port of the existing read replica
- [ ] Database and schema names in scope
- [ ] List of tables in scope, with the primary key for each. Tables without a primary key need a plan before we start.
- [ ] DDL for each table in scope (`SHOW CREATE TABLE` output for MySQL, `pg_dump --schema-only` for PostgreSQL)
- [ ] Columns with large or unusual types: `TEXT`/`BLOB`, JSON, spatial, enums, arrays
- [ ] Character set and collation
- [ ] Timezone that timestamps are stored in. UTC is ideal; local time needs a daylight-saving plan.
- [ ] Any views or derived tables we should use instead of raw tables
- [ ] Data dictionary or column descriptions, if one exists

**Archive database (MariaDB on AWS)**

- [ ] MariaDB version, and how it's hosted on AWS (a managed service or self-managed)
- [ ] Hostname and port, and whether it has a read replica
- [ ] Is it reachable through the same Twingate network as the Aurora replica, or would it need separate access?
- [ ] Same read-only credentials as the current database, or separate ones?
- [ ] Is the archive still being written to? If so, how and how often does data move from Aurora into it?
- [ ] Do archive tables have the same structure as the current tables, or different?
- [ ] DDL, character set, collation and timestamp timezone, as for the current database

## Data volume and growth

These numbers drive storage and compute provisioning on any target. Ask for per-table figures, not just a cluster total, because two or three large tables usually dominate.

**Current size**

- [ ] Total size of the in-scope data, including indexes
- [ ] Size and approximate row count per table (queries below)
- [ ] The largest single table and its size. This drives how long the initial backfill takes.
- [ ] Cluster volume from the Aurora `VolumeBytesUsed` CloudWatch metric, as a cross-check

**Archive**

- [ ] MariaDB archive size, including indexes, and per-table sizes (the MySQL query below also works on MariaDB)
- [ ] How far back the archive goes
- [ ] If the archive still grows, how much data moves into it per day or month
- [ ] Is any data in both the current database and the archive at the same time?

**Growth and change rate**

- [ ] Rows inserted, updated and deleted per day for each large table. Updates and deletes matter as much as inserts.
- [ ] Average daily growth in GB, and the trend over the last 12 months
- [ ] Peak periods (month-end, seasonal, batch jobs) and how much higher volume runs then
- [ ] Planned changes: new tables, large migrations, new customers or products that would change volume

**Queries the vendor can run for per-table sizes**

Aurora MySQL or the MariaDB archive (row counts from `information_schema` are estimates for InnoDB tables):

```sql
SELECT table_name,
       table_rows AS approx_rows,
       ROUND((data_length + index_length) / POWER(1024, 3), 2) AS size_gb
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY size_gb DESC;
```

Aurora PostgreSQL:

```sql
SELECT relname AS table_name,
       n_live_tup AS approx_rows,
       ROUND(pg_total_relation_size(relid) / POWER(1024, 3)::numeric, 2) AS size_gb
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

## Change tracking and extraction

An incremental load only works if we can reliably tell what changed since the last run. These answers decide between incremental loads and full reloads, table by table.

- [ ] Does every table have a last-modified timestamp (for example `updated_at`)? Is it set by the application, by a trigger, or by the database?
- [ ] Is that timestamp updated on every change, including bulk updates and backfill scripts run outside the application?
- [ ] For append-only tables, is there an increasing ID that could be used instead?
- [ ] Are deletes hard or soft? Hard deletes are invisible to timestamp-based pulls and would need a delete log, a periodic key comparison, or replication.
- [ ] Which tables are small enough to reload in full every run?
- [ ] Do archive tables carry the same change-tracking columns?
- [ ] Is binlog (MySQL/MariaDB) or logical replication (PostgreSQL) enabled, and what is the retention? Relevant if continuous replication is considered.
- [ ] How will you notify us of schema changes (new, renamed or dropped columns, type changes), and how much notice will we get?
- [ ] Are there windows when the read replica shouldn't be queried, or a maximum query runtime to stay under?
- [ ] Any row-level filtering required (for example, only certain customers or regions)?

## Access, network and authentication

If data is pulled from the read replica, we'd need a read-only database login and a network path. Twingate and a site-to-site VPN are both possible paths.

**Database access**

- [ ] A dedicated read-only database user, limited to the in-scope tables, on the read replica and on the archive if it's in scope
- [ ] How credentials would be delivered (a secrets tool or a one-time link, not email), and the rotation schedule with notice
- [ ] Which authentication method: password over TLS, or IAM database authentication? IAM auth relies on short-lived tokens generated with AWS credentials, which affects how we'd connect from outside AWS.
- [ ] Is TLS enforced? Which CA bundle should we trust?
- [ ] Maximum concurrent connections allowed for our user
- [ ] Security group rules on the read replica and archive, so traffic from your Twingate Connector or our VPN subnet is allowed on the database port

**Twingate**

- [ ] Would you create a Twingate service account for us and issue a service key? Service accounts are meant for machine-to-machine access. They can't satisfy 2FA or use identity-provider logins, and they run the client in headless mode ([Twingate: How Service Accounts Work](https://twingate.com/docs/service-accounts-guide)).
- [ ] Service key expiry, and how a replacement would be sent before it expires
- [ ] Would the Twingate resource be scoped to the read replica and archive hostnames and database port only?
- [ ] Do those hostnames resolve through Twingate DNS?
- [ ] Any device-posture or session policies an unattended server can't satisfy
- [ ] How many Connectors serve that network?
- [ ] Idle or session timeouts that could cut off a long-running extract

**Site-to-site VPN**

- [ ] Your VPC CIDR ranges, so we can confirm they don't overlap ours
- [ ] VPN peer public IPs (AWS Site-to-Site VPN provides two tunnels)
- [ ] IKE version, encryption, integrity and Diffie-Hellman settings, and how the pre-shared key will be exchanged
- [ ] Static routes or BGP
- [ ] Who monitors the tunnel on your side, and the escalation path when it drops

## File delivery

If data is delivered as files, these details would define the handoff.

**Format**

- [ ] File format: SQL dump, CSV or Parquet. For CSV: delimiter, quoting, escaping, header row, null representation, encoding.
- [ ] Compression (gzip, zstd, Snappy for Parquet)
- [ ] One file per table, or split into chunks? Target file size?
- [ ] Full snapshot every delivery, or only changes since the last delivery? If changes, how are deletes represented?
- [ ] How will a schema change show up in the files, and how much notice will we get?

**Location and access**

- [ ] Delivery location: their S3 bucket with cross-account read access, our S3 or Azure Blob container, or SFTP
- [ ] Who owns the storage, the credentials and the rotation
- [ ] Encryption at rest (SSE-S3, SSE-KMS) and, if KMS, access to the key
- [ ] Folder and file naming convention, including date and time in UTC

**Completeness and timing**

- [ ] A manifest per delivery listing files, row counts and checksums, written only after all data files finish
- [ ] Delivery times, and the latest a delivery can arrive before we treat it as late
- [ ] How we'll be notified of a failed or skipped delivery

**Retention and replay**

- [ ] How long files stay available, so we can reload after a failure on our side
- [ ] Can we request a redelivery or a one-off full snapshot, and how quickly?

## Operations, security and compliance

These questions cover who does what when something breaks, and what rules apply to the data.

**Operations**

- [ ] Technical contact and escalation path, including out-of-hours coverage
- [ ] Maintenance windows and failover events we should expect, and how we'll be notified
- [ ] Expected availability of the read replica, archive database or file drop
- [ ] How the initial historical backfill will happen: over the same path, as a one-off bulk export, or as a snapshot export to S3
- [ ] Who validates that our copy matches theirs, and how (row counts, checksums on key tables)

**Security and compliance**

- [ ] Data classification: does any in-scope data include PII, payment data, health data or anything else regulated?
- [ ] Required agreements: NDA, data processing agreement, security questionnaire
- [ ] Restrictions on where the data may be stored (region, cloud, on-prem only)
- [ ] Retention or deletion obligations we inherit, including deletion requests passed down from their customers
- [ ] Audit or logging requirements on our access
- [ ] Offboarding: how access is revoked and what we must delete if the relationship ends

## Sources

- [Twingate: How Service Accounts Work](https://twingate.com/docs/service-accounts-guide)
