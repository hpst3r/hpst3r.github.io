---
title: "Performing a partial or full restore of an Elasticsearch cluster from a snapshot"
date: 2026-02-23T20:30:00-00:00
draft: false
---

See the docs at [elastic.co/docs](https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore).

I'm starting with a three-node cluster (see [Deploying a three-node HA Elasticsearch 9 cluster on AlmaLinux 9](https://wporter.org/building-out-a-test-3-node-elasticsearch-9-cluster-on-almalinux-9)). It consists of:

3x KVM VM (4 vCPU, 16 GiB RAM, 128 GB of SSD)

- es9-1
- es9-2
- es9-3

DNS is working.

I've loaded the Flights [sample dataset](https://www.elastic.co/docs/manage-data/ingest/sample-data) to the cluster with Kibana just to have something to look at.

I'm adding:

1x KVM VM (2 vCPU, 4 GiB RAM, 128 GB of SSD), for a snapshot repository (NFS server):

- snapshot-repo

1x KVM VM (4 vCPU, 16 GiB RAM, 128 GB of SSD), for a restore target (fresh single-node cluster):

- es9c2n1

## Configure the snapshot repository

I'll be snapshotting to a NFS mount running on one of our new VMs.

### Configure the NFS server

This VM, "snapshot-repo", will just be serving a directory to the cluster.

We'll set up `/mnt/data/snapshots` and configure NFS:

- I'm `chown`-ing to the nobody user (uid 65534) so I can use all_squash to a minimally privileged user.
- You will need working reverse DNS if you intend to use wildcard FQDN exports like I did here.
- Consider that we're not setting up authentication. NFS things.

```sh
sudo mkdir -p /mnt/data/snapshots
sudo chown 65534:65534 /mnt/data/snapshots
sudo dnf install -y nfs-utils
sudo tee /etc/exports > /dev/null << 'EOT'
/mnt/data/snapshots *.elastic.wporter.org(rw,all_squash)
EOT
sudo systemctl enable --now nfs-server
```

### On the Elastic nodes

First, you'll need to install the `nfs-utils` package.

```sh
sudo dnf install -y nfs-utils
sudo mkdir -p /mnt/data/snapshots
```

To mount the share once:

```sh
sudo mount -t nfs snapshots.elastic.wporter.org:/mnt/data/snapshots /mnt/data/snapshots
```

Or, to mount the share at boot going forwards:

```sh
sudo tee -a /etc/fstab > /dev/null << 'EOT'
snapshots.elastic.wporter.org:/mnt/data/snapshots /mnt/data/snapshots nfs vers=4.2,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,_netdev,noatime,actimeo=0 0 0
EOT
sudo systemctl daemon-reload
sudo mount -av
```

Then, you'll have to populate the `path.repo` property on all Elastic nodes to include the snapshot repo path (in this case, `/mnt/data/snapshots`):

```sh
sudo tee -a /etc/elasticsearch/elasticsearch.yml > /dev/null << 'EOT'
path:
  repo:
    - /mnt/data/snapshots
EOT
```

You'll need to restart Elasticsearch on your nodes to apply the change. Then, you can register the snapshot repository with the cluster:

```http
PUT _snapshot/snapshots_es_nl
{
  "type": "fs",
  "settings": {
    "location": "/mnt/data/snapshots"
  }
}
```

To create a snapshot, use the [`PUT /_snapshot/{repository}/{snapshot_name}` API endpoint](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-snapshot-create).

Arguments:

- `"indices": []` means include all indices on the cluster
- `"ignore_unavailable": true` means it's OK to skip restoring unavailable shards

```http
PUT /_snapshot/snapshots_es_nl/snapshot_foobar?wait_for_completion=true
{
  "indices": [],
  "ignore_unavailable": true,
  "include_global_state": true,
  "metadata": {
    "taken_by": "wporter",
    "taken_because": "test backup"
  }
}
```

DO NOT modify these files on disk. You may back up or restore these files only if they are not being written to by Elastic (consider making them read-only before running backups).

```txt
[wporter@es9-3 ~]$ sudo ls -l /mnt/data/snapshots
total 96
-rw-r--r--.  1 nobody nobody 12660 Dec 24 10:49 index-0
-rw-r--r--.  1 nobody nobody     8 Dec 24 10:49 index.latest
drwxr-xr-x. 46 nobody nobody  4096 Dec 24 10:49 indices
-rw-r--r--.  1 nobody nobody 66189 Dec 24 10:49 meta-KHTlbXnpS92aRSqxTH9-tw.dat
-rw-r--r--.  1 nobody nobody  1429 Dec 24 10:49 snap-KHTlbXnpS92aRSqxTH9-tw.dat
```

Snapshots **don’t** contain or back up:

