# Backup and restore Red Hat Advanced Cluster Management for Kubernetes


## Introduction

RedHat Advanced Cluster Management for Kubernetes (RHACM in the following) supplies the ability to manage fleets of Kubernetes and Openshift clusters. RHACM model consists of one control plane Openshift cluster (HUB in the following) and several managed clusters where the workloads run. RHACM model as inspired by the ubiquitous _two layers model_ for example:
 * Kubernetes control plane and compute nodes
 * SDN control and data planes

DevOps use to work with two levels-architecture, increasing understanding, and finally predictability.
At the same time, with its simplicity, the two layers approach increases robustness.

Unluckily the single pane of glass is irreparably linked to the "single point of failure" problem. In the last months, understandably, RHACM users keep reporting the need of backing up the control plane configurations to recover quickly in case of an outage. 

### DR foundations

The ability to backup, restore RHACM, and re-registering the managed clusters lays the foundations for an Active-Passive [Disaster Recovery](https://en.wikipedia.org/wiki/Disaster_recovery) (DR) solution. Ideally, an organization could backup the HUB configuration with some frequency, restoring the configurations elsewhere in case of an outage. Obviously, a full DR solution is well beyond the scope of this article, and a more robust solution is needed even to start thinking about DR, too many parameters could impact _business continuity_ and every organization has to consider carefully what, how, and when should be backed-up and restored to minimize [RTO](https://www.forbes.com/sites/sungardas/2015/04/30/like-the-nfl-draft-is-the-clock-the-enemy-of-your-recovery-time-objective/) and RPO.

### This blog
While most of the configuration could be re-created from scratch through, for example,  [GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops) approach, at the end of a backup procedure the managed cluster fleet is not correctly registered in the new HUB. The goal of this article is to show how RHACM managed clusters configurations could be restored. We're going to use only common Unix command (`bash`) and the Openshift `oc` client. 
As backup and restore solution we use [Velero](https://velero.io/): the de-facto standard to backup a Kubernetes cluster.


## Velero

Velero is an open-source tool to backup and restore a Kubernetes cluster. Velero follows the operator pattern reacting to the creation of some custom resources. The custom resources used in this article are `backups.velero.io ` and `restores.velero.io` but Velero exposes more than 10 custom resources to fully automate more complex workflows.

We need to install Velero to back up the cluster and restore the cluster as well. For this article, as we restore the backup in another cluster, Velero has to be installed on both clusters.

Velero saves the resources as manifest files in a tarball format and it stores the tarball in S3 storage. It supports all major S3 providers, in this article we're going to use AWS S3.

We're going to deploy `Velero` through its CLI but it can be installed in other ways, for example through [helm](https://artifacthub.io/packages/helm/vmware-tanzu/velero#using-helm-3) or as a usual deployment or through OperatorHub.  Whatever the installation method, velero needs the S3 bucket name, the backup storage and the volume snapshot location region, and obviously the S3 credentials.


## The configuration

Our _fleet_ is composed of only one managed cluster `managed-one`. Having several managed clusters doesn't change the fundamental ideas we present in this article while it may over-complexify the bash commands through `loop` and potentially the need to `wait until` each command to be finished. 

![](https://i.imgur.com/0WlqaBZ.png)

Pointing the `oc` openshift client to `hub-dr1` cluster we can report the managed clusters:

```shell
$ oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        5d1h
managed-one     true                                  True     True        4d23h
```

As already mentioned we need `Velero` so we install it through the CLI. We need to supply the S3 credentials and since in this article we do use AWS S3 we need to supply this file:

```shell
$ cat credentials-velero
[default]
aws_access_key_id = <MY AWS ACCESS KEY ID>
aws_secret_access_key = <MY AWS SECRET ACCESS KEY>
```
Now we can proceed and install `Velero`:

```shell
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.2.0 \
  --bucket acm-dr-blog \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file  credentials-velero
[...] # output not reported
CustomResourceDefinition/backups.velero.io: created
[...] # output not reported 
CustomResourceDefinition/restores.velero.io: created
[...] # output not reported
Namespace/velero: created
[...] # output not reported
ClusterRoleBinding/velero: created
[...] # output not reported
ServiceAccount/velero: created
[...] # output not reported
Secret/cloud-credentials: created
[...] # output not reported
Deployment/velero: created
Velero is installed! ⛵ Use 'kubectl logs deployment/velero -n velero' to view the status.
```

The `install` command creates the `velero` namespace and installs all what is needed by Velero. I've removed most of the all the CRDs creation leaving only `restores.velero.io` and `backups.velero.io`. `install` command creates the `velero` namespace and few `Velero` running resources plus the necessary RBACs resources. One can check the running resources through the command:


```shell
$ oc get all -n velero
NAME                          READY   STATUS    RESTARTS   AGE
pod/velero-658979bddd-6qgjk   1/1     Running   0          7m49s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velero   1/1     1            1           7m50s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/velero-658979bddd   1         1         1       7m50s
```

A simple way to check whether `Velero` can connect to S3 storage could be through running `grep available` in the logs:

```shell
$ oc logs deployment/velero -n velero | grep available
...
time="2021-08-01T20:24:56Z" level=info msg="Backup storage location valid, marking as available" backup-storage-location=default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:121"
...
```

As soon `Velero` is up and running, we can run the backup command.

## The backup process

The backup process is performed backing-up all the main RHACM namespaces and all the managed clusters namespaces, in this case only the `managed-one`. We don't backup cluster scoped resources to minimize the amount of data to backup. Avoiding the backup of cluster scoped resources will have some impact during the _restore_ since we'll need to recreate `clusterroles` and `clusterrole-bindings`. The managed cluster in RHACM is represented by a namespace (backed-up) and by a custom resource `managedclusters.cluster.open-cluster-management.io`.

We execute backup through `velero` CLI:

```shell
$ velero backup create acm-backup-blog \
  --wait \ 
  --include-cluster-resources=false \
  --exclude-resources nodes,events,certificatesigningrequests \ 
  --include-namespaces managed-one,hive,openshift-operator-lifecycle-manager,open-cluster-management-agent,open-cluster-management-agent-addon
[...]
Backup request "acm-backup-blog" submitted successfully.
Waiting for backup to complete. You may safely press ctrl-c to stop waiting - your backup will continue in the background.
......................................................
Backup completed with status: Completed. You may check for more information using the commands `velero backup describe acm-backup-blog` and `velero backup logs acm-backup-blog`.
```

At the end of the backup process, we list the backups (only one currently), in `Completed` status without errors or warnings.

```shell
$ velero backup get -n velero
NAME              STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
acm-backup-blog   Completed   0        0          2021-08-02 12:12:08 +0200 CEST   29d       default            <none>
```

In the `S3 Console` we can see the `acm-backup-log` currently present in the `S3 Console`
![](https://i.imgur.com/PgnhKsz.png)

For the sake of this article, the backup process is finished. As already mentioned in a _production environment_ definitively more is needed. Generally we could cite :
 * error handling
 * backup frequency
 * S3 storage space handling
 * Encrypting data at rest

the list could (and should) continue accordingly to the specific use case.

---

## The restore process

At this point, we should assume the current Hub `dr-hub1` is severely impacted by a _disaster_. So we operate on another Hub (`dr-hub2`) to restore the data and to re-register the managed cluster `managed-one`. To simplify this article, I assume ACM is already installed on `dr-hub2`.

To proceed on the restore we need to install `Velero` configuring the S3 storage in the same way

```shell
```shell
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.2.0 \
  --bucket acm-dr-blog \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file  credentials-velero
```

As soon the `Velero` is available the backups CRs should be available also on `dr-hub2`

```shell
$ velero backup get
NAME              STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
acm-backup-blog   Completed   0        0          2021-08-02 12:12:08 +0200 CEST   29d       default            <none>
```

or through the usual Openshift client

```shell
$ oc get backups -n velero
NAME              AGE
acm-backup-blog   58m
```

Now we can restore the managed cluster in `dr-hub2`, we're going to use `Velero` CLI again:

```shell
velero restore create --from-backup acm-backup-blog
Restore request "acm-backup-blog-20210802182417" submitted successfully.
Run `velero restore describe acm-backup-blog-20210802182417` or `velero restore logs acm-backup-blog-20210802182417` for more details.
$ velero restore get
NAME                             BACKUP            STATUS      STARTED                          COMPLETED                        ERRORS   WARNINGS   CREATED                          SELECTOR
acm-backup-blog-20210802182417   acm-backup-blog   Completed   2021-08-02 18:24:17 +0200 CEST   2021-08-02 18:24:57 +0200 CEST   0        56         2021-08-02 18:24:17 +0200 CEST   <none>
```

At the end of the restore process, even without errors as in our case, we can verify the creation of the managed cluster namespace but no managed cluster registered.

```shell
$ oc get ns managed-one --show-labels=true
NAME          STATUS   AGE     LABELS
managed-one   Active   5m37s   cluster.open-cluster-management.io/managedCluster=managed-one
$ oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        5h11m
```
Since this is not a real Disaster Recovery scenario we can push our analysis a little further:
looking at the UIs respectively we find the the `managed-one` cluster is still registered to the `dr-hub1` Hub.

![](https://i.imgur.com/Nr5vZzX.png)

Even if the managed clusters is not registered it appears in the UI.
![](https://i.imgur.com/gHIYsSM.png)

UI detects the presence of the `managed-one` namespace but it returns the failure since it cannot fetch any information from `managedcluster` CRD.

### Registering the managed cluster

The current solution to register each managed cluster consists in having each managed cluster `registration` operator  register to the new Hub. Each managed cluster `registration` operator watches the `bootstrap-hub-kubeconfig` secret inside `open-cluster-management-agent` namespace. The general idea is:
 1.  Generate the new HUB kubeconfig (dr-hub2 in our case)
 2.  Fetch admin-kubeconfig secret from the managed cluster namespace.
 3.  Supplying the authorization to the `registration` operator to register the managed cluster
 4.  Using kubeconfig fetched at point 2 to replace the `boostrap-hub-kubeconfig` with kubeconfig generated at point 1


Let's start by generate the new Hub kubeconfig.  We're going to generate it through an [heredoc](https://en.wikipedia.org/wiki/Here_document). Assuming `oc` openshift client is pointing to the new Hub we can generate through this series of commands.

```bash=
managedclustername=managed-one
server=$(oc config view -o jsonpath='{.clusters[0].cluster.server}')
secretname=$(oc get secret -o name -n $managedclustername | grep ${managedclustername}-bootstrap-sa-token)
ca=$(oc get ${secretname} -n ${managedclustername} -o jsonpath='{.data.ca\.crt}')
token=$(oc get ${secretname} -n ${managedclustername} -o jsonpath='{.data.token}' | base64 --decode)

cat << EOF > newbootstraphub.kubeconfig
apiVersion: v1
kind: Config
clusters:
- name: default-cluster
  cluster:
    certificate-authority-data: ${ca}
    server: ${server}
contexts:
- name: default-context
  context:
    cluster: default-cluster
    namespace: default
    user: default-auth
current-context: default-context
users:
- name: default-auth
  user:
    token: ${token}
EOF
```

Now that we have the `newbootstraphub.kubeconfig` we can replace `bootrap-hub-kubeconfig` but beforehand we have to fetch the `managed-one` kubeconfig secret just restored through velero. Pointing `oc` to the new Hub we have to get the `admin-kubeconfig` secret inside the managed cluster namespace. 
Running the command and filtering through `admin-kubeconfig` we get `managed-one-0-qg8wm-admin-kubeconfig`

```shell=
$ oc get secret -o name -n $managedclustername  | grep admin-kubeconfig
secret/managed-one-0-qg8wm-admin-kubeconfig
```

so to automate a little bit we can simply fetch the kubeconfig through the following commands

```shell=
$ managed_kubeconfig_secret=$(basename $(oc get secret -o name -n $managedclustername | grep admin-kubeconfig))
$ get secret $managed_kubeconfig_secret -n ${managedclustername} -o jsonpath={.data.kubeconfig} | base64 -d > managedcluster.kubeconfig
```
We can test the `managedcluster.kubeconfig` file through any command, but we can try to get the `boostrap-hub-kubeconfig` since we need to replace it.

```shell=
$  oc --kubeconfig=managedcluster.kubeconfig get secret bootstrap-hub-kubeconfig -n open-cluster-management-agent
NAME                       TYPE     DATA   AGE
bootstrap-hub-kubeconfig   Opaque   1      6d16h
```

Since the managed cluster `registration` operator needs rights to register the managed clusters we have to supply them. Remember we did not backup cluster scope resources and the rights are cluster scoped resources. Let's use heredoc again:

```bash=
cat << EOF | oc  apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:open-cluster-management:managedcluster:bootstrap:${managedclustername}
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - create
  - get
  - list
  - watch
- apiGroups:
  - cluster.open-cluster-management.io
  resources:
  - managedclusters
  verbs:
  - get
  - create
EOF

cat << EOF | oc  apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  managedFields:
  name: system:open-cluster-management:managedcluster:bootstrap:${managedclustername}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:open-cluster-management:managedcluster:bootstrap:${managedclustername}
subjects:
- kind: ServiceAccount
  name: ${managedclustername}-bootstrap-sa
  namespace: ${managedclustername}
EOF

cat << EOF | oc  apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: open-cluster-management:managedcluster:${managedclustername}
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests
  verbs:
  - create
  - get
  - list
  - watch
- apiGroups:
  - register.open-cluster-management.io
  resources:
  - managedclusters/clientcertificates
  verbs:
  - renew
- apiGroups:
  - cluster.open-cluster-management.io
  resourceNames:
  - ${managedclustername}
  resources:
  - managedclusters
  verbs:
  - get
  - list
  - update
  - watch
- apiGroups:
  - cluster.open-cluster-management.io
  resourceNames:
  - ${managedclustername}
  resources:
  - managedclusters/status
  verbs:
  - patch
  - update
EOF

cat << EOF | oc  apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: open-cluster-management:managedcluster:${managedclustername}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: open-cluster-management:managedcluster:${managedclustername}
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:open-cluster-management:${managedclustername}
EOF
```

Now we can replace `boostrap-hub-kubeconfig`:

```shell=
oc --kubeconfig=managedcluster.kubeconfig delete secret bootstrap-hub-kubeconfig -n open-cluster-management-agent
oc --kubeconfig=managedcluster.kubeconfig create secret generic bootstrap-hub-kubeconfig --from-file=kubeconfig=newbootstraphub.kubeconfig -n open-cluster-management-agent
```
Replacing the `boostrap-hub-kubeconfig` trigger the refresh of `pod/klusterlet-registration-agent` and `pod/klusterlet-work-agent`. When these pods restart, the managed cluster will appear in the list of the managed clusters.

```shell=
$ oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        23h
managed-one     false                                          True        7m1s
```

Let's have a look to the UI, the `dr-hub2` UI simply display `managed-one` cluster not (yet) accepted

![](https://i.imgur.com/2fnnKe6.png)

Since the `dr-hub2` is still up and running (luckily no disaster, at least today) we can also take a look to `dr-hub1`. It displays `managed-one` as Offline.

![](https://i.imgur.com/zbj4GcA.png)

### Accepting `managed-one` in the new HUB

We must manually accept the `managed-one` cluster in `dr-hub2` HUB using the command like this:

```shell=
 oc patch managedcluster ${managedclustername} -p='{"spec":{"hubAcceptsClient":true}}' --type=merge
```

And verify that the managed cluster has joined.

```shell=
oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        24h
managed-one     true                                  True     True        104m
```

And now the `dr-hub2` ACM UI correctly reports:

![](https://i.imgur.com/As5Uh6o.png)

## Conclusions

We've seen how a managed cluster configurations can be backed up and restored in a new Hub without affecting other PODs except for the RHACM pods in the managed cluster (restarting due to `boostrap-hub-kubeconfig` replacing).

The author would like to thanks Zachary Kayyali, David Schmidt, Christine Rizzo for reviewing and contributing to this blog. A special thanks go to Chris Doan for reviews and insights and for being the best teammate one would like to have.






