# NFS-to-ODF-Migration
# Playbook for Migration of Cloud Pak for Data from NFS to ODF

## Runbook Version

Runbook Version: __1.5.1__ 




# Table of Contents

- [Cloud Pak for Data operators backup and restore](#Cloud-pak-for-data-operators-backup-and-restore)
- [MTC installation and setup on source and target clusters](#MTC-installation-and-setup-on-source-and-target-clusters)
- [MTC migration plan creation](#MTC-migration-plan-creation)
- [Shutdown Cloud Pak for Data instance on source cluster](#Shutdown-cloud-pak-for-data-instance-on-source-cluster)
- [MTC migration cutover to target cluster](#MTC-migration-cutover-to-target-cluster)
- [Patch and update new storage class references on target cluster](#Patch-and-update-new-storage-class-references-on-target-cluster)
- [Start Cloud Pak for Data instance on target cluster](#Start-Cloud-Pak-for-Data-instance-on-target-cluster)
- [CPD Troubleshooting](#CPD-Troubleshooting)
- [Migration Toolkit Lessons learned and troubleshooting](#Migration-Toolkit-Lessons-learned-and-troubleshooting)

---

This migration playbook requires the `cpd-migrate.sh` and `cpd-migrate.input.json` (also referenced as input.json) which will also be provided alongside this playbook.

Additionally, this playbook is for clusters on a restricted network. Below are the links for offline installations of CPDBR, MTC and OADP.

CPDBR: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=utility-moving-images-backup-restore-utilities

MTC: https://access.redhat.com/documentation/en-us/openshift_container_platform/4.10/html/migration_toolkit_for_containers/installing-mtc-restricted

MTC rsync fix image: `quay.io/konveyor/mig-controller@sha256:68effcbdddd6a96c3185daea84f8741e6881fb117ca2e70d76957d53ae779c79` (Fixed on MTC verion >=1.7.10, not necessary if already on this version or newer)

OADP: https://docs.openshift.com/container-platform/4.10/operators/admin/olm-restricted-networks.html#olm-restricted-networks

Note: This playbook has been verified to work for Cloud Pak for Data 4.6.x. It will not work with Cloud Pak for Data 4.7.x, which will require different steps for it to work properly.

# Cloud pak for data operators backup and restore

## OADP installation and setup (source and target clusters)

Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=iobru-installing-cloud-pak-data-oadp-backup-restore-utility-components

**This operation needs to be done on both the SOURCE cluster and the TARGET cluster.**

>**IMPORTANT: If OADP operator is already installed in oadp-operator namespace, reinstall it to refresh the oadp crds, due to conflict between CPD and MTC versions of OADP that are used.**

Follow steps in the official documentation above which include:
- Create the oadp-operator project
- Annotate the oadp-operator project so that Restic pods can be scheduled on all nodes
- Install OADP operator
- Create Secret and environment variables for S3 backups
- Create OADP CR


**Example of CPD OADP operator installation (v1.1.4):**
![image](https://media.github.ibm.com/user/30787/files/d6f69943-8a78-4fe8-8733-1a0fb639dc5c)
```console
$ oc get csv -n oadp-operator
NAME                   DISPLAY         VERSION   REPLACES               PHASE
oadp-operator.v1.1.4   OADP Operator   1.1.4     oadp-operator.v1.1.3   Succeeded
```

**Example of creation of s3 object storage credentials secret:**

In this example, the s3 object storage credential secret is called `cloud-credentials`:
```console
$ cat credentials-velero-2
[default]
aws_access_key_id = 131f141055654845adf3a7918178xxxx
aws_secret_access_key = xxxx

$ oc create secret generic cloud-credentials \
--namespace oadp-operator \
--from-file cloud=./credentials-velero-2
secret/cloud-credentials created
$
```

**Example of creation of OADP CR. In this example it is called `dataprotectionapp-2.yaml`.**
```yaml
$ cat dataprotectionapp-2.yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: cpst-dpa
spec:
  configuration:
    velero:
      customPlugins:
      - image: icr.io/cpopen/cpd/cpdbr-velero-plugin:4.0.0-beta1-1-x86_64
        name: cpdbr-velero-plugin
      defaultPlugins:
      - aws
      - openshift
      - csi
      podConfig:
        resourceAllocations:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 256Mi
    restic:
      enable: true
      timeout: 12h
      podConfig:
        resourceAllocations:
          limits:
            cpu: "1"
            memory: 8Gi
          requests:
            cpu: 500m
            memory: 256Mi
          tolerations:
          - key: icp4data
            operator: Exists
            effect: NoSchedule
  backupImages: false
  backupLocations:
    - velero:
        provider: aws
        default: true
        objectStorage:
          bucket: mtc-todd
          prefix: cpst-9c9f-backup
        config:
          region: us-south
          s3ForcePathStyle: "true"
          s3Url: https://s3.us-south.cloud-object-storage.appdomain.cloud
        credential:
          name: cloud-credentials
          key: cloud
          
$ oc apply -f dataprotectionapp-2.yaml
dataprotectionapplication.oadp.openshift.io/cpst-dpa created
```

## Create cpd operators configmap on source cluster
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=utility-downloading-backup-restore-scripts

Download cpd-operators.sh script on a box with oc cli access to the source cluster:
```console
wget https://raw.githubusercontent.com/IBM/cpd-cli/master/cpdops/files/cpd-operators.sh
```

Note: foundation-namespace and operators namespace are the same for cpd express installations (ibm-common-services by default)

```console
cpd-operators.sh backup --foundation-namespace ibm-common-services --operators-namespace ibm-common-services
```
**Verify the configmap is present:**
```console
oc get configmap cpd-operators -n ibm-common-services
```

## Backup cpd operators on source cluster
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=obrou-scenario-creating-offline-backup-cloud-pak-data-instance-restoring-it-different-cluster

**Configure cpd-cli oadp to point to the project where the Velero instance is installed:**
```console
cpd-cli oadp client config set namespace=oadp-operator
```       

**Trigger a cpd oadp offline backup for cpd-operators namespace:**
```console
cpd-cli oadp backup create <cpd-ops-backup-name> --include-namespaces ibm-common-services --include-resources='namespaces,operatorgroups,configmaps,scheduling,crd' --skip-hooks --log-level=debug --verbose
```
**List backups:**
```console
cpd-cli oadp backup ls
```

**Verify the backup:**
```console
cpd-cli oadp backup status <cpd-ops-backup-name> --details
```

## Restore cpd operators on target cluster

**Check catalog sources on the target cluster and clean up any CPD related catalog sources**

Before running the CPD operator subscriptions restore, make sure the catalog sources are cleaned of CPD related ones on the target cluster.

Example output, showing no CPD related catalog sources:
```console
$ oc get catsrc -n openshift-marketplace
NAME                   DISPLAY                TYPE   PUBLISHER   AGE
certified-operators    Certified Operators    grpc   Red Hat     21d
community-operators    Community Operators    grpc   Red Hat     21d
ibm-operator-catalog   IBM Operator Catalog   grpc   IBM         21d
redhat-marketplace     Red Hat Marketplace    grpc   Red Hat     21d
redhat-operators       Red Hat Operators      grpc   Red Hat     21d
$
```
Verify ibm-common-services namespace does not exist in target cluster:
```console
oc get ns |grep common
```

**Configure cpd-cli oadp to point to the project where the Velero instance is installed:**
```console
cpd-cli oadp client config set namespace=oadp-operator
```      

**Restore cpd operators crds:**
```console
cpd-cli oadp restore create <cpd-operators-restore-crds> \
--from-backup=<cpd-ops-backup-name> \
--include-resources='crd' \
--include-cluster-resources=true \
--skip-hooks \
--log-level=debug \
--verbose
```

**Verify crd restore operation:**
```console
cpd-cli oadp restore status <cpd-operators-restore-crds> --details
```

**Restore additional resources, such as projects and operator groups, etc:**

Restore command:
```console
cpd-cli oadp restore create <cpd-operators-restore-remaining> \
--from-backup=<cpd-ops-backup-name>  \
--include-resources='namespaces,operatorgroups,scheduling,crd' \
--include-cluster-resources=true \
--skip-hooks \
--log-level=debug \
--verbose
```

**Verify ibm-common-services namespace and resources were created:**
```console
$ oc get ns |grep ibm-common-services
ibm-common-services                                Active   67s

$ oc get operatorgroups -n ibm-common-services
NAME            AGE
operatorgroup   83s
```

**Verify restore operation for remaining resources:**
```console
cpd-cli oadp restore status <cpd-operators-restore-remaining> --details
```

**Restore cpd operators configmaps:**

Configmaps before restore:
```console
$ oc get cm -n ibm-common-services
NAME                       DATA   AGE
kube-root-ca.crt           1      2m11s
openshift-service-ca.crt   1      2m11s
```

Restore command:
```console
cpd-cli oadp restore create <cpd-operators-restore-cm> \
 --from-backup=<cpd-ops-backup-name> \
 --include-resources='configmaps' \
 --selector 'app=cpd-operators-backup' \
 --skip-hooks \
 --log-level=debug \
 --verbose
```

Configmaps after restore:
```console
$ oc get cm -n ibm-common-services
NAME                       DATA   AGE
cpd-operators              18     24s
kube-root-ca.crt           1      3m30s
openshift-service-ca.crt   1      3m30s
```


**Verify restore operation for configmaps:**
```console
cpd-cli oadp restore status <cpd-operators-restore-cm>
```

**Restore operator subscriptions**
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=utility-downloading-backup-restore-scripts

**Download cpd-operators.sh script on target cluster:**
```console
wget https://raw.githubusercontent.com/IBM/cpd-cli/master/cpdops/files/cpd-operators.sh
```
**Run command:**
```console
cpd-operators.sh restore --foundation-namespace ibm-common-services --operators-namespace ibm-common-services

```

**Verify restore operator subscriptions and CSVs:**
Subscriptions
```console
$ oc get sub -n ibm-common-services
NAME                                                                          PACKAGE                              SOURCE                                       CHANNEL
cpd-operator                                                                  cpd-platform-operator                cpd-platform                                 v3.8
fdb-kubernetes-operator-v2.4-ibm-fdb-operator-catalog-openshift-marketplace   fdb-kubernetes-operator              ibm-fdb-operator-catalog                     v2.4
ibm-cert-manager-operator                                                     ibm-cert-manager-operator            opencloud-operators                          v3.23
ibm-common-service-operator                                                   ibm-common-service-operator          opencloud-operators                          v3.23
ibm-cpd-ae-operator                                                           analyticsengine-operator             ibm-cpd-ae-operator-catalog                  v3.5
ibm-cpd-ccs-operator                                                          ibm-cpd-ccs                          ibm-cpd-ccs-operator-catalog                 v6.5
ibm-cpd-datarefinery-operator                                                 ibm-cpd-datarefinery                 ibm-cpd-datarefinery-operator-catalog        v6.5
ibm-cpd-datastage-operator                                                    ibm-cpd-datastage-operator           ibm-cpd-datastage-operator-catalog           v3.5
ibm-cpd-iis-operator                                                          ibm-cpd-iis                          ibm-cpd-iis-operator-catalog                 v3.5
ibm-cpd-wkc-operator-catalog-subscription                                     ibm-cpd-wkc                          ibm-cpd-wkc-operator-catalog                 v3.5
ibm-cpd-wml-operator                                                          ibm-cpd-wml-operator                 ibm-cpd-wml-operator-catalog                 v3.5
ibm-cpd-ws-operator                                                           ibm-cpd-wsl                          ibm-cpd-ws-operator-catalog                  v6.5
ibm-cpd-ws-runtimes-operator                                                  ibm-cpd-ws-runtimes                  ibm-cpd-ws-runtimes-operator-catalog         v6.5
ibm-db2aaservice-cp4d-operator                                                ibm-db2aaservice-cp4d-operator       ibm-db2aaservice-cp4d-operator-catalog       v3.2
ibm-db2oltp-cp4d-operator-catalog-subscription                                ibm-db2oltp-cp4d-operator            ibm-db2oltp-cp4d-operator-catalog            v3.2
ibm-db2u-operator                                                             db2u-operator                        ibm-db2uoperator-catalog                     v3.2
ibm-db2wh-cp4d-operator-catalog-subscription                                  ibm-db2wh-cp4d-operator              ibm-db2wh-cp4d-operator-catalog              v3.2
ibm-dmc-operator-subscription                                                 ibm-dmc-operator                     ibm-dmc-operator-catalog                     v2.2
ibm-fdb-operator                                                              ibm-opencontent-foundationdb         ibm-fdb-operator-catalog                     v2.4
ibm-namespace-scope-operator                                                  ibm-namespace-scope-operator         opencloud-operators                          v3.23
ibm-watson-openscale-operator-subscription                                    ibm-cpd-wos                          ibm-openscale-operator-catalog               v3.5
ibm-zen-operator                                                              ibm-zen-operator                     opencloud-operators                          v3.23
manta-adl-operator                                                            manta-adl-operator                   manta-adl-operator-catalog                   v1.10
operand-deployment-lifecycle-manager-app                                      ibm-odlm                             opencloud-operators                          v3.23
redis-operator                                                                ibm-cloud-databases-redis-operator   ibm-cloud-databases-redis-operator-catalog   v1.6
```
CSVs
```console
$ oc get csv -n ibm-common-services
NAME                                           DISPLAY                                                              VERSION   REPLACES                           PHASE
cpd-platform-operator.v3.8.0                   Cloud Pak for Data Platform Operator                                 3.8.0     cpd-platform-operator.v3.7.0       Succeeded
db2u-operator.v3.2.0                           IBM Db2                                                              3.2.0                                        Succeeded
fdb-kubernetes-operator.v2.4.6                 FoundationDB Kubernetes                                              2.4.6                                        Succeeded
ibm-cert-manager-operator.v3.25.2              IBM Cert Manager                                                     3.25.2                                       Succeeded
ibm-cloud-databases-redis.v1.6.6               IBM Operator for Redis                                               1.6.6     ibm-cloud-databases-redis.v1.6.5   Succeeded
ibm-common-service-operator.v3.23.2            IBM Cloud Pak foundational services                                  3.23.2                                       Succeeded
ibm-cpd-ae.v3.5.0                              IBM Analytics Engine Powered by Apache Spark Service                 3.5.0                                        Succeeded
ibm-cpd-ccs.v6.5.0                             Common Core Services                                                 6.5.0                                        Succeeded
ibm-cpd-datarefinery.v6.5.0                    IBM Data Refinery                                                    6.5.0                                        Succeeded
ibm-cpd-datastage-operator.v3.5.0              IBM DataStage                                                        3.5.0                                        Succeeded
ibm-cpd-iis.v1.6.5                             IIS Services                                                         1.6.5                                        Succeeded
ibm-cpd-wkc.v1.6.5                             WKC Services                                                         1.6.5                                        Succeeded
ibm-cpd-wml-operator.v3.5.0                    IBM WML Services                                                     3.5.0                                        Succeeded
ibm-cpd-wos.v3.5.0                             IBM Watson OpenScale                                                 3.5.0                                        Succeeded
ibm-cpd-ws-runtimes.v6.5.0                     Watson Studio Notebook Runtimes                                      6.5.0                                        Succeeded
ibm-cpd-wsl.v6.5.0                             Watson Studio                                                        6.5.0                                        Succeeded
ibm-databases-dmc.v2.2.0                       IBM Db2 Data Management Console                                      2.2.0                                        Succeeded
ibm-db2aaservice-cp4d-operator.v3.2.0          IBM Db2 as a Service Operator Extension for IBM Cloud Pak for Data   3.2.0                                        Succeeded
ibm-db2oltp-cp4d-operator.v3.2.0               IBM® Db2 Operator Extension for IBM Cloud Pak for Data               3.2.0                                        Succeeded
ibm-db2wh-cp4d-operator.v3.2.0                 IBM® Db2 Warehouse Operator Extension for IBM® Cloud Pak for Data    3.2.0                                        Succeeded
ibm-namespace-scope-operator.v1.17.2           IBM NamespaceScope Operator                                          1.17.2                                       Succeeded
ibm-opencontent-foundationdb.v2.4.6            IBM Opencontent FoundationDB                                         2.4.6                                        Succeeded
ibm-zen-operator.v1.8.3                        IBM Zen Service                                                      1.8.3                                        Succeeded
manta-adl-operator.v1.10.0                     MANTA Automated Data Lineage                                         1.10.0                                       Succeeded
operand-deployment-lifecycle-manager.v1.21.2   Operand Deployment Lifecycle Manager                                 1.21.2                                       Succeeded
```

Note: The exact subscription and csv installed can vary based on the services installed on source cluster. 


## Collect an oadp resources dump of the cpd namespace on source cluster

To collect CPD resources from source cluster, perform the following:
```console
cpd-cli oadp backup create <src-cpd-instance-resources> \
--include-namespaces <cpd-namespace> \
--exclude-resources='event,event.events.k8s.io,imagetags.openshift.io' \
--include-cluster-resources=true \
--snapshot-volumes=false \
--skip-hooks=true \
--log-level=debug \
--verbose
```

Download the resource list:
```
cpd-cli oadp backup download <src-cpd-instance-resources> -o <src-cpd-instance-resources>-data.tar.gz
```

Provide the data.tar.gz tarball to IBM. This will be used to help identify all of the resources with storage class references and will be used to build the `input.json` file, which will be used in a later step to run the `cpd-migrate.sh` tool.

**Example for how to use the cpd resource tarball to identify resources with storage class references:**

Extract tarball
```console
$ tar -xzf src-cpd-instance-resources-data.tar.gz

$ pwd
/home/admin/cp4d/cpd-cli-linux-EE-12.0.5-61/src-cpd-instance-resources-data

$ ls -l
total 12
drwxrwxr-x.   2 admin admin   21 May 15 19:44 metadata
drwxrwxr-x. 129 admin admin 8192 May 15 19:44 resources
```

From the extracted oadp tar bundle, we can perform a series of grep commands to narrow in on the subset of k8s resources that are affected with storage class references.

Use helper commands to find potential resources to change.
  - If looking at the cpd-cli oadp backup tar bundle, example grep commands to help find potential resources for patching.
 
  - The best grep command will be to grep on the actual storageclass name(s). This will directly find where the storageclass is referenced. 
```console
grep -Rl -E "<cpd-storage-class>" ./* | \
grep -vE "persistentvolumes|persistentvolumeclaims|preferredversion|storageclasses.storage.k8s.io"
```
  - The next grep command is to identify which of those json files from the previous grep might have both RWO and RWX references in them. 
```console
grep -Rl -E "<cpd-storage-class>" ./* | \
grep -vE "persistentvolumes|persistentvolumeclaims|preferredversion|storageclasses.storage.k8s.io" | \
xargs -I{} sh -c "echo ------ {} -------;cat {} | \
jq . | grep -E 'ReadWriteOnce|ReadWriteMany|<cpd-storage-class>'"
```


# MTC installation and setup on source and target clusters

> **IMPORTANT: If MTC is already installed in openshift-migration namespace, reinstall just the OADP operator (version v1.0.x) in the openshift-migration namespace to refresh the oadp crds, due to conflict between CPD and MTC versions of OADP that are used.**

**Summary of steps in this section:**
- Install Migration Toolkit for Containers Operator (MTC) (currently v1.7.9) on both source and target clusters
- From source cluster MTC, add target cluster "Add cluster"
- Add replication repository (s3 object store, accessible from both source and target clusters)

**Install MTC and create MigrationController on source and target clusters**

MTC and MigrationControllor need to be installed and created on both the source cluster and the target cluster.

Install MTC operator from Operator hub: 
<img width="1304" alt="image" src="https://media.github.ibm.com/user/216626/files/7b36477f-8e70-4197-a793-9fba0477bf92">

![image](https://media.github.ibm.com/user/30787/files/fd801f30-39ed-40a7-9fec-a5a7f995d461)

After operator is installed, select to create a MigrationController instance.
 - Change the view from "Form view" to "YAML view"
 - Add the following properties to YAML for known rsync issue under the `spec` section:

**Note: If you have installed MTC v1.7.10 or newer you do NOT have to setup the `mig_controller_image_fqin` because the fix is already integrated**
```yaml
  mig_controller_image_fqin: quay.io/konveyor/mig-controller@sha256:68effcbdddd6a96c3185daea84f8741e6881fb117ca2e70d76957d53ae779c79
  migration_rsync_privileged: true
```

Example YAML with updated `spec` section: 

```yaml
apiVersion: migration.openshift.io/v1alpha1
kind: MigrationController
metadata:
  creationTimestamp: '2023-05-25T07:55:19Z'
  generation: 2
  managedFields:
    - apiVersion: migration.openshift.io/v1alpha1
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          'f:migration_controller': {}
          'f:migration_log_reader': {}
          'f:cluster_name': {}
          'f:restic_timeout': {}
          'f:migration_rsync_privileged': {}
          'f:mig_pv_limit': {}
          'f:migration_velero': {}
          .: {}
          'f:mig_namespace_limit': {}
          'f:mig_controller_image_fqin': {}
          'f:azure_resource_group': {}
          'f:mig_pod_limit': {}
          'f:migration_ui': {}
          'f:olm_managed': {}
      manager: Mozilla
      operation: Update
      time: '2023-05-25T07:55:19Z'
    - apiVersion: migration.openshift.io/v1alpha1
      fieldsType: FieldsV1
      fieldsV1:
        'f:status':
          .: {}
          'f:conditions': {}
      manager: ansible-operator
      operation: Update
      subresource: status
      time: '2023-05-25T07:55:19Z'
    - apiVersion: migration.openshift.io/v1alpha1
      fieldsType: FieldsV1
      fieldsV1:
        'f:spec':
          'f:version': {}
      manager: OpenAPI-Generator
      operation: Update
      time: '2023-05-25T07:55:30Z'
  name: migration-controller
  namespace: openshift-migration
  resourceVersion: '44253841'
  uid: aa036929-1a16-4e7f-b967-5f0c3ad3bfb6
spec:
  mig_controller_image_fqin: >-
    quay.io/konveyor/mig-controller@sha256:68effcbdddd6a96c3185daea84f8741e6881fb117ca2e70d76957d53ae779c79
  mig_namespace_limit: '10'
  migration_ui: true
  mig_pod_limit: '100'
  migration_controller: true
  migration_log_reader: true
  olm_managed: true
  cluster_name: host
  restic_timeout: 1h
  migration_rsync_privileged: true
  migration_velero: true
  mig_pv_limit: '100'
  version: 1.7.9
  azure_resource_group: ''
status:
  conditions:
    - lastTransitionTime: '2023-05-25T07:56:16Z'
      message: ''
      reason: ''
      status: 'False'
      type: Failure
    - ansibleResult:
        changed: 0
        completion: '2023-05-31T21:17:47.506537'
        failures: 0
        ok: 51
        skipped: 22
      lastTransitionTime: '2023-05-25T07:55:19Z'
      message: Awaiting next reconciliation
      reason: Successful
      status: 'True'
      type: Running
    - lastTransitionTime: '2023-05-31T21:17:47Z'
      message: Last reconciliation succeeded
      reason: Successful
      status: 'True'
      type: Successful
```


Click "Create" and ensure MTC status is `Running`.
![image](https://media.github.ibm.com/user/30787/files/794d2537-2e94-448d-a71f-ad368e38ec4c)

**Add target cluster to source cluster MTC:**


Launch the migration controller console by clicking on the route created in "openshift-migration" namespace.

From source cluster MTC controller, go to "Clusters" > "Add cluster".

> IMPORTANT: MTC will also try to migrate application's images. The user can avoid the image migration if for example the target repository/artifactory already has the images needed.  If the user does require image migration it is highly recommended to enable direct image migration by putting the exposed registry hostname while adding the cluster. Please see the last section of this document for more information

Add target cluster access details. For Service account token, you can execute the following commands in the target cluster.
Please find the correct secret in your environment by executing `oc -n openshift-migration get secret|grep velero-token` and substute it in the command below:
```console
oc -n openshift-migration get secret velero-token-XXXXX -ojsonpath='{.data.token}'|base64 -d
```
![image](https://media.github.ibm.com/user/30787/files/129de39d-7e1f-41e3-9625-86de20ecc370)
    
Next, verify source and target clusters are available and connected.
![image](https://media.github.ibm.com/user/30787/files/dc827845-f119-496c-ab1a-8110e501a455)


**Add replication repository (access to s3 object store):**

From `source cluster` MTC controller. Go to "Replication repositories" > "Add replication repository"
Add the s3 object store access details and click "Add repository" and verify it is successfully connected.
![image](https://media.github.ibm.com/user/30787/files/73f10c4c-ee2c-4d92-a669-b47e7070b5bc)

Repo should now be accessible:
![image](https://media.github.ibm.com/user/30787/files/d70ab5a8-fe66-4128-b841-7b665cfc73dc)

# MTC Migration plan creation

From source MTC migration console, select "Migration Plans" > "Add migration plan"
Select "Full migration" migration type. Complete the remainder of the "General" section of the migration plan.
![image](https://media.github.ibm.com/user/30787/files/b6bed41d-f3f2-4851-bf71-390c53455b8f)

**Namespaces:** Select your Cloud Pak for Data namespace. 
![image](https://media.github.ibm.com/user/30787/files/db9f984d-d7b9-453b-b76b-b471adb881a0)

**Persistent volumes:** For PVs, after persistent volumes are discovered, the MTC UI will have all PVs selected, with PV Migration Type as "Filesystem copy" by default. Leave the defaults and click "Next".
![image](https://media.github.ibm.com/user/30787/files/8fd1d179-bb9d-4ec6-bccd-0b46e8c58726)

**Copy options:** Leave all of the default values and click "Next" without making any changes.
In general, you would want to select the "file" storage class for RWX volumes and the "block" storage class for RWO volumes.

    For ODF, it will be:
        RWX - file sc: ocs-storagecluster-cephfs
        RWO - block sc: ocs-storagecluster-ceph-rbd

However, MTC detects if the PVC is RWX vs RWO and attempts to select the correct storage class on the target cluster.

**IMPORTANT NOTE: Even though MTC will detect most of the PVCs correct access mode, Please review this section thoroughly, there are some cases where some pvcs are populated with a default storage class instead of the correct ones (ODF).**

Example screenshot:
![image](https://media.github.ibm.com/user/30787/files/754c882c-88a7-482e-9eb4-d95cfd2c7f69)

**Migration options:**

For Persistent volumes, the option "Use direct PV migration for filesystem copies" option has the following effect:
If selected, then it will create an rsync server on the target cluster and perform rsync copies of the PV data from source to target.
If not selected, then it will perform an indirect copy, where the data is pushed to s3 object storage location via velero/restic. Then pulled from the s3 object store to the PV.
The direct method is faster, since it performs a direct copy, as long as the two clusters have network connectivity to one another.
![image](https://media.github.ibm.com/user/30787/files/5d251490-00f3-4b74-a0c0-6a8c3554fec5)
 
**Hooks:** For CPD migration, no hooks are used. Click "Next".
![image](https://media.github.ibm.com/user/30787/files/499bc0f4-4bfa-4932-83af-d474c01638eb)

**Migration plan validation:**



The migration plan validation might display the following known warning, which indicates that it cannot calculate how much capacity for each PVC has been used in the current state. So just make sure that you have not exceeded the capacity of any of the PVCs, otherwise the data transfer will fail. But as long as the applications honor the PVC limit size, this will not be a problem.
> Failed to compute PV resizing data for the following volumes. PV resizing will be disabled for these volumes and the migration may fail if the volumes are full or their requested and actual capacities differ in the source cluster.


### Migration Plan PVC storage class validation

After the migration plan is created, you can see the migplan CR:

Example command:
```console
$ oc get migplan -n openshift-migration
NAME                    READY   SOURCE   TARGET        STORAGE                AGE
cp4d-465-wvyb-to-9c9f   True    host     target-wvyb   ibmcos-mtc-todd-repo   34m
$
```

Run the following command against the migration plan to see if the PVCs have the right matching of storage class to access mode:
Command:
```console
oc get migplan -n openshift-migration <migplan-name> -o json | jq -r '.spec.persistentVolumes[] | "[PVC:" + .pvc.name + "][Access mode:" + .pvc.accessModes[0] + "][New storage class:" + .selection.storageClass + "]"'
```

Compare the output one by one to validate it is as expected.

Below are grep commands to help validate quickly:
- Validate that all RWX have cephfs storage class (you want no grep matches):
```console
oc get migplan -n openshift-migration <migplan-name> -o json | jq -r '.spec.persistentVolumes[] | "[PVC:" + .pvc.name + "][Access mode:" + .pvc.accessModes[0] + "][New storage class:" + .selection.storageClass + "]"' |grep ReadWriteMany |grep -v cephfs
```
- Validate that all RWO have ceph-rbd storage class (you want no grep matches):
```console
oc get migplan -n openshift-migration <migplan-name> -o json | jq -r '.spec.persistentVolumes[] | "[PVC:" + .pvc.name + "][Access mode:" + .pvc.accessModes[0] + "][New storage class:" + .selection.storageClass + "]"' |grep ReadWriteOnce |grep -v ceph-rbd
```

- Example output:
```console
$ oc get migplan -n openshift-migration <migplan-name> -o json | jq -r '.spec.persistentVolumes[] | "[PVC:" + .pvc.name + "][Access mode:" + .pvc.accessModes[0] + "][New storage class:" + .selection.storageClass + "]"' |grep ReadWriteMany |grep -v cephfs
[PVC:cc-home-pvc][Access mode:ReadWriteMany][New storage class:ocs-storagecluster-cephfs]

$ oc get migplan -n openshift-migration <migplan-name> -o json | jq -r '.spec.persistentVolumes[] | "[PVC:" + .pvc.name + "][Access mode:" + .pvc.accessModes[0] + "][New storage class:" + .selection.storageClass + "]"' |grep ReadWriteOnce |grep -v ceph-rbd
[PVC:zen-metastore-edb-1][Access mode:ReadWriteOnce][New storage class:ocs-storagecluster-ceph-rbd]
[PVC:export-zen-minio-0][Access mode:ReadWriteOnce][New storage class:ocs-storagecluster-ceph-rbd]
[PVC:export-zen-minio-2][Access mode:ReadWriteOnce][New storage class:ocs-storagecluster-ceph-rbd]
[PVC:zen-metastore-edb-2][Access mode:ReadWriteOnce][New storage class:ocs-storagecluster-ceph-rbd]
[PVC:ibm-zen-objectstore-backup-pvc][Access mode:ReadWriteOnce][New storage class:ocs-storagecluster-ceph-rbd]
[PVC:export-zen-minio-1][Access mode:ReadWriteOnce][New storage class:ocs-storagecluster-ceph-rbd]
```

If the target storage classes for any of the PVCs do not match what you intended, then you will need to edit the Migration Plan and adjust the target storage classes. 
This can be accomplished by one of these methods:
- Go to the MTC console, Migration plans. Select the Migration Plan and edit.
- Edit the Migration Plan from cli: 

```console
oc edit migplan -n openshift-migration <migplan-name>
```

# Shutdown Cloud Pak for Data instance on source cluster

The goal of this step is to have ALL the CPD resources either shutdown or in maintenance mode. 

## Set oadp prehooks pre-reqs
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=backup-prerequisite-tasks

DB2 instances:

Obtain all the db2uclusters in the namespace

```console
oc -n <cpd-namespace> get db2ucluster 
```

For each of the db2ucluster displayed in the command above execute the following command:

```console
oc -n <cpd-instance> label db2ucluster <DB2UCLUSTER> db2u/cpdbr=db2u --overwrite
```

Verify each db2ucluster has the label `db2u/cpdbr=db2u`:


```console
oc -n <cpd-instance> get db2ucluster --show-labels
```

## Scale down CPD services by running backup prehooks

Example command:
```console
cpd-cli oadp backup prehooks --include-namespaces <cpd-namespace> --log-level=debug --verbose
```

Check the state of the deployments and statefulsets in the CPD instance namespace by running the following commands, and save this output for future comparison:
```console
oc get deploy -n <cpd-namespace>
oc get sts -n <cpd-namespace>
oc get pod -n <cpd-namespace> --no-headers |grep -v Completed |wc -l
```

Run backup prehooks:
```
cpd-cli oadp backup prehooks --include-namespaces <cpd-namespace> --log-level=debug --verbose
```

Verify successful completion of backup prehooks by ensuring output for command above includes these messages: 
```
configmap pre-backup hooks completed
backup prehooks command completed
```

This command will change the status of the different resources in CPD to maintenance mode, Some of them might require manual intervention, the test team has identify some of the services that require manual assistance, but please consider that if other services besides the one posted in this playbook are installed in the cluster and they are not properly handled by the backup prehooks command they might need extra procedures as well. 

Check for all the resources 

After running the backup prehooks, check for the remaining deployments/statefulsets/pods running:

Example: 
```console
$ oc get deploy -n <cpd-namespace> |grep -v "0/0"
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
ibm-nginx                                    2/2     2            2           16h
manta-admin-gui                              1/1     1            1           13h
manta-artemis                                1/1     1            1           13h
manta-configuration-service                  1/1     1            1           13h
manta-dataflow                               1/1     1            1           13h
manta-flow-agent                             1/1     1            1           13h


$ oc get sts -n <cpd-namespace> |grep -v "0/0"
NAME                             READY   AGE
c-db2oltp-iis-db2u               1/1     13h
c-db2oltp-wkc-db2u               1/1     13h


$ oc get pod -n <cpd-namespace> |grep -v Completed
NAME                                                      READY   STATUS      RESTARTS      AGE
c-db2oltp-iis-db2u-0                                      1/1     Running     0             13h
c-db2oltp-wkc-db2u-0                                      1/1     Running     0             13h
ibm-nginx-69dd4dc867-9vrdz                                1/1     Running     0             16h
ibm-nginx-69dd4dc867-lc7pf                                1/1     Running     0             16h
manta-admin-gui-5bcd558778-bq9mz                          1/1     Running     0             13h
manta-artemis-8f4b5d55f-xmjfk                             1/1     Running     0             13h
manta-configuration-service-5d87768b87-49bsw              1/1     Running     0             13h
manta-dataflow-56577f6f75-kxtss                           1/1     Running     0             13h
manta-flow-agent-5dfcb4864b-rfd2n                         1/1     Running     0             13h
wkc-foundationdb-cluster-cluster-controller-1             2/2     Running     0             14h
wkc-foundationdb-cluster-log-1                            2/2     Running     0             14h
wkc-foundationdb-cluster-log-2                            2/2     Running     0             14h
wkc-foundationdb-cluster-storage-1                        2/2     Running     0             14h
wkc-foundationdb-cluster-storage-2                        2/2     Running     1 (13h ago)   14h
wkc-foundationdb-cluster-storage-3                        2/2     Running     0             14h

```


### Scale down remaining services and resources running in CPD namespace.

For the remaining pods running in the CPD instance namespace, they will have to be shutdown manually. The following is a list of known services/deployments/statefulsets that have to be shutdown manually, and the process to do so.


**nginx:**
```console
oc get deploy/ibm-nginx -n <cpd-namespace>  #retain this output for the resume
oc scale --replicas=0 deploy/ibm-nginx -n <cpd-namespace>
```

**db2uclusters statefulsets:**
```console
oc get sts -n <cpd-namespace> |grep -E "NAME|db2"  #retain this output for the resume
oc scale --replicas=0 sts/c-db2oltp-iis-db2u -n <cpd-namespace>
oc scale --replicas=0 sts/c-db2oltp-wkc-db2u -n <cpd-namespace>

## Get sts for your db2wh instance
oc get sts -licpdsupport/addOnId=db2wh -n <cpd-namespace> #retain this output for the resume, you will see two sts (one for db2u and one for etcd)
oc scale --replicas=0 sts/<db2wh-db2u-sts> -n <cpd-namespace>
oc scale --replicas=0 sts/<db2wh-etc-sts> -n <cpd-namespace>
```
Note: There may be additional db2u related statefulsets that need to be scaled down. Scale them down similar to the above commands.


**Foundationdb (FDB)** 
```console
oc patch -p '{"spec":{"shutdown":"true"}}' --type=merge fdbcluster wkc-foundationdb-cluster -n <cpd-namespace>
```


**MANTA**
(IBM internal reference: https://github.ibm.com/manta/manta-adl-operator#scaling)
```console
oc patch -p '{"spec":{"replicas":0}}' --type=merge mantaflow mantaflow-wkc -n <cpd-namespace>
```


**Openscale** 

Note: Even though the openscale deployments and statefulsets are shutdown during the backup prehooks, the service itself is not shutdown and not in maintenance mode. That means that it could be reconciled and started up again when not desired on the target cluster after the cpd operators are restored. To prevent this, we need to manually shut down the openscale service on the source cluster.
```console
oc patch -p '{"spec":{"shutdown":"true"}}' --type=merge woservice aiopenscale -n <cpd-namespace>
oc patch -p '{"spec":{"ignoreForMaintenance": true}}' --type=merge woservice aiopenscale -n <cpd-namespace>
```

**Datastage deploy and statefulset**
```console
## Record this output for the resume operations:
oc get deploy,sts -n <cpd-namespace>|grep ds-px-default 

oc scale --replicas=0 deploy/ds-px-default-ibm-datastage-px-runtime -n <cpd-namespace>
oc scale --replicas=0 sts/ds-px-default-ibm-datastage-px-compute -n <cpd-namespace>
```

**Example Output for scale down and shutdown commands above:**

**nginx**
```console
$ oc get deploy ibm-nginx -n cpd
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
ibm-nginx   2/2     2            2           16h
$ oc scale --replicas=0 deploy/ibm-nginx
deployment.apps/ibm-nginx scaled
$ oc get deploy ibm-nginx -n cpd
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
ibm-nginx   0/0     0            0           16h
$ oc get pod -n cpd |grep -v Completed |grep -E "NAME|nginx"
NAME                                                      READY   STATUS      RESTARTS      AGE
$
```

**db2ucluster statefulsets**
```console
$ oc get db2ucluster -n cpd
NAME                     STATE   MAINTENANCESTATE   AGE
db2oltp-iis              Ready   InMaintenance      2d17h
db2oltp-wkc              Ready   InMaintenance      8d
db2wh-1686277435040922   Ready   InMaintenance      13h

$ oc get sts -n cpd |grep -v "0/0"
NAME                                     READY   AGE
c-db2oltp-iis-db2u                       1/1     2d17h
c-db2oltp-wkc-db2u                       1/1     8d
c-db2wh-1686277435040922-db2u            1/1     13h
c-db2wh-1686277435040922-etcd            1/1     13h
ds-px-default-ibm-datastage-px-compute   2/2     8d

$ oc  scale --replicas=0 sts/c-db2oltp-iis-db2u -n cpd
statefulset.apps/c-db2oltp-iis-db2u scaled
$ oc scale --replicas=0 sts/c-db2oltp-wkc-db2u -n cpd
statefulset.apps/c-db2oltp-wkc-db2u scaled
$ oc scale --replicas=0 sts/c-db2wh-1686277435040922-db2u -n cpd
statefulset.apps/c-db2wh-1686277435040922-db2u scaled
$ oc scale --replicas=0 sts/c-db2wh-1686277435040922-etcd -n cpd
statefulset.apps/c-db2wh-1686277435040922-etcd scaled


$ oc get sts -n cpd |grep -E "NAME|db2u"
NAME                             READY   AGE
c-db2oltp-iis-db2u               0/0     14h
c-db2oltp-wkc-db2u               0/0     14h
c-db2wh-1686277435040922-db2u            0/0     13h
c-db2wh-1686277435040922-etcd            0/0     13h

$ oc get pod -n cpd |grep -v Completed |grep -E "NAME|db2"
NAME                                                      READY   STATUS      RESTARTS      AGE
$
```

**Foundationdb shutdown**
```console
$ oc describe fdbcluster wkc-foundationdb-cluster -n cpd |grep " foundationdb.opencontent.ibm.com/backup-trigger"
              foundationdb.opencontent.ibm.com/backup-trigger: pre-backup

$ oc patch -p '{"spec":{"shutdown":"true"}}' --type=merge fdbcluster wkc-foundationdb-cluster -n cpd
fdbcluster.foundationdb.opencontent.ibm.com/wkc-foundationdb-cluster patched

$ oc get fdbcluster wkc-foundationdb-cluster -n cpd -o yaml |grep shutdown
  shutdown: "true"
  shutdownStatus: ' '

$ oc get pod -n cpd |grep -E "NAME|fdb|found"
NAME                                                      READY   STATUS        RESTARTS      AGE
wkc-foundationdb-cluster-cluster-controller-1             2/2     Terminating   0             14h
wkc-foundationdb-cluster-log-1                            2/2     Terminating   0             14h
wkc-foundationdb-cluster-log-2                            2/2     Terminating   0             14h
wkc-foundationdb-cluster-storage-1                        0/2     Terminating   0             14h
wkc-foundationdb-cluster-storage-2                        2/2     Terminating   1 (14h ago)   14h
wkc-foundationdb-cluster-storage-3                        2/2     Terminating   0             14h

$ oc get fdbcluster wkc-foundationdb-cluster -n cpd -o yaml |grep shutdown
  shutdown: "true"
  shutdownStatus: shutdown

$ oc get pod -n cpd |grep -E "NAME|fdb|found"
NAME                                                      READY   STATUS      RESTARTS   AGE

```

**MANTA scaledown replica**
```console
$ oc get pod -n cpd |grep -v Completed
NAME                                                      READY   STATUS      RESTARTS   AGE
manta-admin-gui-5bcd558778-bq9mz                          1/1     Running     0          13h
manta-artemis-8f4b5d55f-xmjfk                             1/1     Running     0          13h
manta-configuration-service-5d87768b87-49bsw              1/1     Running     0          13h
manta-dataflow-56577f6f75-kxtss                           1/1     Running     0          13h
manta-flow-agent-5dfcb4864b-rfd2n                         1/1     Running     0          13h

$ oc get mantaflow -n cpd
NAME            AGE
mantaflow-wkc   13h

$ oc get mantaflow -n cpd mantaflow-wkc -o yaml |grep replicas
  replicas: 1

$ oc patch -p '{"spec":{"replicas":0}}' --type=merge mantaflow mantaflow-wkc -n cpd
mantaflow.adl.getmanta.com/mantaflow-wkc patched

$ oc get mantaflow mantaflow-wkc -o yaml |grep replicas
  replicas: 0

$ oc get pod -n cpd |grep -E "NAME|manta"
NAME                                                      READY   STATUS      RESTARTS   AGE

```

**Openscale shutdown:**

Shutdown openscale:
```console
$ oc -n cpd get woservice
NAME                        TYPE              STORAGE   SCALECONFIG   PHASE   RECONCILED   STATUS
aiopenscale                 service                     small         Ready   4.6.5        Completed
openscale-defaultinstance   serviceInstance             small         Ready   4.6.5        Completed

$ oc get woservice aiopenscale -o json |jq .spec.shutdown
"false"

$ oc patch -p '{"spec":{"shutdown":"true"}}' --type=merge woservice aiopenscale -n cpd
woservice.wos.cpd.ibm.com/aiopenscale patched

$ oc -n cpd get woservice aiopenscale -o json |jq -y .spec.shutdown
'true'

```

Place it in maintenance mode:
```console
$ oc patch -p '{"spec":{"ignoreForMaintenance": true}}' --type=merge woservice aiopenscale -n cpd

$ oc get woservice
NAME                        TYPE              STORAGE   SCALECONFIG   PHASE           RECONCILED   STATUS
aiopenscale                 service                     small         InMaintenance   4.6.5        InMaintenance
openscale-defaultinstance   serviceInstance             small         shutdown        4.6.5        shutdown
```
```yaml
$ oc get woservice aiopenscale -o yaml
apiVersion: wos.cpd.ibm.com/v1
kind: WOService
metadata:
  creationTimestamp: "2023-05-19T05:34:44Z"
  finalizers:
  - wos.cpd.ibm.com/finalizer
  generation: 5
  name: aiopenscale
  namespace: cpdinstance
  resourceVersion: "13630612"
  uid: 18596e99-8ffc-47f4-9068-a0d2d3fd8542
spec:
  acceptRollback: false
  blockStorageClass: cpd-storage
  fileStorageClass: cpd-storage
  ignoreForMaintenance: true
  license:
    accept: true
    license: Enterprise
  scaleConfig: small
  shutdown: "true"
  type: service
  version: 4.6.5
status:
  conditions:
  - ansibleResult:
      changed: 1
      completion: 2023-05-19T18:49:06.575661
      failures: 0
      ok: 2
      skipped: 0
    lastTransitionTime: "2023-05-19T18:49:01Z"
    message: Awaiting next reconciliation
    reason: Successful
    status: "True"
    type: Running
  - lastTransitionTime: "2023-05-19T18:49:06Z"
    message: Last reconciliation succeeded
    reason: Successful
    status: "True"
    type: Successful
  - lastTransitionTime: "2023-05-19T18:49:01Z"
    message: ""
    reason: ""
    status: "False"
    type: Failure
  phase: InMaintenance
  versions:
    reconciled: 4.6.5
  wosBuildNumber: "105"
  wosStatus: InMaintenance
$
```
**Datastage**
```console
$ oc scale --replicas=0 deploy/ds-px-default-ibm-datastage-px-runtime -n cpd
deploy.apps/ds-px-default-ibm-datastage-px-runtime scaled

$ oc scale --replicas=0 sts/ds-px-default-ibm-datastage-px-compute -n cpd
statefulset.apps/ds-px-default-ibm-datastage-px-compute scaled

```
**Validate all CPD services and pods are now shutdown:**

Ensure all pods in the CPD instance namespace are now shutdown:
```console
$ oc get pod -n <cpd-namespace> |grep -v Completed
NAME                                                      READY   STATUS      RESTARTS   AGE
$
```

Ensure all CPD services are in maintenance mode:
```console
for cpdcrd in $(oc get crd |grep cpd.ibm.com |awk '{print $1}');do echo --- $cpdcrd ---;oc get $cpdcrd -o name |xargs -I{} sh -c "oc get {} -o yaml |grep -i maintenance";done
```
Example:
```console
$ for cpdcrd in $(oc get crd |grep cpd.ibm.com |awk '{print $1}');do echo --- $cpdcrd ---;oc get $cpdcrd -o name |xargs -I{} sh -c "oc get {} -o yaml |grep -i maintenance";done
--- analyticsengines.ae.cpd.ibm.com ---
  ignoreForMaintenance: true
  analyticsengineStatus: InMaintenance
--- ccs.ccs.cpd.ibm.com ---
  ignoreForMaintenance: true
  ccsStatus: InMaintenance
--- datarefinery.datarefinery.cpd.ibm.com ---
  ignoreForMaintenance: true
  datarefineryStatus: InMaintenance
--- datastages.ds.cpd.ibm.com ---
--- db2aaserviceservices.databases.cpd.ibm.com ---
  ignoreForMaintenance: true
  db2aaserviceStatus: InMaintenance
--- db2oltpservices.databases.cpd.ibm.com ---
  ignoreForMaintenance: true
  db2oltpStatus: InMaintenance
--- db2whservices.databases.cpd.ibm.com ---
  ignoreForMaintenance: true
  db2whStatus: InMaintenance
--- ibmcpds.cpd.ibm.com ---
--- iis.iis.cpd.ibm.com ---
  ignoreForMaintenance: true
  iisStatus: InMaintenance
--- notebookruntimes.ws.cpd.ibm.com ---
  ignoreForMaintenance: true
  runtimeStatus: InMaintenance
--- pxruntimes.ds.cpd.ibm.com ---
--- ug.wkc.cpd.ibm.com ---
  ignoreForMaintenance: true
  ugStatus: InMaintenance
--- wkc.wkc.cpd.ibm.com ---
  ignoreForMaintenance: true
  wkcStatus: InMaintenance
--- wmlbases.wml.cpd.ibm.com ---
  ignoreForMaintenance: true
  wmlStatus: InMaintenance
--- woservices.wos.cpd.ibm.com ---
  ignoreForMaintenance: true
  phase: InMaintenance
  wosStatus: InMaintenance
  ignoreForMaintenance: false
--- ws.ws.cpd.ibm.com ---
  ignoreForMaintenance: true
  wsStatus: InMaintenance
--- zenextensions.zen.cpd.ibm.com ---
--- zenservices.zen.cpd.ibm.com ---
  ignoreForMaintenance: true
  zenStatus: InMaintenance
```

# PVC resizing script FIX (Target Cluster)

A problem was found during the execution of the current playbook, some of the PVCs got filled during the migration because the big amount of data present environment. Because of this the migration phase failed after a lot of time. In order to address this problem Mr. @JC Lopez created a script to dynamically catch the `near_full_alert` and `critical_full_alert` and resize the PVC accordingly.

Check that the target storage classes has the volume expansion set
```console
oc get sc|grep -E "NAME|ocs"                                                                                                                                                                   IAM#israel.a.vizcarra@ibm.com@:openshift-migration
NAME                                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ocs-storagecluster-ceph-rbd                   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   26d
ocs-storagecluster-cephfs                     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   26d
```

If the storage class does not have the allow volume expansion setting as true use the following command to patch it
```console
oc patch sc <sc> -p '{"allowVolumeExpansion": true}'
```

In a bastion system with access to the target cluster create the following bash script using `vi grow_pvc.sh`
```console
#!/bin/bash
grow_pvc_on_nearfull=${1:-"No"} 
grow_pvc_on_full=${2:-"Yes"} 
grow_rate=${3:-1.25}
grow_debug={$4:-""}

echo "Resizing on nearfull alert=${grow_pvc_on_nearfull}. Resizing on Full alert=${grow_pvc_on_full}. Expansion ratio set to ${grow_rate} times."

i=0
alertmanagerroute=$(oc get route -n openshift-monitoring | grep alertmanager-main | awk '{ print $2 }')
curl -sk -H "Authorization: Bearer $(oc sa get-token prometheus-k8s -n openshift-monitoring)"  https://${alertmanagerroute}/api/v1/alerts | jq -r '.' >./tt.txt
export total_alerts=$(cat ./tt.txt | jq '.data | length')
echo "Looping at $(date +"%Y-%m-%d %H:%M:%S")"

while true
do
    export entry=$(cat ./tt.txt | jq ".data[$i]")
    thename=$(echo $entry | jq -r '.labels.alertname')
    if [ x"${thename}" = "xPersistentVolumeUsageNearFull" ]
    then
#       echo $entry
       if [ "x${grow_pvc_on_nearfull}" = "xYes" ]
       then
          ns=$(echo $entry | jq -r '.labels.namespace')
          pvc=$(echo $entry | jq -r '.labels.persistentvolumeclaim')
          echo "Processing NearFull alert for PVC ${pvc} in namespace ${ns}"
          currentsize=$(oc get pvc ${pvc} -n ${ns} -o json | jq -r '.spec.resources.requests.storage')
          echo "PVC current size is ${currentsize}. Will be increased ${grow_rate} times." 
          if [[ "$currentsize" == *"Mi" ]]
          then
             rawsize=$(echo $currentsize | sed -e 's/Mi//g')
             unitsize="Mi"
          elif [[ "$currentsize" == *"Gi" ]]
          then
             rawsize=$(echo $currentsize | sed -e 's/Gi//g')
             unitsize="Gi"
          elif [[ "$currentsize" == *"Ti" ]]
          then
             rawsize=$(echo $currentsize | sed -e 's/Ti//g')
             unitsize="Ti"
          else
             echo "Unknown unit this PVC: ${currentsize}"
          fi
          newsize=$(echo "${rawsize} * ${grow_rate}" | bc | cut -f1 -d'.')
          if [ "${newsize}" = "${rawsize}" ]
          then
             newsize=$(( rawsize + 1 ))
             echo "New adjusted calculated size for the PVC is ${newsize}${unitsize}"
          else
             echo "New calculated size for the PVC is ${newsize}${unitsize}"
          fi
          result=$(oc patch pvc ${pvc} -n ${ns} --type json --patch  "[{ "op": "replace", "path": "/spec/resources/requests/storage", "value": "${newsize}${unitsize}" }]")
          echo ${result}
       else
          ns=$(echo $entry | jq -r '.labels.namespace')
          pvc=$(echo $entry | jq -r '.labels.persistentvolumeclaim')
          echo "NOT processing NearFull alert for PVC ${pvc} in namespace ${ns}"
       fi
    elif [ x"${thename}" = "xPersistentVolumeUsageCritical" ]
    then
#       echo $entry
       if [ "x${grow_pvc_on_full}" = "xYes" ]
       then
          ns=$(echo $entry | jq -r '.labels.namespace')
          pvc=$(echo $entry | jq -r '.labels.persistentvolumeclaim')
          echo "Processing CriticalFull alert for PVC ${pvc} in namespace ${ns}"
          currentsize=$(oc get pvc ${pvc} -n ${ns} -o json | jq -r '.spec.resources.requests.storage')
          echo "PVC current size is ${currentsize}. Will be increased ${grow_rate} times." 
          if [[ "$currentsize" == *"Mi" ]]
          then
             rawsize=$(echo $currentsize | sed -e 's/Mi//g')
             unitsize="Mi"
          elif [[ "$currentsize" == *"Gi" ]]
          then
             rawsize=$(echo $currentsize | sed -e 's/Gi//g')
             unitsize="Gi"
          elif [[ "$currentsize" == *"Ti" ]]
          then
             rawsize=$(echo $currentsize | sed -e 's/Ti//g')
             unitsize="Ti"
          else
             echo "Unknown unit this PVC: ${currentsize}"
          fi
          newsize=$(echo "${rawsize} * ${grow_rate}" | bc | cut -f1 -d'.')
          if [ "${newsize}" = "${rawsize}" ]
          then
             newsize=$(( rawsize + 1 ))
             echo "New adjusted calculated size for the PVC is ${newsize}${unitsize}"
          else
             echo "New calculated size for the PVC is ${newsize}${unitsize}"
          fi
          result=$(oc patch pvc ${pvc} -n ${ns} --type json --patch  "[{ "op": "replace", "path": "/spec/resources/requests/storage", "value": "${newsize}${unitsize}" }]")
          echo ${result}
       else
          ns=$(echo $entry | jq -r '.labels.namespace')
          pvc=$(echo $entry | jq -r '.labels.persistentvolumeclaim')
          echo "NOT processing CriticalFull alert for PVC ${pvc} in namespace ${ns}"
       fi
    else
       if [ "x${grow_debug}" = "x-v" ]
       then
           echo "Alert ${thename} ignored"
           echo "----------"
           echo $entry
           echo "----------"
       fi
    fi
    (( i = i + 1 ))
    if (( i == total_alerts ))
    then
       sleep 300
       rm -f ./tt.txt
       alertmanagerroute=$(oc get route -n openshift-monitoring | grep alertmanager-main | awk '{ print $2 }')
       curl -sk -H "Authorization: Bearer $(oc sa get-token prometheus-k8s -n openshift-monitoring)"  https://${alertmanagerroute}/api/v1/alerts | jq -r '.' >./tt.txt
       total_alerts=$(cat ./tt.txt | jq '.data | length')
       i=0
       echo "Looping at $(date +"%Y-%m-%d %H:%M:%S")"
    fi
done

```
Then give it execution permissions
```console
chmod +x grow_pvc.sh
```

Finally run the script in the background
```console
nohup ./grow_pvc.sh > resizeoutput.log &
```

Note: Make sure to keep your oc session up and running (login to openshift `oc login` in the bastion system frequently)


# MTC Migration Cutover to target cluster

MTC has two main processes:
- Stage
- Cutover

Staging phase goal is to prepare the target environment before the CutOver step. The Stage phase will create the corresponding namespace on the target cluster, then it will create the PVs and PVCs and copy the current data to the target cluster PVs, using whichever copy options were selected in the migration plan.

The stage process does not affect the functionality of the application in the Source cluster, that means that the staging phase can be run incrementally without bringing down the application in the source cluster. Each time the staging phase is executed, MTC will internally evaluate the delta of information missing and it will rcopy it to the target cluster, reducing the time of the cutover considerably.

Cutover on the other hand is the real migration process, This phase by itself will include another stage step to verify the latest data is present in the target cluster, it will halt the application and then it will migrate the custom resources.

Due to some problems while executing staging phase more than one time in the environments during the last migrations, the playbook has been changed to run the CutOver step directly. By this approach there will be only one staging phase involved

Before cutover, validate again that no pods are running in source cluster and target cluster. This is needed to ensure that everything is stopped on source cluster for the cutover, so when the cutover is performed, MTC does not attempt to resume any operations. The services will be resumed manually for CPD in later steps.

Source cluster example: 
```console
$ oc get pod -n <cpd-namespace> |grep -v Completed
NAME                                                      READY   STATUS      RESTARTS   AGE
$
```
Target cluster example:
```console
$ oc get pod -n <cpd-namespace>
No resources found in cpdinstance namespace.
$
```

**Perform the MTC Cutover from source cluster MTC UI:**

Gather target cluster resources before cutover by running the following command: 
```console
$ NSPACE="<cpd-namespace>";oc get $(oc api-resources --namespaced=true --verbs=list -o name | awk '{printf "%s%s",sep,$0;sep=","}')  --ignore-not-found -n ${NSPACE} |grep -v packagemanifest
```

- From Migration Controller, go to "Migration plans". For the CPD migration plan, select the three dots on right hand side and select "Cutover".
<img width="1409" alt="image" src="https://media.github.ibm.com/user/30787/files/680381c5-faf4-4ccf-93a7-c55bb4d9d0d9">

- A warning is displayed that applications will be halted. But this is not an issue, since we manually halted all applications prior to cutover. Make sure `Halt Applications...` option is selected and click `Migrate`
<img width="957" alt="image" src="https://media.github.ibm.com/user/30787/files/b8df46bd-1b9e-407f-887c-62bd9a63f272">

Migration cutover started

![Screenshot 2023-05-15 at 1 05 47 PM](https://media.github.ibm.com/user/67796/files/7c8f9422-b082-40c7-a3b1-4e8650896fbf)

![Screenshot 2023-05-15 at 1 17 27 PM](https://media.github.ibm.com/user/67796/files/11388b2e-65e9-43e5-819f-08f4a885d0ba)

Migration completed: 

![Screenshot 2023-05-15 at 1 53 27 PM](https://media.github.ibm.com/user/67796/files/497490e6-b8d9-42e9-98e7-c66421b070db)

During the migration cutover, an `rsync-server` pod will be created on the target cluster and `rsync` pods will start on the source cluster (similar to migration stage).

rsync-server pod on target cluster
```console
$ oc get pod -n <cpd-namespace> | grep rsync
NAME           READY   STATUS    RESTARTS   AGE
rsync-server   2/2     Running   0          8m16s
```

rsync pods on source cluster
```console
$ oc get pod -n <cpd-namespace> | grep rsync
rsync-4q5gc                                               0/2     Completed   0          89s
rsync-4rbsk                                               0/2     Completed   0          32s
rsync-6vs6g                                               0/2     Completed   0          3m10s
rsync-9jjt7                                               0/2     Completed   0          2m9s
rsync-bf6tl                                               0/2     Completed   0          4m14s
rsync-c547g                                               0/2     Completed   0          3m52s
rsync-cgwqh                                               0/2     Completed   0          109s
rsync-dkhxx                                               0/2     Completed   0          2m49s
rsync-hvd5m                                               0/2     Completed   0          7m45s
rsync-lbxvl                                               0/2     Completed   0          4m36s
rsync-lzdcn                                               0/2     Completed   0          6m32s
rsync-m2g2p                                               0/2     Completed   0          8m2s
rsync-m7wm4                                               0/2     Completed   0          6m56s
rsync-ncbwl                                               0/2     Completed   0          3m31s
rsync-nfn7f                                               0/2     Completed   0          4m59s
rsync-pprkq                                               0/2     Completed   0          2m29s
rsync-qzkrm                                               0/2     Completed   0          6m8s
rsync-rdc54                                               0/2     Completed   0          51s
rsync-rm9mh                                               0/2     Completed   0          5m45s
rsync-sdzck                                               0/2     Completed   0          7m20s
rsync-tw8j7                                               0/2     Completed   0          70s
rsync-w54bc                                               0/2     Completed   0          13s
rsync-xxxvz                                               0/2     Completed   0          5m22s
```

Note: During the Cutover process, you may see an error indicating "Migration may be stuck." creating the rsync pods. Be patient in this phase, as the rsync pods will usually create successfully and the migration activity will complete.
<img width="1393" alt="image" src="https://media.github.ibm.com/user/30787/files/5a0b911d-d43f-4775-9c1f-1c5a221c9fb1">
If the Cutover process fails, check velero status of the failed velero "migration" job.
Example: 
```console
velero describe restore migration-xxx-final-xxx -n openshift-migration
```

reference:https://github.com/redhat-cop/openshift-migration-best-practices/blob/main/05-troubleshooting.md

## Migration Cutover Validation on Target cluster


Gather resources in CPD namespace:
```console
$ oc get zenservice -n <cpd-namespace>
NAME      AGE
lite-cr   40m
$ oc get wkc -n <cpd-namespace>
NAME     VERSION   RECONCILED   STATUS   AGE
wkc-cr   4.6.5                           40m
$ oc get db2ucluster -n <cpd-namespace>
NAME          STATE   MAINTENANCESTATE   AGE
db2oltp-wkc                              40m

$ oc get ws
NAME    VERSION   RECONCILED   STATUS   AGE
ws-cr   6.5.0                           41m

$ oc get sts -n <cpd-namespace>
NAME                             READY   AGE
aiopenscale-ibm-aios-etcd        0/0     42m
aiopenscale-ibm-aios-kafka       0/0     42m
aiopenscale-ibm-aios-redis       0/0     42m
aiopenscale-ibm-aios-zookeeper   0/0     42m
c-db2oltp-wkc-db2u               0/0     42m
dsx-influxdb                     0/0     42m
elasticsearch-master             0/0     42m
rabbitmq-ha                      0/0     42m
redis-ha-server                  0/0     42m
wdp-couchdb                      0/0     42m
wml-cpd-etcd                     0/0     42m
wml-deployment-agent             0/0     42m
zen-metastoredb                  0/0     42m


$ for cpdcrd in $(oc get crd |grep cpd.ibm.com |awk '{print $1}');do echo --- $cpdcrd ---;oc get $cpdcrd -o name |xargs -I{} sh -c "oc get {} -o yaml |grep -i maintenance";done
--- analyticsengines.ae.cpd.ibm.com ---
  ignoreForMaintenance: true
  analyticsengineStatus: InMaintenance
--- ccs.ccs.cpd.ibm.com ---
  ignoreForMaintenance: true
  ccsStatus: InMaintenance
--- datarefinery.datarefinery.cpd.ibm.com ---
  ignoreForMaintenance: true
  datarefineryStatus: InMaintenance
--- datastages.ds.cpd.ibm.com ---
--- db2aaserviceservices.databases.cpd.ibm.com ---
  ignoreForMaintenance: true
  db2aaserviceStatus: InMaintenance
--- db2oltpservices.databases.cpd.ibm.com ---
  ignoreForMaintenance: true
  db2oltpStatus: InMaintenance
--- db2whservices.databases.cpd.ibm.com ---
  ignoreForMaintenance: true
  db2whStatus: InMaintenance
--- ibmcpds.cpd.ibm.com ---
--- iis.iis.cpd.ibm.com ---
  ignoreForMaintenance: true
  iisStatus: InMaintenance
--- notebookruntimes.ws.cpd.ibm.com ---
  ignoreForMaintenance: true
  runtimeStatus: InMaintenance
--- pxruntimes.ds.cpd.ibm.com ---
--- ug.wkc.cpd.ibm.com ---
  ignoreForMaintenance: true
  ugStatus: InMaintenance
--- wkc.wkc.cpd.ibm.com ---
  ignoreForMaintenance: true
  wkcStatus: InMaintenance
--- wmlbases.wml.cpd.ibm.com ---
  ignoreForMaintenance: true
  wmlStatus: InMaintenance
--- woservices.wos.cpd.ibm.com ---
  ignoreForMaintenance: true
  phase: InMaintenance
  wosStatus: InMaintenance
  ignoreForMaintenance: false
--- ws.ws.cpd.ibm.com ---
  ignoreForMaintenance: true
  wsStatus: InMaintenance
--- zenextensions.zen.cpd.ibm.com ---
--- zenservices.zen.cpd.ibm.com ---
  ignoreForMaintenance: true
  zenStatus: InMaintenance

```

No pods found running in CPD namespace
```console
$ oc get pod -n <cpd-namespace>
No resources found in cpd namespace.
```

Note: If running pods related to cronjobs are found in the target namespace, please make sure to delete the cronjobs. They will get recreated once the cpd instance is back up.

Statefulsets will still point to the old storage class (Source storage class used in this example is called `cpd-storage`) 

Example command: 
```
$ oc get sts -n <cpd-namespace> c-db2oltp-wkc-db2u -o yaml | grep <cpd-storage-class>
storageClassName: cpd-storage
```

Migration cutover completed successfully.

# Patch and update new storage class references on target cluster

**Description of why patching is required:**

MTC migration does not scour and patch all k8s resources for storage class references. It will only change the PV/PVC references, then copy over all of the existing k8s resources from the source namespace. However, any CR resources/etc that internally have references to storage classes and PVC claims for storage classes need to be adjusted to fit the new target cluster's storage classes.

Patch all kubernetes resources in the CPD namespace, by changing all references of old storage class(es) to new storage class(es).
- Find all resources with matches for the old storage class. Then strategically edit/change the corresponding k8s resource and change the referenced storage class. 
- **Note:** You must make sure you pick the correct RWO or RWX storage class. Do not find/replace indiscriminately.

## Discover all resources in the cpd instance namespace with storage class references

If an oadp dump of the cpd namespace was collected in the earlier step **"Collect an oadp resources dump of the cpd namespace on source cluster"**, that can be used to discover the resources that will need to be patched ahead of time. Do not attempt to collect an oadp resources dump if MTC was installed after cpd oadp was installed due to conflicts with oadp versions.

Alternatively, the resources can be queried live on either the source or target cluster. 

Run the following command on the target cluster to look through all resources in the cpd namespace to find references to the old storage class, which will need to be patched/replaced with the new storage class(es). (For this command, replace `<cpd-namespace>` and `<source-storage-class>` with your values.)
```console
NSPACE="<cpd-namespace>";SOURCE_SC="<source-storage-class>";for cr in $(oc get $(oc api-resources --namespaced=true --verbs=list -o name | grep -vE "packagemanifests|events|endpointslice|persistentvolumeclaim" | awk '{printf "%s%s",sep,$0;sep=","}') --ignore-not-found -n ${NSPACE} -o name);do cr_grep=$(oc get $cr -oyaml |grep -E "${SOURCE_SC}" |grep -v "{\"apiVersion\":\"");if [[ -n "$cr_grep" ]];then echo $cr;fi;done
```

Example:
```console
$ NSPACE="cpdinstance";SOURCE_SC="cpd-storage";for cr in $(oc get $(oc api-resources --namespaced=true --verbs=list -o name | grep -vE "packagemanifests|events|endpointslice|persistentvolumeclaim" | awk '{printf "%s%s",sep,$0;sep=","}') --ignore-not-found -n ${NSPACE} -o name);do cr_grep=$(oc get $cr -oyaml |grep -E "${SOURCE_SC}" |grep -v "{\"apiVersion\":\"");if [[ -n "$cr_grep" ]];then echo $cr;fi;done
configmap/ibm-cpp-config
configmap/iis-db2u-config
configmap/wdp-profiling-iae-config
configmap/wkc-db2u-config
configmap/zen-lite-operation-configmap
mantaflow.adl.getmanta.com/mantaflow-wkc
analyticsengine.ae.cpd.ibm.com/analyticsengine-sample
statefulset.apps/aiopenscale-ibm-aios-etcd
statefulset.apps/aiopenscale-ibm-aios-kafka
statefulset.apps/aiopenscale-ibm-aios-zookeeper
statefulset.apps/c-db2oltp-iis-db2u
statefulset.apps/c-db2oltp-wkc-db2u
statefulset.apps/dsx-influxdb
statefulset.apps/elasticsearch-master
statefulset.apps/kafka
statefulset.apps/rabbitmq-ha
statefulset.apps/redis-ha-server
statefulset.apps/solr
statefulset.apps/wdp-couchdb
statefulset.apps/zen-metastoredb
statefulset.apps/zookeeper
foundationdbcluster.apps.foundationdb.org/wkc-foundationdb-cluster
ccs.ccs.cpd.ibm.com/ccs-cr
ibmcpd.cpd.ibm.com/ibmcpd-cr
db2oltpservice.databases.cpd.ibm.com/db2oltp-cr
datarefinery.datarefinery.cpd.ibm.com/datarefinery-sample
db2ucluster.db2u.databases.ibm.com/db2oltp-iis
db2ucluster.db2u.databases.ibm.com/db2oltp-wkc
formation.db2u.databases.ibm.com/db2oltp-iis
formation.db2u.databases.ibm.com/db2oltp-wkc
fdbcluster.foundationdb.opencontent.ibm.com/wkc-foundationdb-cluster
iis.iis.cpd.ibm.com/iis-cr
ug.wkc.cpd.ibm.com/ug-cr
wkc.wkc.cpd.ibm.com/wkc-cr
woservice.wos.cpd.ibm.com/aiopenscale
woservice.wos.cpd.ibm.com/openscale-defaultinstance
zenservice.zen.cpd.ibm.com/lite-cr

```

## Execute the cpd-migrate.sh script to patch the storage class references

**Prepare input.json file:**

Before running the cpd-migrate.sh script, prepare the input.json file that will be fed into the cpd-migrate.sh script. This input.json file is expected to have matching json entries for every resource found in the discovery, from the previous step.

The creation of this input.json file was completed by the IBM team and it is called `cpd-migrate.input.json`. It has been provided for use with the `cpd-migrate.sh` script.

An example of the format of the input.json file:
```json
{
  "analyticsengines.ae.cpd.ibm.com": {
    "analyticsengine-sample": [
      {
        "action": "patch",
        "type": "json-path",
        "sourcepath": ".spec.storageClass",
        "sourcevalue": "--source-file-storage-class",
        "targetvalue": "--target-file-storage-class"
      }
    ]
  },
  "statefulsets.apps": {
    "aiopenscale-ibm-aios-etcd": [
      {
        "action": "delete-create",
        "type": "json-path",
        "sourcepath": ".spec.volumeClaimTemplates[0].spec.storageClassName",
        "sourcevalue": "--source-block-storage-class",
        "targetvalue": "--target-block-storage-class"
      }
    ],
    "zen-metastoredb": [
      {
        "action": "delete-create",
        "type": "json-path",
        "sourcepath": ".spec.volumeClaimTemplates[0].spec.storageClassName",
        "sourcevalue": "--source-block-storage-class",
        "targetvalue": "--target-block-storage-class"
      }
    ]
  }
}
```

**Run the cpd-migrate.sh script in preview mode:** 

Run the cpd-migrate.sh in preview mode, which will output json files with the updated storage class references. The resulting json files should be inspected to validate that the new storage class references are accurate and contains valid content to be applied to patch the corresponding resource.

Command (assumes new storage classes are default ODF storage class names):
```console
$ ./cpd-migrate.sh restore \
--input-file cpd-migrate.input.json \
--cpd-operand-namespace <cpd-namespace> \
--source-block-storage-class <source-cluster-nfs-sc> \
--target-block-storage-class ocs-storagecluster-ceph-rbd \
--source-file-storage-class <source-cluster-nfs-sc> \
--target-file-storage-class ocs-storagecluster-cephfs \
--output-directory /tmp/cpd-migrate \
--preview
```

Example command: 
```console
./cpd-migrate.sh restore --input-file cpd-migrate.input.json --cpd-operand-namespace cpdinstance --source-block-storage-class cpd-storage --target-block-storage-class ocs-storagecluster-ceph-rbd --source-file-storage-class cpd-storage --target-file-storage-class ocs-storagecluster-cephfs --preview --output-directory ./preview
```

Example output-directory contents with resulting json files:
```console
$ ls -1
cpdinstance.analyticsengines.ae.cpd.ibm.com.analyticsengine-sample.json
cpdinstance.ccs.ccs.cpd.ibm.com.ccs-cr.json
cpdinstance.configmaps.wdp-profiling-iae-config.json
cpdinstance.configmaps.wkc-db2u-config.json
cpdinstance.configmaps.zen-lite-operation-configmap.json
cpdinstance.datarefinery.datarefinery.cpd.ibm.com.datarefinery-sample.json
cpdinstance.db2uclusters.db2u.databases.ibm.com.db2oltp-iis.json
cpdinstance.db2uclusters.db2u.databases.ibm.com.db2oltp-wkc.json
cpdinstance.fdbclusters.foundationdb.opencontent.ibm.com.wkc-foundationdb-cluster.json
cpdinstance.formations.db2u.databases.ibm.com.db2oltp-iis.json
cpdinstance.formations.db2u.databases.ibm.com.db2oltp-wkc.json
cpdinstance.foundationdbclusters.apps.foundationdb.org.wkc-foundationdb-cluster.json
cpdinstance.ibmcpds.cpd.ibm.com.ibmcpd-cr.json
cpdinstance.iis.iis.cpd.ibm.com.iis-cr.json
cpdinstance.mantaflows.adl.getmanta.com.mantaflow-wkc.json
cpdinstance.statefulsets.apps.aiopenscale-ibm-aios-etcd.json
cpdinstance.statefulsets.apps.aiopenscale-ibm-aios-kafka.json
cpdinstance.statefulsets.apps.aiopenscale-ibm-aios-zookeeper.json
cpdinstance.statefulsets.apps.c-db2oltp-iis-db2u.json
cpdinstance.statefulsets.apps.c-db2oltp-wkc-db2u.json
cpdinstance.statefulsets.apps.dsx-influxdb.json
cpdinstance.statefulsets.apps.elasticsearch-master.json
cpdinstance.statefulsets.apps.kafka.json
cpdinstance.statefulsets.apps.rabbitmq-ha.json
cpdinstance.statefulsets.apps.redis-ha-server.json
cpdinstance.statefulsets.apps.solr.json
cpdinstance.statefulsets.apps.wdp-couchdb.json
cpdinstance.statefulsets.apps.zen-metastoredb.json
cpdinstance.statefulsets.apps.zookeeper.json
cpdinstance.ug.wkc.cpd.ibm.com.ug-cr.json
cpdinstance.wkc.wkc.cpd.ibm.com.wkc-cr.json
cpdinstance.woservices.wos.cpd.ibm.com.aiopenscale.json
cpdinstance.woservices.wos.cpd.ibm.com.openscale-defaultinstance.json
cpdinstance.zenservices.zen.cpd.ibm.com.lite-cr.json
```


**Run the cpd-migrate.sh script to perform the resource updates:** 

After running cpd-migrate.sh in preview mode, and inspecting and validating the resulting json files, run the cpd-migrate.sh script without the preview option. This will take action on the kubernetes resources in the actual cluster, updating the resources with storage class references in the cpd instance namespace.

**Note:** There will be a small subset of resources that will have actions of "ignore" or "manual". This means that the cpd-migrate.sh tool will not take action on those resources. 
- For a resource with action of "ignore", that indicates that no action is required and the existing reference to the old storage class can remain.

Example of "ignore" action: 
```console
--------------------------------------------------
Time: 2023-05-31T06:05:41.543+0000 level=info - patch-resource: cpdinstance, configmaps, ibm-cpp-config, [{"action":"ignore","type":"json-path","sourcepath":".data.storageclass.default","sourcevalue":"--source-block-storage-class","targetvalue":""},{"action":"ignore","type":"json-path","sourcepath":".data.storageclass.list","sourcevalue":"--source-block-storage-class","targetvalue":""}]

Time: 2023-05-31T06:05:43.186+0000 level=info - Migrate Action: {"action":"ignore","type":"json-path","sourcepath":".data.storageclass.default","sourcevalue":"--source-block-storage-class","targetvalue":""}
Time: 2023-05-31T06:05:43.497+0000 level=warning - Ignore .data.storageclass.default: "cpd-storage"

Time: 2023-05-31T06:05:43.500+0000 level=info - Migrate Action: {"action":"ignore","type":"json-path","sourcepath":".data.storageclass.list","sourcevalue":"--source-block-storage-class","targetvalue":""}
Time: 2023-05-31T06:05:43.809+0000 level=warning - Ignore .data.storageclass.list: "cpd-storage"

Time: 2023-05-31T06:05:43.812+0000 level=info - configmaps ibm-cpp-config - No Change
--------------------------------------------------
```

- For a resource with action of "manual", that indicates that the cpd-migrate.sh tool is not prepared to patch that resource. The user must manually edit/change the resource (i.e. `oc edit <kind>/<name>`; search for the source storage class name; replace with appropriate target storage class name).

Example of "manual" action: 
```console
--------------------------------------------------
Time: 2023-05-31T06:05:39.575+0000 level=info - patch-resource: cpdinstance, configmaps, db2oltp-1685056932270995-db2oltp-cm, [{"action":"manual","type":"string","sourcepath":".data.genkeys.sh","sourcevalue":"--source-file-storage-class","targetvalue":"--target-file-storage-class"}]

Time: 2023-05-31T06:05:41.176+0000 level=info - Migrate Action: {"action":"manual","type":"string","sourcepath":".data.genkeys.sh","sourcevalue":"--source-file-storage-class","targetvalue":"--target-file-storage-class"}
Time: 2023-05-31T06:05:41.492+0000 level=warning - Manual Edit Required .data.genkeys.sh: "cpd-storage"

Time: 2023-05-31T06:05:41.494+0000 level=info - configmaps db2oltp-1685056932270995-db2oltp-cm - No Change
--------------------------------------------------
```

Command (assumes new storage classes are default ODF storage class names):
```console
$ ./cpd-migrate.sh restore \
--input-file cpd-migrate.input.json \
--cpd-operand-namespace <cpd-namespace> \
--source-block-storage-class <source-cluster-nfs-sc> \
--target-block-storage-class ocs-storagecluster-ceph-rbd \
--source-file-storage-class <source-cluster-nfs-sc> \
--target-file-storage-class ocs-storagecluster-cephfs \
--output-directory /tmp/cpd-migrate
```

**Validate patching completed successfully**

After the patching activities, validate that the cpd namespace has no more references to the source cluster's storage class. With the exception of the resources that the cpd-migrate.sh tool identified as "ignore" or "manual", which need to be handled manually.

Run the following command on the target cluster to query for any remaining references to the old storage class, and handle appropriately:
```console
NSPACE="<cpd-namespace>";SOURCE_SC="<source-storage-class>";for cr in $(oc get $(oc api-resources --namespaced=true --verbs=list -o name | grep -vE "packagemanifests|events|endpointslice|persistentvolumeclaim" | awk '{printf "%s%s",sep,$0;sep=","}') --ignore-not-found -n ${NSPACE} -o name);do cr_grep=$(oc get $cr -oyaml |grep -E "${SOURCE_SC}" |grep -v "{\"apiVersion\":\"");if [[ -n "$cr_grep" ]];then echo $cr;fi;done
```

# Start Cloud Pak for Data instance on target cluster

## Prepare target cluster for Cloud Pak for Data
Before resuming CPD and services on the target cluster, make sure it has been prepared for CPD. Create custom SCCs and change required node settings.
Make sure also that proper credentials are set in the pull-secret for the repository/artifactory that holds the images
Reference: https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=installing-preparing-your-cluster

Examples:
- Db2 kubeletconfig: `cpd-cli manage apply-db2-kubelet --openshift-type=self-managed`
- WKC SCC: `cpd-cli manage apply-scc --cpd_instance_ns=<cpd-namespace> --components=wkc`
- CRI-O settings: `cpd-cli manage apply-crio --openshift-type=self-managed`

## Start services on the target cluster that were manually shutdown on source cluster

For the services and resources that were manually shutdown in the **"Scale down remaining services and resources running in CPD namespace"** step, startup those services now.

- Examples: nginx deployment, db2ucluster statefulsets, foundationdb, MANTA, openscale, etc.


**nginx deployment (scale back up to 2):**
```console
oc scale --replicas=2 deploy/ibm-nginx -n <cpd-namespace>
```

**MANTA scale up replicas (back to 1):**
```console
oc patch -p '{"spec":{"replicas":1}}' --type=merge mantaflow mantaflow-wkc -n <cpd-namespace>
```

**Openscale startup:**
```console
oc patch -p '{"spec":{"shutdown":"false"}}' --type=merge woservice aiopenscale -n <cpd-namespace>
oc patch -p '{"spec":{"ignoreForMaintenance": false}}' --type=merge woservice aiopenscale -n <cpd-namespace>
```

**Foundationdb startup:**
```console
oc patch -p '{"spec":{"shutdown":"false"}}' --type=merge fdbcluster wkc-foundationdb-cluster -n <cpd-namespace>
```

**db2ucluster statefulsets (scale back up to 1):**
```console
oc scale --replicas=1 sts/c-db2oltp-iis-db2u -n <cpd-namespace>
oc scale --replicas=1 sts/c-db2oltp-wkc-db2u -n <cpd-namespace>

oc scale --replicas=1 sts/<db2wh-db2u-sts> -n <cpd-namespace>
oc scale --replicas=1 sts/<db2wh-etc-sts> -n <cpd-namespace>

```

Note: If there were additional db2u related statefulsets that were scaled down, scale them up here to the same replica number as it was on the source cluster.

**Datastage deploy and statefulset**
``` console
## Record this output for the resume operations:
oc get deploy,sts -n <cpd-namespace>|grep ds-px-default 

oc scale --replicas=1 deploy/ds-px-default-ibm-datastage-px-runtime -n <cpd-namespace>
oc scale --replicas=1 sts/ds-px-default-ibm-datastage-px-compute -n <cpd-namespace>
```

**Remaining resources:**
If there were additional services, deployments, statefulsets, or other that were manually shutdown on the source cluster, resume them here.


**Examples of starting the resources:**

**Scale up nginx deployment**
```console
$ oc get deploy -n cpd |grep -E "NAME|nginx" 
NAME                                         READY   UP-TO-DATE   AVAILABLE   AGE
aiopenscale-ibm-aios-nginx                   0/0     0            0           2d13h
ibm-nginx                                    0/0     0            0           2d13h
ibm-nginx-tester                             0/0     0            0           2d13h
spark-hb-nginx                               0/0     0            0           2d13h
$ oc scale --replicas=2 deploy/ibm-nginx -n cpd
deployment.apps/ibm-nginx scaled
$ oc get deploy ibm-nginx -n cpd
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
ibm-nginx   2/2     2            2           2d13h
$ oc get pod -n cpd
NAME                         READY   STATUS    RESTARTS   AGE
ibm-nginx-7cd879b767-fjgn4   1/1     Running   0          4m47s
ibm-nginx-7cd879b767-fn22k   1/1     Running   0          4m47s
```

**Scale up db2u statefulsets**

```console
$ oc -n cpd get sts -n cpd |grep -E "NAME|db2"
NAME                                     READY   AGE
c-db2oltp-iis-db2u                       0/0     2d18h
c-db2oltp-wkc-db2u                       0/0     8d
c-db2wh-1686277435040922-db2u            0/0     15h
c-db2wh-1686277435040922-etcd            0/0     15h

$ oc scale --replicas=1 sts/c-db2oltp-wkc-db2u -n cpd
statefulset.apps/c-db2oltp-wkc-db2u scaled
$ oc scale --replicas=1 sts/c-db2oltp-iis-db2u -n cpd
statefulset.apps/c-db2oltp-iis-db2u scaled
$ oc scale --replicas=1 sts/c-db2wh-1686277435040922-db2u -n cpd
statefulset.apps/c-db2wh-1686277435040922-db2u scaled
$ oc scale --replicas=1 sts/c-db2wh-1686277435040922-etcd -n cpd
statefulset.apps/c-db2wh-1686277435040922-etcd scaled

$ oc -n cpd get pods|grep db2
c-db2oltp-iis-db2u-0                                      1/1     Running     0             44h
c-db2oltp-iis-instdb-sx2hm                                0/1     Completed   0             2d19h
c-db2oltp-wkc-db2u-0                                      1/1     Running     0             44h
c-db2wh-1686277435040922-db2u-0                           1/1     Running     0             15h
c-db2wh-1686277435040922-etcd-0                           1/1     Running     0             15h
```

**Start foundationdb**

Check foundationdb CR state from CPD namespace: 
```console
$ oc -n cpd get fdbcluster wkc-foundationdb-cluster -o yaml |grep -E "ceph|shutdown"

    pvcStorageClass: ocs-storagecluster-cephfs
            storageClassName: ocs-storagecluster-ceph-rbd
            storageClassName: ocs-storagecluster-ceph-rbd
  shutdown: "true"
  shutdownStatus: shutdown
```

Start foundationdb-cluster CR from CPD namespace: 
```console
$ oc -n cpd get fdbcluster wkc-foundationdb-cluster -o json |jq '.spec.shutdown'
"true"
$ oc -n cpd edit fdbcluster wkc-foundationdb-cluster
fdbcluster.foundationdb.opencontent.ibm.com/wkc-foundationdb-cluster edited
$ oc -n cpd get fdbcluster wkc-foundationdb-cluster -o json |jq '.spec.shutdown'
"false"
$ 

$ oc get pods -n cpd
NAME                                            READY   STATUS    RESTARTS   AGE
c-db2oltp-wkc-db2u-0                            1/1     Running   0          35m
ibm-nginx-7cd879b767-fjgn4                      1/1     Running   0          45m
ibm-nginx-7cd879b767-fn22k                      1/1     Running   0          45m
wkc-foundationdb-cluster-cluster-controller-1   2/2     Running   0          18m
wkc-foundationdb-cluster-log-1                  2/2     Running   0          18m
wkc-foundationdb-cluster-log-2                  2/2     Running   0          18m
wkc-foundationdb-cluster-log-3                  2/2     Running   0          18m
wkc-foundationdb-cluster-log-4                  2/2     Running   0          18m
wkc-foundationdb-cluster-storage-1              2/2     Running   0          18m
wkc-foundationdb-cluster-storage-2              2/2     Running   0          18m
wkc-foundationdb-cluster-storage-3              2/2     Running   0          18m
wkc-foundationdb-cluster-storage-4              2/2     Running   0          18m
wkc-foundationdb-cluster-storage-5              2/2     Running   0          18m
wkc-foundationdb-cluster-storage-6              2/2     Running   0          18m
```

When FDBcluster is manually shutdown the Database gets locked, in order for `wkc-lineage` and `wkc-ingestion` pods to start successfully the following procedure need to be followed
reference: https://github.ibm.com/foundationdb/fdb-cp4d/wiki/Service-Shutdown-Feature

Step1: Run an oc command to get the uid from the annotation:
```console
   oc get fdbcluster/sample-cluster -o yaml | grep -i lockUID
   eg. oc get fdbcluster/sample-cluster -o yaml | grep -i lockUID
   lockUID: 61712bc9bb33f6feaf6750ce8ff3458c
```
Step2: Login to any one of the database pod's terminal
```console
oc exec -it <fdb-database-pod> bash
```
Step3: Type the following at the command prompt:
```console
   $ lockid={uid} (obtained from Step 1)
   $ fdbcli --exec "unlock $lockid"
   The fdbcli will prompt you with something like this:
   Unlocking the database is a potentially dangerous operation.
   Repeat the following passphrase if you would like to proceed ({pass-phase}):
   Please type in the {pass-phase} when prompted to unlock the database
   eg.
   Repeat the following passphrase if you would like to proceed (LCTnl4R3rR) : LCTnl4R3rR
   Database unlocked.
```

**Start MANTA**

From CPD namespace, set replicas to 1:
```console
$ oc -n cpd get mantaflow mantaflow-wkc -o json |jq '.spec.replicas'
0
$ oc -n cpd edit mantaflow mantaflow-wkc
mantaflow.adl.getmanta.com/mantaflow-wkc edited
$ oc -n cpd get mantaflow mantaflow-wkc -o json |jq '.spec.replicas'
1
```

**Start Openscale:**

From CPD namespace, set shutdown=false in the two openscale CR instances:
```console
$ oc patch WOService  openscale-defaultinstance -n cpd --type merge --patch '{"spec": {"shutdown":"false"}}'
woservice.wos.cpd.ibm.com/openscale-defaultinstance patched
$ oc -n cpd patch WOService aiopenscale -n cpd --type merge --patch '{"spec": {"shutdown": "false"}}'
woservice.wos.cpd.ibm.com/aiopenscale patched
$ oc -n cpd get WOService  openscale-defaultinstance -o json | jq '.spec.shutdown'
"false"
$ oc -n cpd get WOService aiopenscale -o json | jq '.spec.shutdown'
"false"
$ 
```
**Datastage**
```console
$ oc scale --replicas=1 deploy/ds-px-default-ibm-datastage-px-runtime -n cpd
deploy.apps/ds-px-default-ibm-datastage-px-runtime scaled
$ oc scale --replicas=1 sts/ds-px-default-ibm-datastage-px-compute -n cpd
statefulset.apps/ds-px-default-ibm-datastage-px-compute scaled

```

## Resume CPD service operations with restore posthooks

The cpd-cli oadp restore posthooks will take all of the CPD services out of maintenance mode and scale back up all of the deployments, statefulsets, etc.

Example Command:
```console
cpd-cli oadp restore posthooks --include-namespaces <cpd-namespace> --log-level=debug --verbose --scale-wait-timeout
```

Run the following commands to verify CPD instance is running:
```console
$ oc get pods -n <cpd-namespace>
$ oc get route -n <cpd-namespace>
$ oc get pvc -n <cpd-namespace>
$ oc get sts -n <cpd-namespace>
$ oc get deployments -n <cpd-namespace>
```

Get CR status: 
```console
$ cpd-cli manage get-cr-status --cpd_instance_ns=<cpd-namespace>
$ oc get ccs -n <cpd-namespace>
$ oc get wkc wkc-cr -o yaml -n <cpd-namespace>
```

**Validate that all installed CPD services resume to Completed status:**
```console
cpd-cli manage get-cr-status --cpd_instance_ns=<cpd-namespace>
```

After the command completes successfully, the migration restore is now complete!

---

# CPD Troubleshooting 

This section highlights the different procedures applied during the migration performed in the environment. 

## CRD conflicts between OADP installations

If more than one OADP operator is installed in the system, the last operator installed will take precedence. This is due to a conflict in the CRDs. 

The workaround applied for this conflict was to reinstall the OADP operator for the version that needs to be used for the expected operation. This can be done the Openshift console>Installed Operators and Openshift console>Operator Hub

For example.

If CPD OADP operator is installed and MTC (which contains another version of OADP) is installed. The CPD OADP operator will not find the correct CRDs to run properly, this can be seen in the cluster by the cpd-cli oadp commands getting stuck. Solution would be to uninstall and reinstall OADP operator for cpd instance. This will not affect the setup It will only update the CRDs

## OOMKilled during the cpd instance unquiesce 

This error was seen during one of the unquiesce process in the cpd instance. 

This error was seen in the REDIS pods, but this error is common when having loaded environments

|  Cause | Solution  |  
|---|---|
| Container memory limit was reached, and the application is experiencing higher load than normal  | Increase memory limit in pod specifications  |
| Container memory limit was reached, and application is experiencing a memory leak  | Debug the application and resolve the memory leak  |
| Node is overcommitted—this means the total memory used by pods is greater than node memory  | Adjust memory requests (minimal threshold) and memory limits (maximal threshold) in your containers |

The best way to find out this error is to check the failing pod description, specifically the containers section 

Debug with following command:
```
oc describe <failing-pod>
```

Solution in the environment was to increase the memory limit in the REDIS deployment by 2x
```
oc edit deploy <redis-deployment>
```
Note: If a pod other than redis is having issues, the user needs to find the parent resources for that pod and apply the modification accordingly (deploy,sts,rs,ds)

## WKC post install pod in Error state

This job could be constantly failing the fix for this issue is involves modifying the configmap to force it to complete successfully.

Debug problem
Example command:
```
# oc logs -f wkc-post-install-init-28kt2
[16-June-2023 16:17 UTC] INFO: Getting access token
[16-June-2023 16:17 UTC] INFO: Check status of categories bootstrap process...
[16-June-2023 16:17 UTC] INFO: Categories bootstrap process status - SUCCEEDED
[16-June-2023 16:17 UTC] INFO: Skipping categories bootstrap process...
[16-June-2023 16:17 UTC] INFO: Calling BG api to create OOTB artifacts
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   296  100   296    0     0     16      0  0:00:18  0:00:18 --:--:--    36
[16-June-2023 16:17 UTC] ERROR: v3/glossary_terms/admin/initialize_content call did not return 200 http code, instead it returned : 409
[16-June-2023 16:17 UTC] ERROR: Output was: { "trace": "d392e8c1-35cb-4b8f-b847-1f3482ee0327", "errors": [ { "code": "conflict", "message": "WKCBG2352E: Uniqueness constraint for type data_class was violated. WKCBG0001I: Need more help? Contact us with this support ID: d392e8c1-35cb-4b8f-b847-1f3482ee0327." } ] }
```

To solve this problem run
```
oc edit configmap <wkc-post-install-job>
```

Find the script being executed by the job and look for `202`. The fix implied to also include `409` Return code as part of the if statement like below
```
 if [ "$http_code" != "409" ] && [ "$http_code" != "200" ]; then
          LogError " v3/glossary_terms/admin/initialize_content call did not return 200 http code, instead it returned : $http_code"
          response=$(cat /tmp/initialize_content)
          LogError "Output was: $response"
          exit 1
    else
       LogInfo "WKC post install config finished successfully"
    fi
```
Note: Avoid restating the WKC operator until pod is completed

## Data Privacy was not scaled down properly by the cpd prehooks

As part of the prehooks (scale down) process we noticed that this service was left up running. 

@Andy Streit CPD arquitect and B/R expert found that the problem was an incomplete installation of Data privacy (probably due to a POC) and due to that the configmap for data privacy was missing 

The solution was:

Scale down operator deployment 
```
oc -n <cpd-operator-namespace> scale deployment <data-privacy-operator-deployment> --replicas=0
```
Put the Data Privacy CR in maintenance mode
```
oc - <cpd-instance-namespace> edit dp 
```
In the `spec` section put the following flag
```
ignoreForMaintenance: "true"
```
Note: be careful with the yaml spacing


## CCS stuck in Inprogress/Failed state while bringing the environment up in the target cluster

Debug:
Check the ccs operator logs
```
oc -n <cpd-operator-namespace> logs <ccs-operator-logs>
```
If an error like below is displayed, This is due to a known issue in CCS 
![Pasted Graphic 11](https://media.github.ibm.com/user/242195/files/26ebdd5f-9f7c-46ed-8b50-36fc2d361787)

The fix was to leave CCS in Maintenance mode From 4.6.5 onwards CCS Maintenance mode does not affect the status of the dependat services 

Put the CCS  CR in maintenance mode
```
oc - <cpd-instance-namespace> edit ccs 
```
In the `spec` section put the following flag
```
ignoreForMaintenance: "true"
```

# Migration Toolkit Lessons learned and troubleshooting

MTC migration will copy the PVC definition and PV data froum the source cluster to target cluster, it will also migrate the images used by the application, finally MTC will migrate the custom resources. The first two operations are run asynchronously, and the user has two options, by either using an S3 bucket to migrate the data and then restoring it in the target system or by using direct migration. If the user decides to use the direct migration the process will speed up considerably. 

MTC operations are divided in two main activities. 

**Stage phase**

Stage phase will create the namespace and copy the data (PVC and PV) from Source cluster to Target cluster without stopping the application, aditionally to this, the staging phase will also copy and restore the images of the application.  This step is mainly used to speed up things when the Cut Over phase is executed and it will not impact the functionality of the source application.

The stage step can be repeated multiple times, MTC will internally evaluate the data present in the PVs and if a delta is found, MTC will rsync the missing information. Repeating the staging phase can reduce the cut over time considerably

**Cut Over phase**

Cut over phase is the actual migration of the application from source to target cluster. This step will execute another stage operation in order to make sure all the latest information is available in the target cluster, Finally it will Migrate the application custom resources. This process will halt the application in the source cluster


### About Direct Volume/Image migration

When a stage operation is executed the MTC daemon set of the source cluster will create an rsync-server pod in the target cluster, and in the source cluster the user will see a lot of rsync pods created that will be copying the data

The operation that most affects the speed of the execution is the copy/restore of the data between the clusters and the S3 bucket. That is why it is highly recommended to enable both the `direct volume migration` and the `direct image migration` if the network allows it

Direct volume migration is enabled if both clusters can discover each other in the network and the checkmark is enabled in the Migration plan (migplan)

Direct image migration is enabled if both the source cluster and the target repository/artifactory can be discovered in the network and the checkmark is enabled in the migration plan.

In order to enable the DIM you have to add the target repository/artifactory hostname when you configure the target cluster like below

![image](https://media.github.ibm.com/user/242195/files/9d52f155-72fd-4a0a-9d20-478170bbe3b2)


![image](https://media.github.ibm.com/user/242195/files/c6cdd3fa-0b07-4133-a467-d7a439440229)

## MTC Troubleshooting section

During migration a few issues were found, This section try to explain and document the workarounds applied for future reference

**SELinux relabeling and MigController flag**

Cloud Pak for Data application in environment has a lot of PVCs/PVs and some of them has big chunk of data and/or files created. In MTC during the direct volume migration one rsync server pod is created in the target cluster. This rsync server pod needs to mount all the PVCs in the application in order to evaluate if there is information missing in them.

Because of this if more than one stage operation is executed. There is a possibility that the `rsync-server` pod in the target cluster is not able to start. In order to fix this problem the Red Hat/MTC teams have provided the following fix:

references:
https://access.redhat.com/solutions/6499541
https://access.redhat.com/solutions/6221251

As a symptom you will see the following error in the pod description
```console

oc describe pod <rsync-server> 
....
..
.
Error: kubelet may be retrying requests that are timing out in CRI-O due to system load. Currently at stage container volume configuration: context deadline exceeded: error reserving ctr name k8s_rsync_rsync-server_hptv-stgcloudpak_2aa85550-1bed-4db7-9ade-71962249fb83_0 for id f78a59a76c10a4ea0e0c4ad0f6bcb825e330c378b3ad7fbdee3b63604f041067: name is reserved
```

In order to fix this problem you need to:

1. Apply the following machine configuration
```yaml
cat <<EOF|oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-selinux-configuration
spec:
  config:
    ignition:
      version: 3.2.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,W2NyaW8ucnVudGltZS5ydW50aW1lcy5zZWxpbnV4XQpydW50aW1lX3BhdGggPSAiL3Vzci9iaW4vcnVuYyIKcnVudGltZV9yb290ID0gIi9ydW4vcnVuYyIKcnVudGltZV90eXBlID0gIm9jaSIKYWxsb3dlZF9hbm5vdGF0aW9ucyA9IFsiaW8ua3ViZXJuZXRlcy5jcmktby5UcnlTa2lwVm9sdW1lU0VMaW51eExhYmVsIl0K
        mode: 0640
        overwrite: true
        path: /etc/crio/crio.conf.d/01-selinux.conf
  osImageURL: ""
EOF
```
2. Create a runtime class 
```yaml
cat <<EOF|oc apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: selinux
handler: selinux
```
3. Set the following flag in the spec section of the MigController in MTC **(IMPORTANT this flag is available from MTC >1.7.10)**
```yaml
migration_rsync_super_privileged: true
```
example
```yaml
oc -n openshift-migration edit migrationcontrollers
....
spec:
  migration_rsync_super_privileged: true
  azure_resource_group: ""
  cluster_name: host
  mig_namespace_limit: "10"
  mig_pod_limit: "100"
  mig_pv_limit: "100"
  migration_controller: true
  migration_log_reader: true
  migration_rsync_privileged: true
  migration_ui: true
  migration_velero: true
  olm_managed: true
  restic_timeout: 1h
  version: 1.7.11
```

If the same problem is hit in other pod other than MTC the flag needs to be setup in the definition of the pod's parent resource
```yaml
metadata:
  name: sandbox
  annotations:
    io.kubernetes.cri-o.TrySkipVolumeSELinuxLabel: "true"
```

**How to avoid running pvc and image migration in the cutover**

During migration we hit one problem in which during staging phase the rsync server pods took a lot of time to copy all the PVC data, but it eventually succeded. We had shutdown the CPD instance prior the staging phase so We were completely sure there was no more PVC information that needed to be copied. Besides that team had copied all the images to the target repository already. So we could safely skip these two steps in the cutover and just go direclty to the migration of the CRs

To do that we modified the migrationcontroller  and added the following flags in the spec section and then ran the cutover
```yaml
disable_image_copy: true
disable_image_migration: true
```
example

```yaml
oc -n openshift-migration edit migrationcontrollers
....
spec:
  disable_image_copy: true
  disable_image_migration: true
  disable_pv_migration: true
  migration_rsync_super_privileged: true
  azure_resource_group: ""
  cluster_name: host
  mig_namespace_limit: "10"
  mig_pod_limit: "100"
  mig_pv_limit: "100"
  migration_controller: true
  migration_log_reader: true
  migration_rsync_privileged: true
  migration_ui: true
  migration_velero: true
  olm_managed: true
  restic_timeout: 1h
  version: 1.7.11
```

**rsync (migration) pods stuck in the target cluster**

During the migration staging phase the rsync pod could not initialize . By checking the description of the failing pod, it was reporting an schedule problem because. The reason was that the cpd instance in the source cluster was running with a node annotation (cpd deployment isolated in a restricted number of nodes)

The fix involved labeling the nodes in the target cluster with the same label of the source cluster
```console
oc label node <node> <label> --overwrite
```

## Useful MTC debug commands

Check overall migration logs

```console
oc -n openshift-migration logs $(oc -n openshift-migration get pods -llogreader=mig -ojsonpath='{.items[*].metadata.name}') color
```

Check direct volume migration progress
```console
oc -n openshift-migration describe directvolumemigrationprogresses <dvmp>
```

Check direct volume migration state
```console
oc -n openshift-migration describe directvolumemigrations <dvm>
```

If user is using direct image migration

Check direct image migration progress
```console
oc -n openshift-migration describe directimagestreammigrations <dism>
```

Check direct image migration state

```console
oc -n openshift-migration describe directimagemigrations <dim>
```

If user is using S3 to migrate images and a failure is hit

Check velero backup commands (source cluster)
```console
oc -n openshift-migration exec -it $(oc -n openshift-migration get pods -ldeploy=velero -ojsonpath='{.items[*].metadata.name}') -- ./velero backup describe
```

Check velero restore commands (target cluster)
```console
oc -n openshift-migration exec -it $(oc -n openshift-migration get pods -ldeploy=velero -ojsonpath='{.items[*].metadata.name}') -- ./velero restore describe
```
