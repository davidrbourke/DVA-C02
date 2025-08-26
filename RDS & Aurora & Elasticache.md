# Relational Database Service (RDS)

- A Managed database service for relational databases that SQL
- Engines:
  - Aurora (AWS Propietary Database)
  - IBM DB2
  - Oracle
  - MySQL
  - MariaDB
  - Postgress
  - SQL Server (Microsoft)

## Managed Service

- Automated Provision
- Automated patching of underlying operating system
- Continuous backups and point in time restore
- Monitoring dashboards
- Read replicas for performance
- Multi AZ for disaster recovery
- Maintenance windows for upgrades
- Scaling capability (vertical and horizontal)
- EBS backed storage
- **Cannot SSH into managed instances - don't have access**

## Storage auto-scaling

When you create the DB you specify storage space required. As you get closer using up the space, more space is **automatically** made available (if enabled).

- Set a Maximum Storage Threshold
- Rules to automatically modify storage:
  - Free storage is < 10%
  - Low storage lasts >= 5 minutes
  - 6 hours have passed since the last modification in storage size
- Supports all RDS engines, and good for where storage is unpredictable.
