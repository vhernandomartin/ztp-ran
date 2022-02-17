# ZTP-ARGOCD

## Description
This procedure describes the required steps to deploy ArgoCD in OpenShift and provides the configuration required to let the ArgoCD prepared to enable the ZTP pipeline.

## Procedure

1. Install the gitops Operator in the Hub Cluster. We force the 1.3.2 version because we found a bug with the latest version at the deployment time (Feb 2022). Note that it is required to approve manually the install plan.
```
$ oc create -f configs/gitops_subscription.yaml

$ oc get installplan
NAME            CSV                                APPROVAL   APPROVED
install-k2j5x   openshift-gitops-operator.v1.3.2   Manual     false

$ oc patch installplan install-k2j5x -n openshift-operators --type merge --patch '{"spec":{"approved":true}}'
installplan.operators.coreos.com/install-k2j5x patched

$ oc get installplan
NAME            CSV                                APPROVAL   APPROVED
install-9xbl4   openshift-gitops-operator.v1.4.0   Manual     false
install-k2j5x   openshift-gitops-operator.v1.3.2   Manual     true

$ oc get csv
NAME                               DISPLAY                    VERSION   REPLACES                           PHASE
openshift-gitops-operator.v1.3.2   Red Hat OpenShift GitOps   1.3.2     openshift-gitops-operator.v1.3.1   Succeeded

$ oc get argocds.argoproj.io -n openshift-gitops
NAME               AGE
openshift-gitops   2m21s

$ oc get pods -n openshift-gitops
NAME                                                          READY   STATUS    RESTARTS   AGE
cluster-85568c9df4-tk824                                      1/1     Running   0          2m12s
kam-67f7c8c5cd-k76tw                                          1/1     Running   0          2m12s
openshift-gitops-application-controller-0                     1/1     Running   0          2m12s
openshift-gitops-applicationset-controller-5cc6fb7fd4-7v4j7   1/1     Running   0          2m11s
openshift-gitops-dex-server-68c58ff59f-5c2jn                  1/1     Running   0          2m11s
openshift-gitops-redis-7867d74fb4-7wkw9                       1/1     Running   0          2m12s
openshift-gitops-repo-server-6674bdc5c4-7mxsp                 1/1     Running   0          2m12s
openshift-gitops-server-c6b674b4d-zgnzh                       1/1     Running   0          2m12s
```

2. Get the ArgoCD Admin password for future use.
```
$ oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d
```

3. Prepare the ArgoCD pipeline configuration.

   - Download the ztp-site-generator image and extract the contents.
```
$ pwd
/home/kubeadmin

$ podman pull quay.io/redhat_emp1/ztp-site-generator:latest
Trying to pull quay.io/redhat_emp1/ztp-site-generator:latest...
Getting image source signatures
Copying blob 47635cb10a81 skipped: already exists
Copying blob 1f8510ff4372 skipped: already exists
Copying blob 86adcf8407f7 skipped: already exists
Copying blob ac10f00499d5 skipped: already exists
Copying blob 95ca19d9993d skipped: already exists
Copying blob 96d53117c12e skipped: already exists
Copying blob 653dfa3b9579 skipped: already exists
Copying blob aab27cc9e311 skipped: already exists
Copying blob bb7e46dcd67e skipped: already exists
Copying blob 6165b3da935a skipped: already exists
Copying blob 3d973de6b383 [--------------------------------------] 0.0b / 0.0b
Copying blob 672b01ac2031 [--------------------------------------] 0.0b / 0.0b
Copying blob 57e1c3669277 [--------------------------------------] 0.0b / 0.0b
Copying blob ec5120993978 [--------------------------------------] 0.0b / 0.0b
Copying config 58edc360b7 done
Writing manifest to image destination
Storing signatures
58edc360b724da144c92a59c0b3aa1061963872dedaa7fe84b23096a1b5e5e4e

$ mkdir -p ./out
$ podman create -ti --name ztp-site-gen ztp-site-generator:latest bash
3c296159554410b2d9070c44b6d4d858619c7e452bd5b80daf76df2afa1e4562

$ podman cp ztp-site-gen:/home/ztp ./out
$ podman rm -f ztp-site-gen
3c296159554410b2d9070c44b6d4d858619c7e452bd5b80daf76df2afa1e4562
```
   - Configure the ArgoCD applications, you can get an example in this repo from **ztp-ran/ztp-argocd/configs/argocd_apps.yaml** .
```
$ cd $PWD/out/ztp/argocd
$ vi deployment/clusters-app.yaml
$ vi deployment/policies-app.yaml
```
4. If you plan to use other namespaces names different from ztp*whatever for storing policies and configs do the following. For example if we use namespace name pattern like **test**
```
$ grep -A 1 test deployment/policies-app-project.yaml
  - namespace: 'test*'
    server: '*'
```

5. Patch the ArgoCD instance to enable the external plugin and the ztp-site-generator image (quay.io/redhat_emp1/ztp-site-generator:latest).
```
$ oc patch argocd openshift-gitops -n openshift-gitops --patch-file deployment/argocd-openshift-gitops-patch.json --type=merge
argocd.argoproj.io/openshift-gitops patched

$ oc patch deployment openshift-gitops-repo-server -n openshift-gitops --patch-file deployment/deployment-openshift-repo-server-patch.json
deployment.apps/openshift-gitops-repo-server patched
```

6. Add your git repo to the ArgoCD configuration. From the ArgoCD UI go to: Settings -> Repositories -> Connect Repo using xxx. You can get the ArgoCD URL with the following command.
```
$ oc -n openshift-gitops get route openshift-gitops-server -o jsonpath='{.spec.host}'
openshift-gitops-server-openshift-gitops.apps.hubcluster.test.redhat.lab
```

7. Apply the prerequisites from the ZTP git repo where all the SiteConfig and Policies are located.
```
$ oc apply -k pre-reqs/
```

8. Apply the pipeline ArgoCD configuration.
```
$ cd $HOME/out/ztp/argocd
$ pwd
/home/kubeadmin/out/ztp/argocd

$ oc apply -k deployment/
namespace/clusters-sub created
namespace/policies-sub created
clusterrolebinding.rbac.authorization.k8s.io/gitops-cluster unchanged
clusterrolebinding.rbac.authorization.k8s.io/gitops-policy unchanged
appproject.argoproj.io/policy-app-project created
appproject.argoproj.io/ztp-app-project created
application.argoproj.io/clusters created
application.argoproj.io/policies created
```

9. Check the ArgoCD Status.
```
$ oc get applications.argoproj.io
NAME       SYNC STATUS   HEALTH STATUS
clusters   Synced        Healthy
policies   Synced        Healthy
```
