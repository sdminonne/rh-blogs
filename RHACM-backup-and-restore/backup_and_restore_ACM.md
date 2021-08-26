# Backup and restore Red Hat Advanced Cluster Management for Kubernetes

**Author**: Salvatore Dario Minonne

## Introduction

Red Hat Advanced Cluster Management for Kubernetes (RHACM) supplies the ability to manage fleets of Kubernetes and Openshift clusters. The RHACM model consists of a central controller plane of OpenShift that runs in a RHACM cluster (known as the hub cluster), and several managed clusters where the workloads run. <!--The RHACM model consists of one control plane OpenShift cluster (HUB in the following) and several managed clusters where the workloads run. ARE YOU STATING THAT OCP IS THE HB CLUSTER OR THAT RHACM IS THE HUB CLUSTER? Dario: I'm stating that RHACM is the Hub cluster. RHACM runs in Openshift. --> <!--For this blog, Red Hat OpenShift Container Platform is the hub cluster.--> The RHACM model is inspired by the ubiquitous _two-layer model_, which includes the following components:

 * Kubernetes control plane and compute nodes
 * SDN control and data planes

DevOps uses this same model to work with a two-level architecture, increasing understanding and finally predictability. At the same time, the simplicity and the two-layer approach increases robustness.

Unfortunately, the single pane of glass is irreparably linked to the "single point of failure" problem. Recently, RHACM users often report the need to back up the control plane configurations for a quick recovery, in case of an outage. 

