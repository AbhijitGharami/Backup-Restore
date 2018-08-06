# Managing Backup & Restore in large scale

In our PostgreSQL cluster setup, we have one primary and one hot standby server. And we have asynchronous replication between them. There are three Pgpool VMs to manage the failover. If the primary server fails, then Pgpool VMs promotes the standby server as a primary and the old primary server comes back as a new standby server. We use STONITH mechanism for failover. 

We have more then 5000 cluster spreads across Openstack private cloud, AWS, Azure and GCP clouds. To support point in recovery (PITR) of the PostgreSQL database we have WAL archiving for all these cluster. For base backup we take either snapshot or tar of the data directory depending upon different IaaS (Infrastructure as a Service) and in recovery process we restore these snapshots or tarball, and we apply the archived WAL logs on top of that restored data.

### Base backup
We have a component called Service-Fabric, it helps in managing the house keeping activities of these clusters. This component takes care of triggering the backup periodically. It triggers the backup for each cluster in every 8 hours interval except Openstack. In case of Openstack, the interval is 24 hours. Each backup is stored against a guid to identify the backup uniquely. 

We take base backup from primary server.  And it is an online backup, which means during backup the service is available for use. The backup approaches are different for different IaaS. In Openstack, first we take a snapshot and then we create a volume from that snapshot. And then we tar and encrypt the data directory that is present in this newly created volume. Finally, we upload the encrypted tarball to the swift. It takes time depending on the data size. For a data size of 100 GB it takes around 1 hour 30 minutes. 

In case of AWS, Azure and GCP we use snapshot-based approach. We take the snapshot of the disk where the PostgreSQL data directory is present. AWS supports the incremental backup where in the first snapshot it takes the backup of the full disk and from the next snapshot it takes the backup of the modified block from the last snapshot. For a data size of 100 GB it takes around 1 hour. For Azure the snapshot takes around 2 to 3 minutes irrespective of the data size.

### Restore
In restore we bring down the PostgreSQL cluster that means the service will be not available to the user. We do restore only on primary VM, and from standby VM we delete the data directory and take a pg_basebackup from the primary server. And thus, we bring back the whole cluster.

In Openstack, we download the encrypted tarball, and we decrypt it and untar it to PostgreSQL data directory. For a 100 GB of data it takes around 3 hours to restore. In case of AWS, Azure and GCP, we create a volume from the snapshot, and we copy the data directory from the newly created volume to the existing volume’s data directory. In AWS, it takes around 1 hour to restore the 100 GB of data. AWS uses lazy loading concept where it downloads the data in background. We use fio, a linux utility to improve the restore time. In Azure the restore takes around 4 hours for 100 GB of data. After restoring the base backup, we apply the WAL logs on top of that to support point in time recovery.
