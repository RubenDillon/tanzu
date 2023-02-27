# Desplegando Tanzu Application Platform 

## Requirements
```
    1. A vSphere cluster with vCenter based in 7.0 U3 release (Im using H20 internal environment)
    2. NSX-T at the networking infrastructure
    3. vSphere with Tanzu already deployed (10.220.49.130 is the Supervisor address)
    4. Integration with Tanzu Mission Control
    5. To deploy 
        - TAP 1.0.2 we need Kubernetes v1.20, v1.21 or v1.22. We are using 1.21.6
        - TAP 1.4.x we need Kubernetes v1.23, v1.24 or v1.25.
```

## Create the environment
```
    1. Join the Supervisor Cluster to TMC
    2. Create the Harbor registry using the embbeded version (deploy using the vCenter UI)
    3. Create a Cluster Group (I create ruben-group cluster group)
    4. At the Cluster Group I activate the Continuous Delivery and deploy Cert Manager and Contour using the tmc_flux git
    5. Create a vSphere namespace where TAP will reside (I create a "tap" vsphere namespace)
    6. Usamos update-registry.yaml para agregar el certificado tap-registry al Supervisor 
    
    (quizas agregarlo antes de crear el cluster...)
    
            conectarse al supervisor
            
            ejecutar kubectl apply -f update.yaml
```       
      
## Create a TKGs cluster using TMC
```
    1. Create the cluster where TAP will reside. For that, use the tkgs-deployment.yaml from this git. 
    2. Modify the Trusts part of the file. You need to convert the Harbor certificate to base64 enconde.
            Obtain the certificate from the Harbor UI (go to tap project, reporsitories and then download Registry Certificate)
            To convert the certificate to base64 encode use for example https://www.online-toolz.com/tools/base64-decode-encode-online.php    
    2. TKGs-deployment.yaml creates a cluster with more space on the storage nodes than the default
            Control Plane a 100GB volume for etcd nodes........ mounting the volume at /var/lib/etcd
            Workers add a 200GB volume for containerd ......... mounting the volume at /var/lib/containerd
    2. That files creates a 3 nodes control plane and 5 workers nodes cluster
```
## Configure our docker machine to accept Harbor certificate
```
    1. Nos conectamos al cluster, corremos allow-run-as.yaml and then run the following
            kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated
    2. Descargar el certificado de harbor y desplegarlo en el equipo desde donde vamos a instalar TAP. Loguearse a harbor 
```
## Relocate container images to local registry
```
    1. Loguearse a la registry de VMware 
            docker login registry.tanzu.vmware.com
    2. Create the variables
            export INSTALL_REGISTRY_USERNAME=administrator@vsphere.local
            export INSTALL_REGISTRY_PASSWORD='xxxxx'
            export INSTALL_REGISTRY_HOSTNAME=10.220.49.131
            export TAP_VERSION=1.0.2
    3. Relocate images using carvel
            imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/tap/tap-packages
    4. This process will take more or less 2 hours

```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 10.220.49.133
```      

## Create a namespace called tap-install

## Create a registry secret and add the Tanzu Application Platform repository 
```
    1. Run the following to create the registry secret
            tanzu secret registry add tap-registry \
            --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
            --server ${INSTALL_REGISTRY_HOSTNAME} \
            --export-to-all-namespaces --yes --namespace tap-install
        
    ---- agregue la parte del certificado de harbor en el tKGs y lo apunte a tap-install -------    
        
    2. Run the following to add the repository
            tanzu package repository add tanzu-tap-repository \
            --url ${INSTALL_REGISTRY_HOSTNAME}/tap/tap-packages:$TAP_VERSION \
            --namespace tap-install
    
```

## Install Tanzu Application Platform package
```
    1. Get the status of the repository and ensure the status updates to Reconcile succeeded by running
            tanzu package repository get tanzu-tap-repository --namespace tap-install
    2. List the available packages 
            tanzu package available list --namespace tap-install
```

## Configure values and deploy TAP
```
    1. Copy the tap-values.yaml from this git. Modificar el certificado, usando el del repositorio TAP de Harbor
    2. Deploy TAP using the following 
            tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-values.yml -n tap-install
    3. Monitor the install by running
            tanzu package installed get tap -n tap-install
    4. Verify all packages are successfully reconciled
            tanzu package installed list -A
            
            

    YES!!!! ... TAP deployed 
    
    
    
```  


  Basado en 
  - https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.0/tap/GUID-install.html
