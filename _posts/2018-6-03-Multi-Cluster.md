---
layout: post
title: Run your CI pipelines across multiple Clusters / Clouds
---

InfraBox has a quite unique feature for running your Continuous Integration
jobs on multiple clusters. So you can have multiple Kubernetes clusters,
even on different networks (like different cloud providers and/or on-prem),
with an InfraBox installation in each of them.

This can be useful in several scenarios:

- You want to run your tests or other task on a certain cloud
infrastructure. Maybe your tests access your cloud vendors API and you
don't want to connect remotely.
- Autoscale your CI infrastructure. InfraBox works very well with
auto-scaling Kubernetes clusters. So you can extend your on-prem cluster
(which might not have auto-scaling) with an auto-scaling cluster in the
cloud and only make use of it if you are low on resources in your local
cluster. This can be exremely useful to keep the testing roundtrips short
on days with usage spikes.
- Depending on your application you might have to test it on different
Kubernetes versions. You could now create multiple Kubernetes clusters with
the versions you want to test on and use the [Namespace Service](https://github.com/SAP/InfraBox/tree/master/src/services/namespace) to
provision a Kubernetes namespace in the same cluster your job is running.
This is an easy way of doing some real end-to-end tests on different
Kubernetes versions or even different Kubernetes distributions and clouds.
- Some applications have to run on different CPU architectures like x64 and
PowerPC. You may want to have a cluster on each of these architectures to
run your builds and tests on both (InfraBox builds for PowerPC are not yet
available, but we are working on it).

The requirements for a multi cluster-configuration are simple. Basically
you need a shared database to which all InfraBox installations can connect
to as well as each InfraBox API endpoint has to be accessible by the others.

![_config.yml]({{ site.baseurl
}}/images/2018-6-3-Multi-Cluster/multi_cluster.svg)

As you can see in the picture each cluster can have a different storage
like S3 or GCS, so you don't need a shared storage, but can rely on local
storage offerings of your cloud vendor. InfraBox currently supports:

- [S3 (or S3 compatible like Minio)](
https://github.com/SAP/InfraBox/blob/master/docs/install/storage/s3.md)
- [Google Cloud Storage](
https://github.com/SAP/InfraBox/blob/master/docs/install/storage/gcs.md)
- [Azure Storage](
https://github.com/SAP/InfraBox/blob/master/docs/install/storage/azure.md)

One of the clusters acts as the `master` installation, which hosts a full
InfraBox installation including the Dashboard. The `worker` clusters only
run the API, scheduler, operators and Docker registry, but not the
Dashboard. You can see all the jobs from all clusters in the UI of the
`master` installation. The clusters register themselves automatically with
the `master` as soon as they connect to the database for the first time.

The installation for `master` and `worker`s are very similar. Let's start with
the `master` (also see the [installation guide](
https://github.com/SAP/InfraBox/tree/master/docs)):

{% highlight AnyLanguage %}
host: master.example.com

database:
    postgres:
        enabled: true
        host: my-db-host.my-domain.com
        port: 5432
        db: # <REQUIRED>
        username: # <REQUIRED>
        password: # <REQUIRED>


storage:
    s3:
        enabled: true
        region: # <REQUIRED>
        endpoint: # <REQUIRED>
        bucket: # <REQUIRED>
        access_key_id: # <REQUIRED>
        secret_access_key: # <REQUIRED>
        secure: true
        port: 443
{% endhighlight %}

This is the configuration of a regular InfraBox instance. This basically
means you don't have to do anything special for the `master` installation. If
you already have an instance you can simply extend it without
reconfiguration.

The `worker` configuration is very similar:

{% highlight AnyLanguage %}
host: worker.example.com

cluster:
    name: worker
    labels: my-worker,GCP

database:
    postgres:
        enabled: true
        host: my-db-host.my-domain.com
        port: 5432
        db: # <REQUIRED>
        username: # <REQUIRED>
        password: # <REQUIRED>


storage:
    gcs:
        enabled: true
        bucket: # <REQUIRED>
        service_account: # <REQUIRED>
{% endhighlight %}

As you can see the database connection is the same. This is the requirement
as all installations share the same database and use it to synchronize the
states. For the `worker`s we have decided to go with a different storage, in
this case GCS, because we assume the cluster is running on GCP and GCS is
then a natural choice. Also the `worker` has a different `host`. This
is important, because the API servers of each installation must talk to
each other. So you have to make sure that you assign a domain to each
cluster which can be accessed by the other clusters. The big difference to
the `master` is the configuration for the `cluster.name` and
`cluster.labels`. The `cluster.name` decides whether this is the
`master` or `worker`. The default value for `cluster.name` is `master`.
Every other name is treated as a `worker`. Make sure you have only one
cluster with the name `master`!

Additionally you can, but don't have to, specify `cluster.labels`. They
can be used in your job definition to force the scheduler to place a job on
a certain cluster.

{% highlight AnyLanguage %}
{
    "version": 1,
    "jobs": [{
        "type": "docker",
        "name": "test",
        "build_only": false,
        "docker_file": "infrabox/test/Dockerfile",
        "resources": {
            "limits": { "cpu": 1, "memory": 1024 }
        },
        "cluster": {
            "selector": ["GCP"]
        }
    }]
}
{% endhighlight %}

The `cluster.selector` can be used to specify on which cluster a job is
supposed to be run. In the example the job will be run on a cluster which
has the label `GCP` attached. If there are multiple clusters with this
label then a cluster is chosen randomly. You may also set multiple labels
in your selector. In this case a cluster is chosen which has all the labels
attached.

There's the special label `default`. Every cluster which has this label
assigned is supposed to run jobs which don't have any selector set. If you
dont set `cluster.labels` then the `default` label will be attached to
the cluster, meaning that the cluster may start every job which doesn't
have a `label.selector` forcing it to some other cluster. You may also set
the `default` label together with other labels like
`cluster.labels=default,GCP`. Then every job which does not have a
`cluster.selector` and the ones with a selector for `GCP` will be run on
this cluster.

Coming back to the autoscaling scenario from above. This can be easily
achieved by configuring your regular `master` cluster and a second `worker`
cluster with `cluster.labels=default` (maybe even with auto-scaling on,
in case it's running on a cloud). Now you can run a lot more jobs and have
to only pay for what you use on a cloud. If you want to disable a `worker`
cluster you can do this (currently with SQL, soon in the UI):

{% highlight AnyLanguage %}
UPDATE cluster SET active = false WHERE name = <WORKER_CLUSTER_NAME>
{% endhighlight %}

And to enable it again:

{% highlight AnyLanguage %}
UPDATE cluster SET active = true WHERE name = <WORKER_CLUSTER_NAME>
{% endhighlight %}

As you can see you only need minimal configuration and you can spin up
multiple InfraBox installations on different Kubernetes distributions,
versions and clouds and connect them all to one big CI infrastructure.

It has never been easier to run cross cloud continuous integration tasks.
