---
title: "5.6.3. Amazon RDS for MySQL"
weight: 3
chapter: false
---

# Amazon RDS for MySQL

## Verified database configuration

`ClinicDatabase` is an `AWS::RDS::DBInstance`. `DatabaseSubnetGroup` is an `AWS::RDS::DBSubnetGroup` containing both private subnets.

| Setting | Current template |
|---|---|
| Engine | `mysql`; no explicit `EngineVersion`. |
| Instance class | `DatabaseInstanceClass` parameter, default `db.t4g.micro`; overridable. |
| Storage | 20 GiB, `gp3`, encrypted. |
| Availability | `MultiAZ: false` (Single-AZ); no specific AZ pinned. |
| Public access | `PubliclyAccessible: false`. |
| Port / Security Group | TCP 3306 / `DatabaseSecurityGroup`. |
| Subnet group | `DatabaseSubnetGroup`: `PrivateSubnetA`, `PrivateSubnetB`. |
| Credentials | `ManageMasterUserPassword: true`; RDS manages the password in Secrets Manager. |
| Backups | `BackupRetentionPeriod: 1` day. |
| Maintenance | `AutoMinorVersionUpgrade: true`. |
| Log exports | `error` and `slowquery` to CloudWatch. |
| Snapshot tags | `CopyTagsToSnapshot: true`. |
| Removal / replacement | `DeletionPolicy: Snapshot`, `UpdateReplacePolicy: Snapshot`. |
| Deletion protection | Not explicitly set; no live setting is asserted. |

A short excerpt of the DB properties:

```yaml
Engine: mysql
AllocatedStorage: '20'
DBInstanceClass:
  Ref: DatabaseInstanceClass
MultiAZ: false
PubliclyAccessible: false
StorageEncrypted: true
StorageType: gp3
Port: 3306
ManageMasterUserPassword: true
```

The DB subnet group makes private subnet placement available across two AZs. It does **not** create a second database instance or a standby when `MultiAZ` is false.

## Why managed MySQL

Schedules, appointments, users, and doctors form a relational workload with transactional changes. RDS fits this model while handling database operations as a managed service, avoiding the need to operate a MySQL server directly on EC2. Private routing and the Lambda-only MySQL ingress rule protect the database network boundary.

Single-AZ and a small default instance class fit the learning/demo environment. They do not provide production high availability. Production availability requirements may justify Multi-AZ and separate capacity decisions.

Application schema migrations belong to a later deployment stage. This page does not connect to the database or run migrations.

## Cost and retention implications

A running DB instance incurs instance charges, with storage and other applicable charges considered separately. See [Amazon RDS for MySQL pricing](https://aws.amazon.com/rds/mysql/pricing/). No estimated dollar amount or live billing result is asserted here.

Snapshot deletion/replacement policies can leave retained snapshots after stack removal or DB replacement. Those snapshots may continue to incur storage costs and require a later retention/cleanup decision.

{{< notice warning >}}
This is a learning/demo database, not a production high-availability design. RDS can incur ongoing costs while running. Detailed cost and cleanup steps come later; no cleanup is performed here.
{{< /notice >}}

📷 Screenshot to add: Private RDS configuration showing Single-AZ, encryption, subnet group, and public-access setting. Redact endpoints, credentials, and private identifiers.
