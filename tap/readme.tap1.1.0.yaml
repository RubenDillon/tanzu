# Desplegando Tanzu Application Platform 

## Requirements
```
    1. A vSphere cluster with vCenter based in 7.0 U3 release (Im using H20 internal environment)
    2. NSX-T at the networking infrastructure
    3. vSphere with Tanzu already deployed (10.220.49.130 is the Supervisor address)
    4. Integration with Tanzu Mission Control
    5. To deploy 
        - TAP 1.1.0 we need Kubernetes v1.20, v1.21 or v1.22. We are using 1.21.6
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
## create the secret using TMC
```
        Secret name: tap-registry
        Image registry URL: registry.tanzu.vmware.com
        Username: <tanzu-net-username>
        Password: <tanzu-net-password>

```

## Create the repository
```
        Name: tanzu-tap-repository
        Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.1.0
```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 10.220.49.132
```      

## Create a namespace called tap-install

## Install Tanzu Application Platform package from TMC Catalog
```
    1. Select the Tanzu Application Platform
    2. Select tap as name and 1.1.0 for the version
    3. Copy the tap-values.yaml from this git. Modificar el certificado, usando el del repositorio TAP de Harbor
    4. Monitor the install by running
            tanzu package installed get tap -n tap-install
    5. Verify all packages are successfully reconciled
            tanzu package installed list -A
            
            
    YES!!!! ... TAP deployed 
    
    
```  

## Review the Self-Guided Workshop
```
    1. run the following command to review the activated portals
            kubectl get trainingportals
    2. then connect to the defined URL
```
    


  Basado en 
  - https://confluence.eng.vmware.com/display/CNA/TAP+-+How+to+install+TAP+using+TMC
  - https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.0/tap/GUID-install.html
