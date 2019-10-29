# CephFS CSI deployment

This project deploys a cephfs-csi driver in Openshift.
The Cephfs-csi driver takes care of provision CephFS PersistentVolumes objects based on incoming PersistentVolumeClaims.
The CSI CephFS deployment integrates the following components:

- Manila CSI provisioner, in charge of creating/deleting CephFS shares via the Openstack Manila API (creates PVs from user PVCs)
- CephFS CSI driver itself is in charge of mounting the CephFS PVs on nodes as required by applications
- Storageclass definitions, which expose the ability to request CephFS PVCs to users

Info in [deploy-cephfs](https://github.com/ceph/ceph-csi/blob/master/docs/deploy-cephfs.md)
and in here, [cloud infra docs](https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/using-manila-provisioner.md#authentication-with-manila-v2-client)

## Deployment details

The cephfs-csi driver is deployed with Helm in each Openshift environment, in the `paas-infra-cephfs` namespace.
The various Openshift environments are defined in [.gitlab-ci.yml](.gitlab-ci.yml).

### Permission initialization for new volumes

We need this permission initialization for CephFS volumes because the volumes provisioned by Manila only give write access to `root`, while OpenShift apps run
as `non-root` users. So with this, we make sure that an OpenShift application will be able to write to `CephFS` volumes.

This [`init-permissions` controller](https://gitlab.cern.ch/paas-tools/storage/init-permission-cephfs-volumes) watches for persistent volume events. When a PV is created,
if its storageClass has an annotation `init-permission-volumes.cern.ch/init-permission=true`, the controller grants permissions to it. This happens by launching a
[Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) that mounts the CephFS volume into a pod and
[runs `chown`/`chmod` on the volume's root](https://its.cern.ch/jira/browse/CIPAAS-543). Once this is done, the volume is unmounted
and the Job deleted. The application that requested that volume can now mount the volume and write to it with its Openshift-assigned non-root user.

### Reclaim CephFS volumes after a grace period

By default, the `persistentVolumeReclaimPolicy` of a CephFS PV creation is set to `Retain`, this means that even when the user deletes this PV,
it will persist in our infrastructure in order to recover PVs unfortunately removed by mistake.

In order to delete PVs marked for deletion, we have created a cronjob in charge of deleting permanently PVs marked for deletion
in a specific period of time, this `time-reclaim` is set in each of the StorageClasses [CIPAAS-542](https://its.cern.ch/jira/browse/CIPAAS-542)

Once this time-reclaim is reached, the cronJob patches the `spec` of the PV and sets the `persistentVolumeReclaimPolicy` to `Delete`,
this will trigger a permanently deletion of the PV.

### Backup CephFS persistent volumes

We need to backup CephFS persistent volumes. By default we do not have any automatically backup solution.

This [backup solution](https://gitlab.cern.ch/paas-tools/storage/backup-cephfs-volumes) backs up the CephFS persistent volumes.
The way it works is filtering the volumes with label `backup-cephfs-volumes.cern.ch/backup=true`.
When a PV has this label specified, a `redis` queue is filled out with all the `json` information about the PV.
Once this process success, we process the elements of the queue with a job. This job creates a pod per each PV `json` and mount the PV into the pod.
When the PV is mounted into the pod, the backup solution (which use [`restic`](https://restic.net/) under the hood) starts to back up each PV.

Once this PV is backed up, we add some annotations into the PV to check whether it succeeded or it failed to back up the PV.

Brief summary:

- The back ups are backed up using S3
- The persistent volumes are going to be backed up once a day

## Deploying to a new environment/cluster

### Prerequisites

NB: the [`container_use_cephfs` SELinux boolean](https://bugzilla.redhat.com/show_bug.cgi?id=1692369) must be
enabled on all Openshift nodes for the containers to have access to CephFS mounts.

Some preparation steps need to be performed with cluster-admin permissions.
This needs to be done only once per Openshift cluster/environment.

Start a Docker container for installation (mounting this git repo in /project):

```bash
docker run --rm -it -v $(pwd):/project -e TILLER_NAMESPACE=paas-infra-cephfs gitlab-registry.cern.ch/paas-tools/openshift-client:v3.11.0 bash
```

Then in that container:

```bash
# login as cluster admin to the destination cluster
oc login https://openshift-test.cern.ch #or openshift-dev.cern.ch or openshift.cern.ch
export HELM_VERSION=2.13.1 # make sure to use the same Helm version found in .gitlab-ci.yml
curl "https://kubernetes-helm.storage.googleapis.com/helm-v${HELM_VERSION}-linux-amd64.tar.gz" | tar zx
mv --force linux-amd64/{helm,tiller} /usr/bin/
# create namespace and serviceaccounts for deployment
helm template /project/prerequisites/ | oc create -f -
# obtain the token for the Tiller service account
oc serviceaccounts get-token -n paas-infra-cephfs tiller
```

The rest of the deployment will be handled by GitLab CI.
We need to create 2 [GitLab CI variables](https://docs.gitlab.com/ee/ci/variables/#via-the-ui) in this project
for each Openshift environment (see environment definitions in [.gitlab-ci.yml](.gitlab-ci.yml) for the exact
variable names for each env):

1. a variable `TILLER_TOKEN_<ENV>` with the token for the `tiller` service account obtained above
2. a variable `HELM_VALUES_<ENV>` with the Openstack project to be used by that Openshift environment to
   provision Manila CephFS shares, and the credentials to use, in the form of a Helm values file as follows:

   ```
   {"openstack":{"userName":"paasmanilaXXXX","password":"XXXXX","projectID":"XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"}}
   ```

NB: since we cannot currently use CSI and thus test cephfs with `oc cluster up`, we also set up a serviceaccount
_in the playground cluster only_ to test that volume provisioning/use/deprovisioning works. See details
in the test definition in [.gitlab-ci.yml](.gitlab-ci.yml). This serviceaccount's token is stored in the `TEST_APP_TOKEN`
CI variable. This can be undone when `oc cluster up` supports CSI drivers.

### Deploy the Cephfs driver

Once the GitLab CI variables have been populated, use the GitLab CI pipelines to deploy the cephfs driver components.

Running the pipeline again will update the deployment as necessary.

## Uninstall the CephFS components

Start a Docker container for installation (mounting this git repo in /project):

```bash
docker run --rm -it -v $(pwd):/project -e TILLER_NAMESPACE=paas-infra-cephfs gitlab-registry.cern.ch/paas-tools/openshift-client:v3.11.0 bash
```

Then in that container:

```bash
# login as cluster admin to the destination cluster
oc login https://openshift-test.cern.ch #or openshift-dev.cern.ch or openshift.cern.ch
# install tiller
export HELM_VERSION=2.13.1
curl "https://kubernetes-helm.storage.googleapis.com/helm-v${HELM_VERSION}-linux-amd64.tar.gz" | tar zx
mv --force linux-amd64/{helm,tiller} /usr/bin/
# initialize local tiller
helm init --client-only
helm plugin install https://github.com/adamreese/helm-local
command -v which || yum install -y which # helm local plugin needs `which`, install if missing
helm local start
helm local status
export HELM_HOST=":44134"
# delete cephfs deployment, then prerequisites
helm del --purge cephfs
helm template /project/prerequisites | oc delete -f -
```
