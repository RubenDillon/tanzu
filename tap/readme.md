# Desplegando Tanzu Application Platform usando TMC Catalog

## Requirements
```
    1. A vSphere cluster with vCenter based in 7.0 U3 release (Im using H20 internal environment)
    2. NSX-T at the networking infrastructure
    3. vSphere with Tanzu already deployed (10.220.49.130 is the Supervisor address)
    4. Integration with Tanzu Mission Control
```

## Create the environment
```
    1. Join the Supervisor Cluster to TMC
    2. Create the Harbor registry using the embbeded version (deploy using the vCenter UI)
    3. Create a Cluster Group (I create ruben-group cluster group)
    4. At the Cluster Group I activate the Continuous Delivery and deploy Cert Manager and Contour using the tmc_flux git
    5. Create a vSphere namespace where TAP will reside (I create a tap vsphere namespace)
```
## Create a TKGs cluster using TMC
```
    1. modify the default configuration allowing more space on the storage nodes
            Control Plane a 100GB volume for etcd nodes........ mount the volume at /var/lib/etcd
            Workers add a 200GB volume for containerd ......... mount the volume at /var/lib/containerd
    2. I create a 3 nodes control plane and 5 nodes workers cluster with enought cpu and memory (8vCPU/64GB and 32vCPU/132GB)
```
## Configure TKGs to accept Harbor certificate
```
    1. Nos conectamos al cluster y corremos allow-run-as.yaml
    2. Conectarse al Supervisor Cluster y luego al vsphere namespace donde el cluster se aloja
            kubectl vsphere login --vsphere-username administrator@vsphere.local --server=https://10.220.49.130  --insecure-skip-tls-verify
            kubectl config use-context tap
            kubectl get secret -n tap tap-default-image-pull-secret -o yaml > image-pull-secret.yaml
    2. Editar image-pull-secret.yaml para definir como namespace a default
    3. Crear el kubeconfig
            kubectl get secret -n tap tap-cluster-kubeconfig -o jsonpath='{.data.value}' | base64 -d > cluster-kubeconfig
    4. Crear el secreto en el cluster
            kubectl --kubeconfig=cluster-kubeconfig apply -f image-pull-secret.yaml

```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 10.220.49.133
```      

## Create a namespace called tap-install

## Using TMC create a secret for Tanzu Net and export it to all namespaces
```
    Secret name: tap-registry
    Image registry URL: registry.tanzu.vmware.com
    Namespace: tap-install
    Username: <tanzu-net-username>
    Password: <tanzu-net-password>
```
## Verify secret is reconciled and is exported

## Add Tanzu Application Platform package repository to the cluster
```
    Name: tanzu-tap-repository
    Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.0.1 (1.4.1 is the last one)
```

## Install Tanzu Application Platform package
```
    The Tanzu Application Platform package is at bottom of list
```
## Select version from drop-down eg 1.0.1 and then click "Install Package"

## Name package and confirm version
```
    Installed package name: tap
    Package version: 1.0.1
```

## Configure values
```
    Copy the tap-values.yaml from this git. Modificar el certificado, usando el del repositorio TAP de Harbor
    Monitor the install
    Verify all packages are successfully reconciled

    YES!!!! ... TAP deployed using TMC Catalog
```  
## .......................

  Basado en https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
