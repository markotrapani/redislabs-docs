---
Title: Active-Active databases
linkTitle: Active-Active databases
description: Content related to Active-Active Redis Enterprise databases for Kubernetes. 
weight: 40
alwaysopen: false
categories: ["Platforms"]
aliases: {
    /kubernetes/active-active/_index.md,
    /kubernetes/active-active/,
    /kubernetes/preview/_index.md,
    /kubernetes/preview/,
    content/kubernetes/active-active/preview/_index.md,
    content/kubernetes/active-active/preview/,
}
---

On Kubernetes, Redis Enterprise [Active-Active]({{<relref "/rs/databases/active-active/">}}) databases provide read and write access to the same dataset from different Kubernetes clusters.

## Active-Active setup methods

There are two methods for creating an Active-Active database with Redis Enterprise for Kubernetes:

- The `RedisEnterpriseActiveActiveDatabase` (REAADB) custom resource is available for versions 6.4.2 and later.
- The `crdb-cli` method is available for versions 6.4.2 or earlier.


We recommend creating new Active-Active databases using the RedisEnterpriseActiveActiveDatabase (REAADB) custom resource. This allows you to manage your Active-Active database with the operator and ensures you have the latest features and functionality.

### Active-Active controller method

Versions 6.4.2 or later support the Active-Active controller. This setup method includes the following steps:

1. Gather REC credentials and [prepare participating clusters]({{<relref "/kubernetes/active-active/prepare-clusters.md">}}).
2. Create [`RedisEnterpriseRemoteCluster` (RERC)]({{<relref "/kubernetes/active-active/create-reaadb#create-rerc">}}) resources.
3. Create [`RedisEnterpriseActiveActiveDatabase` (REAADB)]({{<relref "/kubernetes/active-active/create-reaadb#create-reaadb">}}) resource.

### `crdb-cli` method

For versions 6.4.2 or earlier, this Active-Active setup method includes the following steps:

1. Install and configure an ingress.
2. Gather configuration details.
3. Add the `ActiveActive` field to the REC spec.
4. Create the database with the `crdb-cli` tool.

## Redis Enterprise Active-Active controller for Kubernetes

{{<note>}} The features below are only available for releases 6.4.2-4 and later. {{</note>}}

[Active-Active]({{<relref "/rs/databases/active-active/">}}) databases give you read-and-write access to Redis Enterprise clusters (REC) in different Kubernetes clusters or namespaces. Active-Active deployments managed by the Redis Enterprise operator require two additional custom resources: Redis Enterprise Active-Active database (REAADB) and Redis Enterprise remote cluster (RERC).

To create an Active-Active Redis Enterprise deployment for Kubernetes with these new features, first [prepare participating clusters]({{<relref "/kubernetes/active-active/prepare-clusters.md">}}) then [create an Active-Active database]({{<relref "/kubernetes/active-active/create-reaadb.md">}}).

For the currently supported procedure, see [Create Active-Active databases for Kubernetes]({{<relref "/kubernetes/active-active/create-aa-crdb-cli.md">}}).

### REAADB custom resource

Redis Enterprise Active-Active database (REAADB) contains a link to the RERC for each participating cluster, and provides configuration and status to the management plane.

For a full list of fields and options, see the [REAADB API reference](https://github.com/RedisLabs/redis-enterprise-k8s-docs/blob/master/redis_enterprise_active_active_database_api.md).

### RERC custom resource

Redis Enterprise remote cluster (RERC) custom resource contains configuration details for all the participating clusters.

For a full list of fields and options, see the [RERC API reference](https://github.com/RedisLabs/redis-enterprise-k8s-docs/blob/master/redis_enterprise_remote_cluster_api.md).

### Limitations

* Admission is not blocking REAADB with `shardCount` which exceeds license quota. (RED-96301)

    Workaround: Fix the problems with the REAADB and reapply.
* The `<rec-name>/<rec-namespace>` value must be unique for each RERC resource. (RED-96302)
* Only global database options are supported, no support for specifying configuration per location.
* No support for migration from old (`crdb-cli`) Active-Active database method to new Active-Active controller.
* No support for multiple participating clusters on the same Kubernetes cluster.

## More info

For more general information about Active-Active, see the [Redis Enterprise Software docs]({{<relref "/rs/databases/active-active/">}}).
