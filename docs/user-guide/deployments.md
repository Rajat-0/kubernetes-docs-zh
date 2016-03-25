---
---

* TOC
{:toc}

## What is a _Deployment_?

A _Deployment_ provides declarative updates for Pods and ReplicaSets.
You only need to describe the desired state in a Deployment object, and the deployment
controller will change the actual state to the desired state at a controlled rate for you.
You can define Deployments to create new resources, or replace existing ones
by new ones.

A typical use case is:

* Create a Deployment to bring up a replica set and pods.
* Later, update that Deployment to recreate the pods (for example, to use a new image).
* Rollback to an earlier Deployment revision if the current Deployment isn't stable. 
* Pause and resume a Deployment.

## Creating a Deployment

Here is an example Deployment. It creates a replica set to
bring up 3 nginx pods.

{% include code.html language="yaml" file="nginx-deployment.yaml" ghlink="/docs/user-guide/nginx-deployment.yaml" %}

Run the example by downloading the example file and then running this command:

```shell
$ kubectl create -f docs/user-guide/nginx-deployment.yaml --record
deployment "nginx-deployment" created
```

Setting the kubectl flag `--record` to `true` allows you to record current command in the annotations of the resources being created or updated. It will be useful for future introspection; for example, to see the commands executed in each Deployment revision.

Then running `get` immediately will give:

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
```

This indicates that the Deployment's number of desired replicas is 3 (according to deployment's `.spec.replicas`), the number of current replicas (`.status.replicas`) is 0, the number of up-to-date replicas (`.status.updatedReplicas`) is 0, and the number of available replicas (`.status.availableReplicas`) is also 0. 

Running the `get` again a few seconds later, should give:

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           18s
```