- Transient cluster settings
- Registered snapshot repositories
- Node configuration files
- [Security configuration files](https://www.elastic.co/docs/deploy-manage/security)

Be sure to back up your config files, document adding the snapshot repository, and document setting up nodes to bootstrap a cluster if you want to be capable of a *full* cluster restore.

A snapshot repository should be read/write for one cluster only to prevent corruption. To set the repository read-only, toggle the parameter in Kibana (clicky-click) or PUT/POST:

```http
PUT _snapshot/snapshots_es_nl
{
  "type": "fs",
  "settings": {
    "location": "/mnt/data/snapshots",
    "readonly": true
  }
}
```

## Perform a partial restore

To see what's in a snapshot, e.g., a list of snapshotted indices, version, uuid, shard count, state, start/end - recall that snapshots are not *one* point in time, they're consistent over a range of time - and features, GET the snapshot:

```http
GET /_snapshot/snapshots_es_nl/snapshot_foobar
```

If restoring data to the same cluster or a new cluster, POST to the restore endpoint, selecting the indices you'd like to restore. For example, in this case, I'll restore all non-system indices from the `snapshot_foobar` snapshot.

```http
POST /_snapshot/snapshots_es_nl/snapshot_foobar/_restore
{
  "indices": "*,-.*",
  "include_global_state": false
}
```

The `*,-.*` pattern excludes system indices.

## Full restore to a new cluster

If your cluster has been hit with the God hammer (or something), you can also *completely* restore it (including settings and security) from a snapshot. This takes a little more prep. [Relevant docs](https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore/restore-snapshot#restore-entire-cluster).

I've got a separate restore target - a fresh Elastic install of the same version.

**On the restore target, we will be effectively shutting down most of Elastic's features and overwriting their indices.** This includes:

- Security
- Monitoring
- History
- Watcher

Additionally, if any data indices already exist, they will be overwritten. **If you do not want to overwrite the entire target cluster** do **NOT** do this. Do **NOT** perform a restore to a cluster that is anything other than a clean slate.

### Prepare the restore target

First, we'll need to stop nonessential features so we can close their indices. Some of these may or may not be applicable in your environment.

```sh
# disable GeoIP DB DL and ILM history to close their indices
PUT _cluster/settings
{
  "persistent": {
    "ingest.geoip.downloader.enabled": false,
    "indices.lifecycle.history_index_enabled": false
  }
}

# disable ILM to close its indices
POST _ilm/stop

# disable ML to close its indices
POST _ml/set_upgrade_mode?enabled=true

# disable monitoring to close its indices
PUT _cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": false
  }
}

# disable watcher to close its indices
POST _watcher/_stop

# if Universal Profiling index templ management is on turn it off:
GET /_cluster/settings?filter_path=**.xpack.profiling.templates.enabled&include_defaults=true

PUT _cluster/settings
{
  "persistent": {
    "xpack.profiling.templates.enabled": false
  }
}
```

Next, we'll need to configure a special superuser that has access to delete restricted indices.

> If you use Elasticsearch security features, follow [File-based access recovery](https://www.elastic.co/docs/troubleshoot/elasticsearch/file-based-recovery) to temporarily create a user with temporary elevated permissions to edit restricted indices. Use this file realm user to authenticate requests until the restore operation is complete.

To do this.. let's first go over the file auth realm:

> You don’t need to explicitly configure a `file` realm. The `file` and `native` realms are added to the realm chain by default. Unless configured otherwise, the `file` realm is added first, followed by the `native` realm. You can define only one `file` realm on each node.
>
> **In a self-managed Elasticsearch cluster**, all the data about the users for the `file` realm is stored in two files on each node in the cluster: [`users` and `users_roles`](https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/file-based#using-users-and-users_roles-files). Both files are located in `ES_PATH_CONF` and are read on startup.

As a reminder, `ES_PATH_CONF` is `/etc/elasticsearch` when installed via the RPM packages.

The permissions we need to add are `allow_restricted_indices`. We'll also need something not tied to Elastic Security since we'll be wiping Elastic Security - two birds with one stone, right?

Create the "superduperuser" role with every permission by adding it to the roles file. By default, this file is blank (just has some comments) so I'll just `tee --append`:

```sh
sudo tee -a /etc/elasticsearch/roles.yml > /dev/null << 'EOT'
superduperuser:
  cluster: [ 'all' ]
  indices:
    - names: [ '*' ]
      privileges: [ 'all' ]
      allow_restricted_indices: true
EOT
```

Then, the recommended way to add a user is to use the `elasticsearch-users` utility. To add a 'local' admin (file-based):

```sh
/usr/share/elasticsearch/bin/elasticsearch-users useradd admin -p changeme -r superduperuser
```

Then, confirm the user can be used to authenticate against the new cluster:

```txt
[root@es9c2n1 ~]# /usr/share/elasticsearch/bin/elasticsearch-users useradd admin -p changeme -r superduperuser
[root@es9c2n1 ~]# cat /etc/elasticsearch/users
admin:$2a$10$DRtzaTBnUQ2tJxDi4IVDk.QALcpJM/.jrKctxMSD35j.mBKH2gZp2
[root@es9c2n1 ~]# cat /etc/elasticsearch/users_roles
superduperuser:admin
[root@es9c2n1 ~]# curl https://localhost:9200 -k -u admin:changeme
{
  "name" : "es9c2n1",
  "cluster_name" : "es9c2",
  "cluster_uuid" : "F7ePSggFQWy1SipRSg5kPg",
  "version" : {
    "number" : "9.2.3",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "d8972a71dbbd64ff17f2f4dba9ca2c3fe09fb100",
    "build_date" : "2025-12-16T10:09:08.849001802Z",
    "build_snapshot" : false,
    "lucene_version" : "10.3.2",
    "minimum_wire_compatibility_version" : "8.19.0",
    "minimum_index_compatibility_version" : "8.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

Time to be destructive! Here's what we'll need to do. Using the API, start breaking things (either using Kibana Dev Tools or cURL). Wipe out every non-restricted index.

```http
# allow deletion of indices with wildcards so we can wipe the cluster
PUT _cluster/settings
{
  "persistent": {
    "action.destructive_requires_name": false
  }
}

# delete all existing data streams
DELETE _data_stream/*?expand_wildcards=all

# delete all existing indices
DELETE *?expand_wildcards=all
```

Then, make a web request as our superduperadmin to kill the rest of our indices - this will break everything, so be warned (again).

```sh
curl https://localhost:9200/*?expand_wildcards=all -X "DELETE" -k -u admin:changeme
```

OK! Now we've got an empty cluster.

To perform a full restore of a snapshot to it, make a POST request to the `snapshot/_restore` API endpoint to restore `*` indices and global state:

```sh
# restore ENTIRE snapshot
POST /_snapshot/snapshots_es_nl/snapshot_foobar/_restore
{
  "indices": "*",
  "include_global_state": true
}
```

You won't be able to run that command from Kibana, since we don't really have Kibana joined anymore. So, do it with cURL:

```sh
curl https://localhost:9200/_snapshot/snapshots_es_nl/snapshot_foobar/_restore -H "Content-Type: application/json" -d '{"indices":"*","include_global_state":true}' -k -u admin:changeme
```

To monitor the ongoing restore, you can hit the `_cluster/health` API endpoint (it will typically go green once everything is restored), check `index/_recovery` for each index, or check `_cat/shards?v=true` to see shards.

Once your restore is complete, Kibana will need a rejoin (consider removing the package, `/var/lib/kibana`, and `/etc/kibana` and reinstalling it - this is probably simplest), but try making a request to the Elastic API via `cURL` with the superuser account from the original cluster - it should work:

```txt
[root@es9c2n1 ~]# curl https://localhost:9200/_cluster/health?pretty=true -k -u elastic:changeme
{
  "cluster_name" : "es9c2",
  "status" : "yellow",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 45,
  "active_shards" : 45,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 1,
  "unassigned_primary_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 97.82608695652173
}
```

(This restore target is a single-node cluster receiving a three-node cluster, so it won't automatically place that replica shard).

After reinstalling/reconfiguring/rejoining Kibana, you should see that your accounts, dashboards, etc have been carried over:

{{< figure src="images/0-kibana-post-restore.png" >}}

After the restore, the settings you changed (e.g., watcher enabled, ILM enabled, `action.destructive_requires_name: true`) should have been restored to the state they were in during the snapshot. Confirm this with a few HTTP requests:

```http
GET _cluster/settings?filter_path=**.action.destructive_requires_name&include_defaults=true
# should be destructive_requires_name: true

# reset it if it's not
PUT _cluster/settings
{
  "persistent": {
    "action.destructive_requires_name": null
  }
}

GET _cluster/settings?filter_path=**.ingest.geoip.downloader.enabled&include_defaults=true
# should be enabled: true

# restart ILM
POST _ilm/start

# ensure ML not disabled
POST _ml/set_upgrade_mode?enabled=false

# if needed, restart monitoring
GET _cluster/settings?filter_path=**.xpack.monitoring.collection.enabled&include_defaults=true

PUT _cluster/settings
{
  "persistent": {
    "xpack.monitoring.collection.enabled": true
  }
}

POST _watcher/_start

# if you stopped Universal Profiling, reenable it:
PUT _cluster/settings
{
  "persistent": {
    "xpack.profiling.templates.enabled": true
  }
}
```

Additionally, remove the local superduperuser by wiping it from the users and roles files:

```sh
sed -i -e "s/^admin:.*$//" /etc/elasticsearch/users
sed -i -e "s/^.*:admin$//" /etc/elasticsearch/users_roles
```

This will apply the next time Elastic reads these files (in 5s).
