---
category: SAS Viya with SingleStore
tocprty: 3
---

# SAS SingleStore Cluster Operator

## Overview

`$deploy/sas-bases/examples/sas-singlestore/sas-singlestore-cluster-config.yaml` is a patch transformer that can be used to override the default configuration of the SingleStore database created when deploying an integrated SAS Viya and SingleStore environment.

The SingleStore Operator documentation related to cluster configuration is located at [the SingleStore website](https://docs.singlestore.com/db/v7.6/en/deploy/kubernetes/create-the-object-definition-files/memsql-cluster-yaml.html). 

If your order includes SingleStore integration, the following is deployed by default:

 * The SingleStore operator
 * A daemonset that tunes operating system parameters as [specified by SingleStore](https://docs.singlestore.com/db/v7.6/en/reference/memsql-operator-reference/set-system-requirements.html)
 * A SingleStore cluster that has the following attributes:
   * No Redundancy
   * Two aggregators
      * 512 GB persistent volume
      * azure-premium storage class
   * Two leaf nodes
      * 1200 GB persistent volume
      * azure-premium storage class

Recommendations for SingleStore infrastructure on Azure are located at the [SingleStore System Requirements and Recommendations page](https://docs.singlestore.com/db/v7.6/en/reference/configuration-reference/cluster-configuration/system-requirements-and-recommendations.html). SingleStore engineers also recommend that you use Azure CNI as the Kubernetes network provider and Azure managed-premium storage for your storage. SingleStore also notes that some customer workloads may require Azure Ultra SSD.

## SingleStore Cluster Definition

The configuration of the SingleStore cluster is site-specific.  To create a SingleStore cluster in your deployment:

 * Copy `$deploy/sas-bases/examples/sas-singlestore` into `$deploy/site-config`.
 * Edit `$deploy/site-config/sas-singlestore/sas-singlestore-cluster.yaml`: 
   * Paste the provided SingleStore license code into the file replacing the string "{{ LICENSE-CODE }}"
   * Use the following Python code to obtain the hashed value for the password you wish to use for the admin account. Paste the resulting output into the file, replacing the string "{{HASHED-ADMIN-PASSWORD }}". ***Note:*** The initial asterisk(*) must be included in the hashed password.
  
  ```python
  from hashlib import sha1
  print("*" + sha1(sha1('secretpass'.encode('utf-8')).digest()).hexdigest().upper())
  ```
 
 * Add the following to your base kustomization.yaml ($deploy/`kustomization.yaml`) file:

```yaml
...
transformers:
  - site-config/sas-singlestore/sas-singlestore-cluster-config.yaml
  - sas-bases/overlays/sas-singlestore/transformers.yaml
  - sas-bases/overlays/required/transformers.yaml
...
configurations:
  - sas-bases/overlays/sas-singlestore/kustomizeconfig.yaml
...
vars:
  - name: {{ SAS_COMPONENT_TAG }}_sas-singlestore-node
    objref:
      kind: ConfigMap
      name: sas-components
      apiVersion: v1
    fieldref:
      fieldpath: data.{{ SAS_COMPONENT_TAG }}_sas-singlestore-node
  - name: {{ IMAGE_REGISTRY }}
    objref:
      name: input
      kind: ConfigMap
      apiVersion: v1
    fieldref:
      fieldpath: data.{{ IMAGE_REGISTRY }}
  - name: {{ SAS_COMPONENT_RELPATH }}_sas-singlestore-node
    objref:
      kind: ConfigMap
      name: sas-components
      apiVersion: v1
    fieldref:
      fieldpath: data.{{ SAS_COMPONENT_RELPATH }}_sas-singlestore-node
```

 * You can also override other cluster attributes, such as the number of leaf nodes, the storage class, or the amount of storage allocated to each node type.  In the following example, the leaf node definition has been modified to create 4 leaf nodes each with 750 GB of storage and to use the "managed" storage class.

```yaml
  - op: replace
    path: /spec/leafSpec/count
    value: 4
  - op: replace
    path: /spec/leafSpec/height
    value: 1
  - op: replace
    path: /spec/leafSpec/storageGB
    value: 750
  - op: replace
    path: /spec/leafSpec/storageClass
    value: managed
```