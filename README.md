# ZTP-RAN

## Description
This Repository provides configurations that can be used as an example to deploy OpenShift instances for RAN.

Following you can find how files are distributed in the repo.
```
tree
.
├── README.md
├── pre-reqs
│   ├── kustomization.yaml
│   ├── test-cu-lab
│   │   ├── bmh-secrets.yaml
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── pull-secret.yaml
│   └── test-du-lab
│       ├── bmh-secrets.yaml
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       └── pull-secret.yaml
├── site-configs
│   ├── kustomization.yaml
│   ├── test-cu-lab.yaml
│   └── test-du-lab.yaml
└── site-policies
    ├── common-policies
    │   └── test-common.yaml
    ├── group-policies
    │   ├── test-group-cu-odf.yaml
    │   ├── test-group-cu.yaml
    │   └── test-group-du.yaml
    ├── kustomization.yaml
    ├── ns.yaml
    └── site-specific-policies
        ├── test-cu-lab.yaml
        └── test-du-lab.yaml
```
## Pre requisites
In order to have this ZTP workflow working it will be necessary to have the following items in place:
- OpenShift Cluster acting as a Hub Cluster.
- RHACM (Red Hat Advanced Cluster Management) installed in the Hub Cluster. MultiClusterHub, hive and AssistedServiceConfig needs to be deployed and configured. To deploy RHACM and configure the required ACM objects you can follow the steps of this [repo](https://github.com/vhernandomartin/ocp4-ipibm-acm-ztp.git), follow the steps from step #2 ACM Namespace until #15 AgentServiceConfig.
- Deploy the openshift-gitops-operator in the Hub Cluster. More info [here](https://github.com/vhernandomartin/ztp-ran/blob/main/ztp-argocd/README.md).
- Configure and deploy the ArgoCD applications responsible for synchronizing both SiteConfigs and Policies. More info [here](https://github.com/vhernandomartin/ztp-ran/blob/main/ztp-argocd/README.md).

## Configurations

### pre-reqs
This directory contains the objects that needs to be created manually as a pre step before creating the SiteConfigs and PolicyGenTemplate and Policies CRs.

Just create this objects using the following commmand:
```
$ oc apply -k pre-reqs/
```

### site-configs
In this path we place the SiteConfig CRs, this CRs represents the configuration of the cluster to be deployed, in this example you will find both SNO (Single Node OpenShift) definition and multi-node OpenShift cluster definition.

### site-policies
Here we define the policies we want to apply in our OpenShift Managed clusters. In terms of Policies here we will find two different types of CRs: PolicyGenTemplate and Policy.

The Policy is a simple Policy configuration where we set certain configuration or actions that needs to be applied or executed in the cluster, while the PolicyGenTemplate is a template that it can be used to enable, deploy and configure certain operators and configurations, like: PAO (Performance Addon Operator), SRIOV Operator, LSO (Local Storage Operator), etc.

## Procedure
- Before starting, it is necessary to create your own git repository and place the files configured based on your requirements and configurations.
- The first step is to create the dependent objects, those objects like namespaces, secrets among others need to be created manually as mentioned in the ***pre-reqs*** section.
- Once the pre-reqs objects are created, it's time to enable the deployments just commiting and pushing the SiteConfigs and Policies placed in your git repo.