While most of the configuration can be recreated from scratch using the [GitOps](https://www.redhat.com/en/topics/devops/what-is-gitops) approach, at the end of a backup procedure the managed cluster fleet is not correctly registered in the new hub cluster. The goal of this article is to show how RHACM managed clusters configurations can be restored. In this blog, I use only common Unix command (`bash`) and the OpenShift (`oc`) client. 
As a backup and restore solution, I use [Velero](https://velero.io/), the de-facto standard to back up a Kubernetes cluster.


### Disaster Recovery foundations

The ability to back up, restore, and re-register managed clusters to RHACM, lays the foundation for an active-passive [Disaster Recovery](https://en.wikipedia.org/wiki/Disaster_recovery) (DR) solution. Ideally, an organization can back up the hub cluster configuration at some frequency, restoring the configurations elsewhere in case of an outage. Obviously, a full DR solution is well beyond the scope of this article, and a more robust solution is needed even to start thinking about DR. There are too many parameters that can impact _business continuity_ and every organization has to carefully consider  what, how, and when the configuration should be backed up and restored, to minimize your [Recovery Time Objective  (RTO)](https://www.forbes.com/sites/sungardas/2015/04/30/like-the-nfl-draft-is-the-clock-the-enemy-of-your-recovery-time-objective/) and [Recovery Point Objective (RPO)](https://www.acronis.com/en-us/articles/rto-rpo/).

## Velero

Velero is an open-source tool to back up and restore a Kubernetes cluster. Velero follows the operator pattern reacting to the creation of custom resources. The custom resources used in this article are `backups.velero.io ` and `restores.velero.io`, but Velero exposes more than 10 custom resources to fully automate more complex workflows.

Velero needs to be installed to back up and restore the cluster and restore. As I restore the backup in another cluster, keep in mind that Velero has to be installed on both clusters.

Velero saves the resources as manifest files in a tarball format and it stores the tarball in S3 object storage. Velero supports all major S3 providers, but in this article I use AWS S3.

I demonstrate how to deploy Velero through its CLI, but it can be installed in other ways, for example through [helm](https://artifacthub.io/packages/helm/vmware-tanzu/velero#using-helm-3), as a usual deployment, or through the Operator Hub. Whatever the installation method, Velero needs the S3 bucket name, the backup storage and the volume snapshot location region, and obviously the S3 credentials.


## The configuration

Our _fleet_ is composed of only one managed cluster named `managed-one`. Having several managed clusters doesn't change the fundamental ideas that are present in this article, <!--what are the "fundamental ideas"? I would update the sentence to read: Having several managed clusters require a `loop` through the bash commands and potentially the need to `wait until` each command is finished.--> it only requires a `loop` through the bash commands and potentially the need to wait until each command is succesfully finished <!-- Dario -->

<!--it seems like this image should be after the command and the results--><!--![Cluster management image](https://i.imgur.com/0WlqaBZ.png)-->

Using the OpenShift clien (`oc`) on `hub-dr1` cluster, I can verify the managed clusters that are available on the `hub-dr1` cluster from the command line interface (CLI):

```shell
$ oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        5d1h
managed-one     true                                  True     True        4d23h
```

View the following console image of the accepted clusters:

![Cluster management image](https://i.imgur.com/0WlqaBZ.png)

As previously mentioned, Velero must be installed through the CLI. Be sure to supply the S3 credentials. For this article, AWS S3 configuration need to have the following file information:

```shell
$ cat credentials-velero
[default]
aws_access_key_id = <MY AWS ACCESS KEY ID>
aws_secret_access_key = <MY AWS SECRET ACCESS KEY>
```

Now, run the following command to install Velero:

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

The `install` command creates the `velero` namespace and installs all that is needed by Velero. I've removed most of the all the CRD creation leaving only `restores.velero.io` and `backups.velero.io`. The `install` command creates the `velero` namespace and runs few `velero` resources, along with the necessary RBAC resources. You can check which resources are running with the following command:

```shell
$ oc get all -n velero
NAME                          READY   STATUS    RESTARTS   AGE
pod/velero-658979bddd-6qgjk   1/1     Running   0          7m49s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velero   1/1     1            1           7m50s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/velero-658979bddd   1         1         1       7m50s
```

A simple way to check whether `Velero` can connect to S3 storage can be done by running the `grep available` command in the logs:

```shell
$ oc logs deployment/velero -n velero | grep available
...
time="2021-08-01T20:24:56Z" level=info msg="Backup storage location valid, marking as available" backup-storage-location=default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:121"
...
```

After Velero is up and running, the backup command can be run.

## The backup process

The backup process is performed by backing up all of the main RHACM namespaces and all the managed cluster namespaces, in this case only the `managed-one`. Avoid backing up cluster-scoped resources to minimize the amount of data to backup. When you avoid backing up the cluster-scoped resources, _restore_ is impacted because `clusterroles` and `clusterrole-bindings` need to be recreated. The managed cluster in RHACM is represented by a namespace (backed up) <!--is the namespace named backed-up or are you informing the reader that the namespace is backed up? Dario: It's the namespace that it's backed up, not the name. --> and by a custom resource instance of `managedclusters.cluster.open-cluster-management.io` CRD <!-- Dario: CR is the instance of CRD (D for definitions). `managedclusters.cluster.open-cluster-management.io` is the full name of the CRD -->.

Use the `velero` CL to run the backup command:

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

At the end of the backup process, the backups (only one currently) that have the `Completed` status without errors or warnings are listed. Run the following command:

```shell
$ velero backup get -n velero
NAME              STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
acm-backup-blog   Completed   0        0          2021-08-02 12:12:08 +0200 CEST   29d       default            <none>
```

View the following image of the _S3 console_. Notice that the `acm-backup-log` is currently present in the _S3 console_:
![Amazon S3 image](https://i.imgur.com/PgnhKsz.png)

For the sake of this article, the backup process is finished. As already mentioned in a _production environment_, there is definitively more configurations needed for a _production environment_. Generally, the following configurations can be set:

  * Error handling
  * Backup frequency
  * S3 storage space handling
  * Encrypting data at rest

This list can (and should) continue to grow, according to the specific use cases.

---

## The restore process

At this point, let's assume that the current hub cluster, `dr-hub1`, is severely impacted by a _disaster_. In this case, another hub cluster (`dr-hub2`) is used to restore the data and to re-register the managed cluster (`managed-one`). To maintain simplicity in this article, let's assume the RHACM is already installed on `dr-hub2`.

To proceed in the restore process, `velero` needs to be installed, configuring the S3 storage in the same way:

```shell
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.2.0 \
  --bucket acm-dr-blog \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file  credentials-velero
```

When `velero` is available the backup CRs <!-- Dario: CRs not CRDs --> should also be available on the `dr-hub2` hub cluster. Run the following command to verify:

```shell
$ velero backup get
NAME              STATUS      ERRORS   WARNINGS   CREATED                          EXPIRES   STORAGE LOCATION   SELECTOR
acm-backup-blog   Completed   0        0          2021-08-02 12:12:08 +0200 CEST   29d       default            <none>
```

You can also use the OpenShift client to check for the CRDs in the `velero` namespace:

```shell
$ oc get backups -n velero
NAME              AGE
acm-backup-blog   58m
```

Now, restore the managed cluster in `dr-hub2` using the `Velero` CLI. Run the following command:

```shell
velero restore create --from-backup acm-backup-blog
Restore request "acm-backup-blog-20210802182417" submitted successfully.
Run `velero restore describe acm-backup-blog-20210802182417` or `velero restore logs acm-backup-blog-20210802182417` for more details.
$ velero restore get
NAME                             BACKUP            STATUS      STARTED                          COMPLETED                        ERRORS   WARNINGS   CREATED                          SELECTOR
acm-backup-blog-20210802182417   acm-backup-blog   Completed   2021-08-02 18:24:17 +0200 CEST   2021-08-02 18:24:57 +0200 CEST   0        56         2021-08-02 18:24:17 +0200 CEST   <none>
```

At the end of the restore process, <!--even without errors as in our case, Dario: we don't backup in case of errors (the normal situation). --> verify the creation of the managed cluster namespace and verify that there are no managed cluster registered to your hub cluster:

```shell
$ oc get ns managed-one --show-labels=true
NAME          STATUS   AGE     LABELS
managed-one   Active   5m37s   cluster.open-cluster-management.io/managedCluster=managed-one
$ oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        5h11m
```

Since this is not a real DR scenario, let's push our analysis a little further.

* Taking a look at the consoles respectively, notice that the `managed-one` cluster is still registered to the `dr-hub1` hub cluster:

![Image of cluster management table from the console](https://i.imgur.com/Nr5vZzX.png)

* Even though the managed cluster is not registered it appears in the console:
![Image of failed managed cluster](https://i.imgur.com/gHIYsSM.png)

This error occurs because the console detects the presence of the `managed-one` namespace, but it returns the failure since it cannot fetch any information from the `managedcluster` CRD.

### Registering the managed cluster

The current solution to register each managed cluster consists in having each managed cluster `registration` operator register to the new Hub. Each managed cluster `registration` operator watches the `bootstrap-hub-kubeconfig` secret inside `open-cluster-management-agent` namespace. The general idea is:
 1.  Generate the new HUB kubeconfig (dr-hub2 in our case)
 2.  Fetch admin-kubeconfig secret from the managed cluster namespace.
 3.  Supplying the authorization to the `registration` operator to register the managed cluster
 4.  Using kubeconfig fetched at point 2 to replace the `boostrap-hub-kubeconfig` with kubeconfig generated at point 1


Let's start by generating the new Hub kubeconfig.  We're going to generate it through an [heredoc](https://en.wikipedia.org/wiki/Here_document). Assuming `oc` OpenShift client is pointing to the new Hub we can generate through this series of commands:

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

Now that we have the `newbootstraphub.kubeconfig` we can replace `bootrap-hub-kubeconfig`.  But before that we have to fetch the `managed-one` kubeconfig secret just restored through velero. Pointing `oc` to the new Hub we have to get the `admin-kubeconfig` secret inside the managed cluster namespace. 
Running the command and filtering through `admin-kubeconfig` we get `managed-one-0-qg8wm-admin-kubeconfig`:

```shell=
$ oc get secret -o name -n $managedclustername  | grep admin-kubeconfig
secret/managed-one-0-qg8wm-admin-kubeconfig
```

So to automate a little bit more we can simply fetch the kubeconfig through the following commands:

```shell=
$ managed_kubeconfig_secret=$(basename $(oc get secret -o name -n $managedclustername | grep admin-kubeconfig))
$ get secret $managed_kubeconfig_secret -n ${managedclustername} -o jsonpath={.data.kubeconfig} | base64 -d > managedcluster.kubeconfig
```
We can test the `managedcluster.kubeconfig` file through any command, but we can try to get the `boostrap-hub-kubeconfig` since we need to replace it:

```shell=
$  oc --kubeconfig=managedcluster.kubeconfig get secret bootstrap-hub-kubeconfig -n open-cluster-management-agent
NAME                       TYPE     DATA   AGE
bootstrap-hub-kubeconfig   Opaque   1      6d16h
```

Since the managed cluster `registration` operator needs rights to register the managed clusters we have to supply them. Remember we did not back up cluster scope resources and the rights are cluster scoped resources. Let's use heredoc again:

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

Now let's have a look at the UI. The `dr-hub2` UI simply displays `managed-one` cluster as not (yet) accepted:

![](https://i.imgur.com/2fnnKe6.png)

Since the `dr-hub2` is still up and running (luckily no disaster, at least not today) we can also take a look at `dr-hub1`. It displays `managed-one` as Offline:

![Cluster ](https://i.imgur.com/zbj4GcA.png)

### Accepting `managed-one` in the new HUB

We need to manually accept the `managed-one` cluster in `dr-hub2` HUB using the command like this:

```shell=
 oc patch managedcluster ${managedclustername} -p='{"spec":{"hubAcceptsClient":true}}' --type=merge
```

And verify that the managed cluster has joined:

```shell=
oc get managedclusters
NAME            HUB ACCEPTED   MANAGED CLUSTER URLS   JOINED   AVAILABLE   AGE
local-cluster   true                                  True     True        24h
managed-one     true                                  True     True        104m
```

And now the `dr-hub2` ACM UI correctly reports:

![](https://i.imgur.com/As5Uh6o.png) <!--similar screenshot as 0WlqaBZ in line 39: Dario: yeah! similar but not the same. 0WlqaBZ was for original HUB this one is for the new HUB with same (restored) data. See the URLs -->

## Conclusions

I have demonstrated how managed cluster configurations can be backed up and restored in a new hub cluster without affecting other pods, except for the RHACM pods in the managed cluster. As a reminder, the RHACM pods restart due to the replacement of `boostrap-hub-kubeconfig`.

I want to give a special thanks to Zachary Kayyali, David Schmidt, and Christine Rizzo for reviewing and contributing to this blog. Another special thanks go to Chris Doan for the reviews, insights, and for being the best teammate one would like to have.






