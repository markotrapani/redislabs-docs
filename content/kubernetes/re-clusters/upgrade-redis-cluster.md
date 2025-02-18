---
Title: Upgrade a Redis Enterprise cluster (REC)
linkTitle: Upgrade a Redis cluster
description: This task describes how to upgrade a Redis Enterprise cluster via the operator.
weight: 19
alwaysopen: false
categories: ["Platforms"]
aliases: [
    /rs/administering/kubernetes/upgrading-redis-enterprise-cluster-kubernetes-deployment-operator/,
    /platforms/kubernetes/upgrading-kubernetes-with-operator/,
    /platforms/kubernetes/tasks/upgrading-with-operator.md,
    /platforms/kubernetes/tasks/upgrading-with-operator/,
    /platforms/kubernetes/rec/upgrade-redis-cluster.md,
    /platforms/kubernetes/rec/upgrade-redis-cluster/,
    /kubernetes/re-clusters/upgrade-redis-cluster.md,
    /kubernetes/re-clusters/upgrade-redis-cluster/,
]
---
Redis implements rolling updates for software upgrades in Kubernetes deployments. The upgrade process consists of two steps:

  1. [Upgrade the Redis Enterprise operator](#upgrade-the-operator)
  2. [Upgrade the Redis Enterprise cluster (REC)](#upgrade-the-redis-enterprise-cluster-rec)

## Before upgrading

Review the following warnings before starting your upgrade.

### Supported upgrade paths**

   If you are using a version earlier than 6.2.10-45, you must upgrade to 6.2.10-45 before you can upgrade to versions 6.2.18 or later.

### OpenShift clusters running 6.2.12 or earlier

Version 6.4.2-6 includes a new SCC (`redis-enterprise-scc-v2`) that you need to bind to your service account before upgrading. OpenShift clusters running version 6.2.12 or earlier upgrading to version 6.2.18 or later might get stuck if you skip this step. See [reapply SCC](#reapply-the-scc) for details.

### RHEL7-based images

When upgrading existing Redis Enterprise clusters running on RHEL7-based images, make sure to select a RHEL7-based image for the new version. See [release notes]({{<relref "/kubernetes/release-notes/">}}) for more info.

### Invalid license

Verify your license is valid before upgrading your REC. Invalid licenses will cause the upgrade to fail.

Use `kubectl get rec` and verify the `LICENSE STATE` is valid on your REC before you start the upgrade process.

### Large clusters

On clusters with more than 9 REC nodes, running versions 6.2.18-3 through 6.2.4-4, a Kubernetes upgrade can render the Redis cluster unresponsive in some cases.

A fix is available in the 6.4.2-5 release. Upgrade your operator version to 6.4.2-5 or later before upgrading your Kubernetes cluster. 

## Upgrade the operator

### Download the bundle

Make sure you pull the correct version of the bundle. You can find the version tags
by checking the [operator releases on GitHub](https://github.com/RedisLabs/redis-enterprise-k8s-docs/releases)
or by [using the GitHub API](https://docs.github.com/en/rest/reference/repos#releases).

You can download the bundle for the latest release with the following `curl` command:

```sh
VERSION=`curl --silent https://api.github.com/repos/RedisLabs/redis-enterprise-k8s-docs/releases/latest | grep tag_name | awk -F'"' '{print $4}'`
curl --silent -O https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/$VERSION/bundle.yaml
```

For OpenShift environments, the name of the bundle is `openshift.bundle.yaml`, and so the `curl` command to run is:

```sh
curl --silent -O https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/$VERSION/openshift.bundle.yaml
```

If you need a different release, replace `VERSION` in the above with a specific release tag.

### Apply the bundle

Apply the bundle to deploy the new operator binary. This will also apply any changes in the new release to custom resource definitions, roles, role binding, or operator service accounts.

{{< note >}}
If you are not pulling images from Docker Hub, update the operator image spec to point to your private repository.
If you have made changes to the role, role binding, RBAC, or custom resource definition (CRD) in the previous version, merge them with the updated declarations in the new version files.
{{< /note >}}

Upgrade the bundle and operator with a single command, passing in the bundle YAML file:

```sh
kubectl apply -f bundle.yaml
```

If you are using OpenShift, run this instead:

```sh
kubectl apply -f openshift.bundle.yaml
```

After running this command, you should see a result similar to this:

```sh
role.rbac.authorization.k8s.io/redis-enterprise-operator configured
serviceaccount/redis-enterprise-operator configured
rolebinding.rbac.authorization.k8s.io/redis-enterprise-operator configured
customresourcedefinition.apiextensions.k8s.io/redisenterpriseclusters.app.redislabs.com configured
customresourcedefinition.apiextensions.k8s.io/redisenterprisedatabases.app.redislabs.com configured
deployment.apps/redis-enterprise-operator configured
```

### Reapply the admission controller webhook {#reapply-webhook}

If you have the admission controller enabled, you need to manually reapply the `ValidatingWebhookConfiguration`.

{{<note>}}
{{< embed-md "k8s-642-redb-admission-webhook-name-change.md" >}}
{{</note>}}

{{< embed-md "k8s-admission-webhook-cert.md"  >}}

### Reapply the SCC

If you are using OpenShift, you will also need to manually reapply the [security context constraints](https://docs.openshift.com/container-platform/4.8/authentication/managing-security-context-constraints.html) file ([`scc.yaml`]({{<relref "/kubernetes/deployment/openshift/openshift-cli#deploy-the-operator" >}})) and bind it to your service account.

```sh
oc apply -f openshift/scc.yaml
```

```sh
oc adm policy add-scc-to-user redis-enterprise-scc-v2 \ system:serviceaccount:<my-project>:<rec-name>
```

If you are upgrading from operator version 6.4.2-6 or before, see the ["after upgrading"](#after-upgrading) section to delete the old SCC and role binding after all clusters are running 6.4.2-6 or later.

### Verify the operator is running

You can check your deployment to verify the operator is running in your namespace.

```sh
kubectl get deployment/redis-enterprise-operator
```

You should see a result similar to this:

```sh
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
redis-enterprise-operator   1/1     1            1           0m36s
```

{{< warning >}}
 We recommend upgrading the REC as soon as possible after updating the operator. After the operator upgrade completes, the operator suspends the management of the REC and its associated REDBs, until the REC upgrade completes.
 {{< /warning >}}

## Upgrade the Redis Enterprise cluster (REC)

{{<warning>}}
Verify your license is valid before upgrading. Invalid licenses will cause the upgrade to fail.

Use `kubectl get rec` and verify the `LICENSE STATE` is valid on your REC before you start the upgrade process.
{{</warning>}}

The Redis Enterprise cluster (REC) can be updated automatically or manually. To trigger automatic upgrade of the REC after the operator upgrade completes, specify `autoUpgradeRedisEnterprise: true` in your REC spec. If you don't have automatic upgrade enabled, follow the below steps for the manual upgrade.

Before beginning the upgrade of the Redis Enterprise cluster, check the K8s operator release notes to find the Redis Enterprise image tag. For example, in Redis Enterprise K8s operator release [6.0.12-5](https://github.com/RedisLabs/redis-enterprise-k8s-docs/releases/tag/v6.0.12-5), the `Images` section shows the Redis Enterprise tag is `6.0.12-57`.

After the operator upgrade is complete, you can upgrade Redis Enterprise cluster (REC).

### Edit `redisEnterpriseImageSpec` in the REC spec

1. Edit the REC custom resource YAML file.

    ```sh
    kubectl edit rec <your-rec.yaml>
    ```

1. Replace the `versionTag:` declaration under `redisEnterpriseImageSpec` with the new version tag.

    ```YAML
    spec:
      redisEnterpriseImageSpec:
        imagePullPolicy:  IfNotPresent
        repository:       redislabs/redis
        versionTag:       <new-version-tag>
    ```

1. Save the changes to apply.

### Monitor the upgrade

You can view the state of the REC with `kubectl get rec`.

  During the upgrade, the state should be `Upgrade`.
  When the upgrade is complete and the cluster is ready to use, the state will change to `Running`.
  If the state is `InvalidUpgrade`, there is an error (usually relating to configuration) in the upgrade.

```sh
$ kubectl get rec
NAME   NODES   VERSION      STATE     SPEC STATUS   LICENSE STATE   SHARDS LIMIT   LICENSE EXPIRATION DATE   AGE
rec    3       6.2.10-107   Upgrade   Valid         Valid           4              2022-07-16T13:59:00Z      92m
```

To see the status of the current rolling upgrade, run:

```sh
kubectl rollout status sts <REC_name>
```

### After upgrading

For OpenShift users, operator version 6.4.2-6 introduces a new SCC (`redis-enterprise-scc-v2`). If any of your OpenShift RedisEnterpriseClusters are running versions earlier than 6.2.4-6, you need to keep both the new and old versions of the SCC.

If all of your clusters have been upgraded to operator version 6.4.2-6 or later, you can delete the old version of the SCC (`redis-enterprise-scc`) and remove the binding to your service account.

1. Delete the old version of the SCC

   ```sh
   oc delete scc redis-enterprise-scc
   ```

   The output should look similar to the following:

   ```sh
   securitycontextconstraints.security.openshift.io "redis-enterprise-scc" deleted
   ```

1. Remove the binding to your service account.

   ```sh
   oc adm policy remove-scc-from-user redis-enterprise-scc system:serviceaccount:<my-project>:<rec-name>
   ```


### Upgrade databases

After the cluster is upgraded, you can upgrade your databases. The process for upgrading databases is the same for both Kubernetes and non-Kubernetes deployments. For more details on how to [upgrade a database]({{<relref "/rs/installing-upgrading/upgrading/upgrade-database">}}), see the [Upgrade an existing Redis Enterprise Software deployment]({{<relref "/rs/installing-upgrading/upgrading">}}) documentation.

Note that if your cluster [`redisUpgradePolicy`]({{<relref "/kubernetes/reference/cluster-options#redisupgradepolicy">}}) or your database [`redisVersion`]({{<relref "/kubernetes/reference/db-options#redisversion">}}) are set to `major`, you won't be able to upgrade those databases to minor versions. See [Redis upgrade policy]({{<relref "/rs/installing-upgrading/upgrading#redis-upgrade-policy">}}) for more details.

## How does the REC upgrade work?

The Redis Enterprise cluster (REC) uses a rolling upgrade, meaning it upgrades pods one by one. Each pod is updated after the last one completes successfully. This helps keep your cluster available for use.

To upgrade, the cluster terminates each pod and deploys a new pod based on the new image.
  Before each pod goes down, the operator checks if the pod is a primary (master) for the cluster, and if it hosts any primary (master) shards. If so, a replica on a different pod is promoted to primary. Then when the pod is terminated, the API remains available through the newly promoted primary pod.

After a pod is updated, the next pod is terminated and updated.
After all of the pods are updated, the upgrade process is complete.
