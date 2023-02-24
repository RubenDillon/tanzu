# GIT para poder desplegar TAP usando TMC 

## Requirements
    1. A vSphere cluster with vCenter based in 7.0 U3 release (Im using H20 internal environment)
    2. NSX-T at the networking infrastructure
    3. vSphere with Tanzu already deployed
    4. Integration with Tanzu Mission Control

## Create the environment
    1. Join the Supervisor Cluster to TMC
    2. Create the Harbor registry using the embbeded version (deploy using the vCenter UI)
    3. Create a vSphere namespace where TAP will reside

## Create a TKGs cluster using TMC
    1. modify the default configuration allowing more space on the storage nodes
    2. Control Plane add space for etcd nodes.... mount the volume at /var/lib/etcd
          Workers add space for containerd ......... mount the volume at /var/lib/containerd
    3. I create a 3 control plane and 5 workers cluster with enought cpu and memory (8vCPU/64GB and 32vCPU/132GB)

## Enable Continuous Delivery and deploy Cert Manager and Contour 

## Obtain where is running ENVOY
      1. in our example we assume that is running on the following IP Address: 10.220.8.22
      
## Deploy Harbor using the values that are harbor-values.yaml
    1. Create the project tap on Harbor
    2. Desplegar el certificado de Harbor
            kubectl -n harbor get secret harbor-tls -o=jsonpath="{.data.ca\.crt}" 
    3. Editar y copiar el certificado
            editar harbor-tkgs.yaml con el certificado base64 obtenido
            loguerse en el supervisor
            kubectl apply -f harbor-tkgs.yaml
    4. Esperar que se generen los cambios en los nodos (se crean automaticamente un nuevo set de nodos control plane y workers)
    5. Este proceso en un entorno H20 tardo aproximadamente 80 minutos

## Create a namespace called tap-install

## Create a secret for Tanzu Net and export it to all namespaces
    Secret name: tap-registry
    Image registry URL: registry.tanzu.vmware.com
    Username: <tanzu-net-username>
    Password: <tanzu-net-password>

## Verify secret is reconciled and is exported

## Add Tanzu Application Platform package repository to the cluster
    Name: tanzu-tap-repository
    Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.4.1

## Verify tanzu-tap-repository package repository is successfully added

## Install Tanzu Application Platform package
    The Tanzu Application Platform package is at bottom of list

## Select version from drop-down eg 1.4.1 and then click "Install Package"

## Name package and confirm version
    Installed package name: tap
    Package version: 1.0.1

## Package install resources
    Leave defaults

## Configure values
    Copy the light or full profile from TAP docs and configure the parameters as guided in that doc

Monitor the install

Verify all packages are successfully reconciled

Ta-da... cluster created and TAP installed all from TMC
  
.......................

  Basado en https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
