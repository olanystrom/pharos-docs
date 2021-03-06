# Add-on: Kontena Storage

Kontena Storage is a unified, distributed storage system designed for excellent performance, reliability and scalability. Kontena Storage is built on top of [Rook](https://rook.io) and it uses [Ceph](https://ceph.com/) as underlying storage system.

- version: `0.8.3+kontena.1`
- maturity: `beta`
- architectures: `x86-64`
- available in: `Pro`, `EE`

## Features

- `Block storage` - ReadWriteOnce persistent volumes
- `Shared filesystem` - ReadWriteMany persistent volumes
- `Dashboard` - overview of the status of your storage cluster

## Configuration

```yaml
addons:
  kontena-storage:
    enabled: true
    data_dir: /var/lib/kontena-storage
    storage:
      use_all_nodes: true
      # It's highly recommended to set directories and/or device_filter,
      # otherwise expanding storage does not work properly (see https://github.com/rook/rook/issues/1957)
      directories:
      - path: /mnt/data1
      # device_filter: ^sd[a-d]
    # pool:
    #   replicated:
    #     size: 3
    # dashboard:
    #   enabled: true
    # filesystem:
    #   enabled: true
    #   pool:
    #     replicated:
    #       size: 3
```

## Storage Classes

- `kontena-storage-block` - Ceph block storage. Default storage class.
- `kontena-storage-fs` - Shared filesystem. Only available if filesystem is enabled.

### Options

- `data_dir` - The path on the host (hostPath) where config and data should be stored for each of the services. If the directory does not exist, it will be created. Because this directory persists on the host, it will remain after pods are deleted.
- `storage` - Storage selection and configuration that will be used across the cluster. Note that these settings can be overridden for specific nodes.
    - `use_all_nodes` - `true` or `false`, indicating if all nodes in the cluster should be used for storage according to the cluster level storage selection and configuration values. If individual nodes are specified under the nodes field below, then `use_all_nodes` must be set to false.
    - `nodes` - Names of individual nodes in the cluster that should have their storage included in accordance with either the cluster level configuration specified above or any node specific overrides described in the next section below. `use_all_nodes` must be set to false to use specific nodes and their config.
        - `name` - The name of the node, which should match its `kubernetes.io/hostname` label.
        - [Storage selection settings](#storage-selection-settings)
        - `config` - Config settings applied to all OSDs on the node unless overridden by devices or directories. See the [config settings](#ceph-osd-configuration) below.
    - [Storage selection settings](#storage-selection-settings)
- `dashboard` - Storage dashboard settings
    - `enabled` - `true` or `false`, indicating if dashboard service should be enabled for the storage cluster. Default: `false`
- `filesystem` - Storage shared filesystem settings
    - `enabled` - `true` or `false`, indicating if shared filesystem should be enabled for the storage cluster. Default: `false`
    - `pool` - Filesystem storage pool settings.
        - `replicated` - Settings for the replicated pool.
            - `size` - The number of copies of the data in the pool.
- `placement` - Placement configuration for the cluster services. See [placement configuration](#placement-configuration-settings) below.
- `resources` - Resource configuration for the cluster services. See [resource configuration](#resources-configuration-settings) below.
- `pool` - Storage pool settings.
    - `replicated` - Settings for the replicated pool.
        - `size` - The number of copies of the data in the pool.

#### Storage Selection Settings

Below are the settings available, both at the cluster and individual node level, for selecting which storage resources will be included in the cluster.

- `device_filter` - A regular expression that allows selection of devices to be consumed by OSDs. If individual devices have been specified for a node then this filter will be ignored. This field uses golang regular expression syntax. For example:
    - `sdb` - Only selects the sdb device if found
    - `^sd.` - Selects all devices starting with sd
    - `^sd[a-d]` - Selects devices starting with sda, sdb, sdc, and sdd if found
    - `^s` - Selects all devices that start with s
    - `^[^r]` - Selects all devices that do not start with r
- `devices` - A list of individual device names belonging to this node to include in the storage cluster.
    - `name` - The name of the device (e.g., sda).
    - `config` - Device-specific config settings. See the config settings below.
    - `directories` - A list of directory paths that will be included in the storage cluster. Note that using two directories on the same physical device can cause a negative performance impact.
    - `path` - The path on disk of the directory (e.g., /mnt/data1).
    - `config` - Directory-specific config settings. See the config settings below.
- `directories` - A list of directory paths that will be included in the storage cluster. Note that using two directories on the same physical device can cause a negative performance impact.
    - `path` - The path on disk of the directory (e.g., /mnt/data1).
    - `config` - Directory-specific config settings. See the config settings below.

#### Placement Configuration Settings

Placement configuration for the cluster services. It includes the following keys: `mgr`, `mon`, `osd` and `all`. Each service will have its placement configuration generated by merging the generic configuration under all with the most specific one (which will override any attributes).

A Placement configuration is specified (according to the kubernetes PodSpec) as:

- `nodeAffinity` - kubernetes NodeAffinity
- `podAffinity` -  kubernetes PodAffinity
- `podAntiAffinity` - kubernetes PodAntiAffinity
- `tolerations` - list of kubernetes Toleration

The `mon` pod does not allow Pod affinity or anti-affinity. This is because of the mons having built-in anti-affinity with each other through the operator. The operator chooses which nodes are to run a mon on. Each `mon` is then tied to a node with a node selector using a hostname. See the [mon design doc](https://github.com/rook/rook/blob/master/design/mon-health.md) for more details on the mon failover design.

#### Resources Configuration Settings

Resources should be specified so that the storage components are handled after [Kubernetes Pod Quality of Service](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) classes. This allows to keep storage components running when for example a node runs out of memory and the storage components are not killed depending on their Quality of Service class.

You can set resource requests/limits for rook components through the Resource Requirements/Limits structure in the following keys:

- `mgr` - Set resource [requests/limits](#resource-requirements-limits) for MGRs.
- `mon` - Set resource [requests/limits](#resource-requirements-limits) for Mons.
- `osd` - Set resource [requests/limits](#resource-requirements-limits) for OSDs

#### Resource Requirements/Limits

For more information on resource requests/limits see the official Kubernetes documentation: [Kubernetes - Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#resource-requests-and-limits-of-pod-and-container)

- `requests` - Requests for cpu or memory.
    - `cpu` - Request for CPU (example: one CPU core 1, 50% of one CPU core 500m).
    - `memory` - Limit for Memory (example: one gigabyte of memory 1Gi, half a gigabyte of memory 512Mi).
- `limits` - Limits for cpu or memory.
    - `cpu` - Limit for CPU (example: one CPU core 1, 50% of one CPU core 500m).
    - `memory` - Limit for Memory (example: one gigabyte of memory 1Gi, half a gigabyte of memory 512Mi).

#### Ceph OSD Configuration

- `metadataDevice` - Name of a device to use for the metadata of OSDs on each node. Performance can be improved by using a low latency device (such as SSD or NVMe) as the metadata device, while other spinning platter (HDD) devices on a node are used to store data.
- `storeType` - `filestore` or `bluestore`, the underlying storage format to use for each OSD. The default is set dynamically to `bluestore` for devices, while `filestore` is the default for directories. Set this store type explicitly to override the default. Warning: Bluestore is not recommended for directories in production. Bluestore does not purge data from the directory and over time will grow without the ability to compact or shrink.
- `databaseSizeMB` - The size in MB of a bluestore database. Include quotes around the size.
- `walSizeMB` - The size in MB of a bluestore write ahead log (WAL). Include quotes around the size.
- `journalSizeMB` - The size in MB of a filestore journal. Include quotes around the size.


## Toolbox

Kontena Storage deploys a toolbox pod by default to the `kontena-storage` namespace. Toolbox can be used to debug and test Ceph cluster.

### Getting into the toolbox pod:

```
$ kubectl -n kontena-storage exec -it $(kubectl -n kontena-storage get pod -l "app=kontena-storage-tools" -o jsonpath='{.items[0].metadata.name}') bash
```

#### Common commands

- `ceph status` - a birds-eye view of cluster status.
- `ceph osd status` - to check the storage cluster OSD (Object Storage Daemon) status.
- `ceph df` - to check the storage cluster's data usage and distribution.


## Accessing Dashboard

If dashboard is enabled it can be accessed using the following command:

```
$ kubectl port-forward -n kontena-storage svc/rook-ceph-mgr-dashboard 7000
```

Then open browser at [localhost:7000](http://localhost:7000)