This indicates that the Deployment has created all three replicas, and all replicas are up-to-date (contains the latest pod template) and available (pod status is ready for at least deployment's `.spec.minReadySeconds`). Running `kubectl get rs` and `kubectl get pods` will show the replica set (RS) and pods created.

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   AGE
nginx-deployment-2035384211   3         3         18s 
```

You may notice that the name of the replica set is always `<the name of the Deployment>-<hash value of the pod template>`. 

```shell
$ kubectl get pods --show-labels
NAME                                READY     STATUS    RESTARTS   AGE       LABELS
nginx-deployment-2035384211-7ci7o   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-kzszj   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
nginx-deployment-2035384211-qqcnn   1/1       Running   0          18s       app=nginx,pod-template-hash=2035384211
```

The created replica set will ensure that there are three nginx pods at all times.

## Updating a Deployment

Suppose that we now want to update the nginx pods to start using the `nginx:1.9.1` image
instead of the `nginx:1.7.9` image.
For this, we update our deployment file as follows:

{% include code.html language="yaml" file="new-nginx-deployment.yaml" ghlink="/docs/user-guide/new-nginx-deployment.yaml" %}

We can then `apply` the new Deployment:

```shell
$ kubectl apply -f docs/user-guide/new-nginx-deployment.yaml
deployment "nginx-deployment" configured
```

Alternatively, we can `edit` the Deployment and change `.spec.template.spec.containers[0].image` from `nginx:1.7.9` to `nginx:1.9.1`:

```shell
$ kubectl edit deployment/nginx-deployment
deployment "nginx-deployment" edited
```

Running a `get` immediately will give:

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         0            3           20s
```

The 0 number of up-to-date replicas indicates that the deployment hasn't updated the replicas to the latest configuration. The current replicas indicates the total replicas (3 with old configuration and 0 with new configuration) this Deployment manages, and the available replicas indicates the number of current replicas that are available. 

The Deployment will update all the pods in a few seconds.

```shell
$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           36s
```

We can run `kubectl get rs` to see that the Deployment updated the pods by creating a new replica set and scaling it up to 3 replicas, as well as scaling down the old replica set to 0 replicas.

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   AGE
nginx-deployment-1564180365   3         3         6s
nginx-deployment-2035384211   0         0         36s
```

Running `get pods` should now show only the new pods:

```shell
$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
nginx-deployment-1564180365-khku8   1/1       Running   0          14s
nginx-deployment-1564180365-nacti   1/1       Running   0          14s
nginx-deployment-1564180365-z9gth   1/1       Running   0          14s
```

Next time we want to update these pods, we only need to update and re-apply the Deployment again.

Deployment can ensure that only a certain number of pods may be down while they are being updated. By
default, it ensures that at least 1 less than the desired number of pods are
up (1 max unavailable).

Deployment can also ensure that only a certain number of pods may be created above the desired number of pods. By default, it ensures that at most 1 more than the desired number of pods are up (1 max surge). 

For example, if you look at the above deployment closely, you will see that
it first created a new pod, then deleted some old pods and created new ones. It
does not kill old pods until a sufficient number of new pods have come up, and does not create new pods until a sufficient number of old pods have been killed. It makes sure that number of available pods is at least 2 and the number of total pods is at most 4.

```shell
$ kubectl describe deployments
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 12:01:06 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 updated | 3 total | 3 available | 0 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     <none>
NewReplicaSet:      nginx-deployment-1564180365 (3/3 replicas created)
Events:
  FirstSeen LastSeen    Count   From                     SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                     -------------   --------    ------              -------
  36s       36s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  23s       23s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  23s       23s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  23s       23s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  21s       21s         1       {deployment-controller }                 Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
```

Here we see that when we first created the Deployment, it created a replica set (nginx-deployment-2035384211) and scaled it up to 3 replicas directly.
When we updated the Deployment, it created a new replica set (nginx-deployment-1564180365) and scaled it up to 1 and then scaled down the old replica set to 2, so that at least 2 pods were available and at most 4 pods were created at all times.
It then continued scaling up and down the new and the old replica set, with the same rolling update strategy. Finally, we'll have 3 available replicas in the new replica set, and the old replica set is scaled down to 0.

### Multiple Updates

Each time a new deployment object is observed by the deployment controller, a replica set is
created to bring up the desired pods if there is no existing replica set doing so.
Existing replica set controlling pods whose labels match `.spec.selector` but whose
template does not match `.spec.template` are scaled down.
Eventually, the new replica set will be scaled to `.spec.replicas` and all old replica sets will
be scaled to 0.

If you update a Deployment while an existing deployment is in progress,
the Deployment will create a new replica set as per the update and start scaling that up, and
will roll the replica set that it was scaling up previously -- it will add it to its list of old replica sets and will
start scaling it down.

For example, suppose you create a Deployment to create 5 replicas of `nginx:1.7.9`,
but then updates the Deployment to create 5 replicas of `nginx:1.9.1`, when only 3
replicas of `nginx:1.7.9` had been created. In that case, Deployment will immediately start
killing the 3 `nginx:1.7.9` pods that it had created, and will start creating
`nginx:1.9.1` pods. It will not wait for 5 replicas of `nginx:1.7.9` to be created
before changing course.

## Rolling Back a Deployment

Sometimes we may want to rollback a Deployment; for example, when the Deployment is not stable, such as crash looping.

Suppose that we made a typo while updating the Deployment, by putting the image name as `nginx:1.91` instead of `nginx:1.9.1`:

{% include code.html language="yaml" file="bad-nginx-deployment.yaml" ghlink="/docs/user-guide/bad-nginx-deployment.yaml" %}

```shell
$ kubectl apply -f docs/user-guide/bad-nginx-deployment.yaml
deployment "nginx-deployment" configured
```

You will see that both the number of old replicas (nginx-deployment-1564180365 and nginx-deployment-2035384211) and new replicas (nginx-deployment-3066724191) are 2.

```shell
$ kubectl get rs
NAME                          DESIRED   CURRENT   AGE
nginx-deployment-1564180365   2         2         25s
nginx-deployment-2035384211   0         0         36s
nginx-deployment-3066724191   2         2         6s
```

Looking at the pods created, you will see that the 2 pods created by new replica set are stuck in an image pull loop.

```shell
$ kubectl get pods 
NAME                                READY     STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1       Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1       Running            0          25s
nginx-deployment-3066724191-08mng   0/1       ImagePullBackOff   0          6s
nginx-deployment-3066724191-eocby   0/1       ImagePullBackOff   0          6s
```

Note that the Deployment controller will stop the bad rollout automatically, and will stop scaling up the new replica set.

```shell
$ kubectl describe deployment
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       2 updated | 3 total | 2 available | 2 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     nginx-deployment-1564180365 (2/2 replicas created)
NewReplicaSet:      nginx-deployment-3066724191 (2/2 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  1m        1m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  22s       22s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  21s       21s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  13s       13s         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
```

To fix this, we need to rollback to a previous revision of Deployment that is stable. 

First, check the revisions of this deployment:

```shell
$ kubectl rollout history deployment/nginx-deployment
deployments "nginx-deployment":
REVISION    CHANGE-CAUSE
1           kubectl create -f docs/user-guide/nginx-deployment.yaml --record
2           kubectl apply -f docs/user-guide/new-nginx-deployment.yaml
3           kubectl apply -f docs/user-guide/bad-nginx-deployment.yaml
```

Because we recorded the command while creating this Deployment using `--record`, we can easily see the changes we made in each revision. 

To further see the details of each revision, run:

```shell
$ kubectl rollout history deployment/nginx-deployment --revision=2
deployments "nginx-deployment" revision 2
Labels:     app=nginx,pod-template-hash=1564180365
Annotations:    kubernetes.io/change-cause=kubectl apply -f docs/user-guide/new-nginx-deployment.yaml
Image(s):   nginx:1.9.1
No volumes.
```

Now we've decided to undo the current rollout and rollback to the previous revision:

```shell
$ kubectl rollout undo deployment/nginx-deployment
deployment "nginx-deployment" rolled back
```

Alternatively, you can rollback to a specific revision by specify that in `--to-revision`:

```shell
$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment "nginx-deployment" rolled back
```

The Deployment is now rolled back to a previous stable revision. As you can see, a `DeploymentRollback` event for rolling back to revision 2 is generated from Deployment controller. 

```shell
$ kubectl get deployment 
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           30m
$ kubectl describe deployment 
Name:           nginx-deployment
Namespace:      default
CreationTimestamp:  Tue, 15 Mar 2016 14:48:04 -0700
Labels:         app=nginx
Selector:       app=nginx
Replicas:       3 updated | 3 total | 3 available | 0 unavailable
StrategyType:       RollingUpdate
MinReadySeconds:    0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
OldReplicaSets:     <none>
NewReplicaSet:      nginx-deployment-1564180365 (3/3 replicas created)
Events:
  FirstSeen LastSeen    Count   From                    SubobjectPath   Type        Reason              Message
  --------- --------    -----   ----                    -------------   --------    ------              -------
  30m       30m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-2035384211 to 3
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 1
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-2035384211 to 0
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 2
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-3066724191 to 1
  29m       29m         1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-1564180365 to 2
  2m        2m          1       {deployment-controller }                Normal      ScalingReplicaSet   Scaled down replica set nginx-deployment-3066724191 to 0
  2m        2m          1       {deployment-controller }                Normal      DeploymentRollback  Rolled back deployment "nginx-deployment" to revision 2
  29m       2m          2       {deployment-controller }                Normal      ScalingReplicaSet   Scaled up replica set nginx-deployment-1564180365 to 3
```

## Pausing and Resuming a Deployment 

You can also pause a Deployment mid-way and then resume it. A use case is to support canary deployment. 

Update the Deployment again and then pause the Deployment with `kubectl rollout pause`:

```shell
$ kubectl apply -f docs/user-guide/new-nginx-deployment; kubectl rollout pause deployment/nginx-deployment
deployment "nginx-deployment" configured
deployment "nginx-deployment" paused
```

Note that any current state of the Deployment will continue its function, but new updates to the Deployment will not have an effect as long as the Deployment is paused. 

The Deployment was still in progress when we paused it, so the actions of scaling up and down replica sets are paused too. 

```shell
$ kubectl get rs 
NAME                          DESIRED   CURRENT   AGE
nginx-deployment-1564180365   2         2         1h
nginx-deployment-2035384211   2         2         1h
nginx-deployment-3066724191   0         0         1h
```

To resume the Deployment, simply do `kubectl rollout resume`:

```shell
$ kubectl rollout resume deployment/nginx-deployment
deployment "nginx-deployment" resumed
```

Then the Deployment will continue and finish the rollout:

```shell
$ kubectl get rs 
NAME                          DESIRED   CURRENT   AGE
nginx-deployment-1564180365   3         3         1h
nginx-deployment-2035384211   0         0         1h
nginx-deployment-3066724191   0         0         1h
```

Note: A paused Deployment cannot be scaled at this moment, and we will add this feature in 1.3 release, see [issue #20853](https://github.com/kubernetes/kubernetes/issues/20853). You cannot rollback a paused Deployment either, and you should resume a Deployment first before doing a rollback. 

## Writing a Deployment Spec

As with all other Kubernetes configs, a Deployment needs `apiVersion`, `kind`, and
`metadata` fields.  For general information about working with config files,
see [deploying applications](/docs/user-guide/deploying-applications), [configuring containers](/docs/user-guide/configuring-containers), and [using kubectl to manage resources](/docs/user-guide/working-with-resources) documents.

A Deployment also needs a [`.spec` section](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/docs/devel/api-conventions.md#spec-and-status).

### Pod Template

The `.spec.template` is the only required field of the `.spec`.

The `.spec.template` is a [pod template](/docs/user-guide/replication-controller/#pod-template).  It has exactly
the same schema as a [pod](/docs/user-guide/pods), except it is nested and does not have an
`apiVersion` or `kind`.

### Replicas

`.spec.replicas` is an optional field that specifies the number of desired pods. It defaults
to 1.

### Selector

`.spec.selector` is an optional field that specifies label selectors for pods
targeted by this deployment. Deployment kills some of these pods, if their
template is different than `.spec.template` or if the total number of such pods
exceeds `.spec.replicas`. It will bring up new pods with `.spec.template` if
number of pods are less than the desired number.

### Strategy

`.spec.strategy` specifies the strategy used to replace old pods by new ones.
`.spec.strategy.type` can be "Recreate" or "RollingUpdate". "RollingUpdate" is
the default value.

#### Recreate Deployment

All existing pods are killed before new ones are created when
`.spec.strategy.type==Recreate`.

#### Rolling Update Deployment

The Deployment updates pods in a [rolling update](/docs/user-guide/update-demo/) fashion
when `.spec.strategy.type==RollingUpdate`.
You can specify `maxUnavailable` and `maxSurge` to control
the rolling update process.

##### Max Unavailable

`.spec.strategy.rollingUpdate.maxUnavailable` is an optional field that specifies the
maximum number of pods that can be unavailable during the update process.
The value can be an absolute number (e.g. 5) or a percentage of desired pods
(e.g. 10%).
The absolute number is calculated from percentage by rounding up.
This can not be 0 if `.spec.strategy.rollingUpdate.maxSurge` is 0.
By default, a fixed value of 1 is used.

For example, when this value is set to 30%, the old replica set can be scaled down to
70% of desired pods immediately when the rolling update starts. Once new pods are
ready, old replica set can be scaled down further, followed by scaling up the new replica set,
ensuring that the total number of pods available at all times during the
update is at least 70% of the desired pods.

##### Max Surge

`.spec.strategy.rollingUpdate.maxSurge` is an optional field that specifies the
maximum number of pods that can be created above the desired number of pods.
Value can be an absolute number (e.g. 5) or a percentage of desired pods
(e.g. 10%).
This can not be 0 if `MaxUnavailable` is 0.
The absolute number is calculated from percentage by rounding up.
By default, a value of 1 is used.

For example, when this value is set to 30%, the new replica set can be scaled up immediately when
the rolling update starts, such that the total number of old and new pods do not exceed
130% of desired pods. Once old pods have been killed,
the new replica set can be scaled up further, ensuring that the total number of pods running
at any time during the update is at most 130% of desired pods.

### Min Ready Seconds

`.spec.minReadySeconds` is an optional field that specifies the
minimum number of seconds for which a newly created pod should be ready
without any of its containers crashing, for it to be considered available.
This defaults to 0 (the pod will be considered available as soon as it is ready).
To learn more about when a pod is considered ready, see [Container Probes](/docs/user-guide/pod-states/#container-probes).

### Rollback To 

`.spec.rollbackTo` is an optional field with the configuration the Deployment is rolling back to. Setting this field will trigger a rollback, and this field will be cleared every time a rollback is done. 

#### Revision

`.spec.rollbackTo.revision` is an optional field specifying the revision to rollback to. This defaults to 0, meaning rollback to the last revision in history. 

### Revision History Limit

`.spec.revisionHistoryLimit` is an optional field that specifies the number of old replica sets to retain to allow rollback. All old replica sets will be kept by default, if this field is not set. The configuration of each Deployment revision is stored in its replica sets; therefore, once an old replica set is deleted, you lose the ability to rollback to that revision of Deployment. 

### Paused

`.spec.paused` is an optional boolean field for pausing and resuming a Deployment. It defaults to false (a Deployment is not paused). 

## Alternative to Deployments

### kubectl rolling update

[Kubectl rolling update](/docs/user-guide/kubectl/kubectl_rolling-update) updates pods and replication controllers in a similar fashion.
But deployments is recommended, since it's declarative and is server side, and has more features, such as rolling back to any previous revision even after the rolling update is done. Also, replica sets supersede replication controllers. 