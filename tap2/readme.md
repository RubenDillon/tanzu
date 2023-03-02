# Deploy Tanzu Application Platform v1.4.0 using Shared Services cluster for Harbor

## Requirements
```
    1. A vSphere cluster with vCenter based in 7.0 U3 release (Im using H20 internal environment)
    2. NSX-T at the networking infrastructure
    3. vSphere with Tanzu already deployed (10.220.22.146 is the Supervisor address)
    4. Integration with Tanzu Mission Control
    5. To deploy 
        - TAP 1.4.x we need Kubernetes v1.23, v1.24 or v1.25. We will be using 1.25
```

## Create the environment
```
    1. Join the Supervisor Cluster to TMC. 
    2. Create a Cluster Group (I create tap cluster group)
    3. Create a vSphere namespace where TAP and Harbor clusters will reside (I create a "tap" vsphere namespace)
    4. Deploy a TKGs cluster using the file tkgs-shared.yaml provided on this git
    3. Deploy Cert-Manager using the defaults
    4. Deploy Contour using the contour-values.yaml from this git
    5. crear namespace tanzu-system-service-discovery 
            kubectl create ns tanzu-system-service-discovery
    
    4. Crear un secreto con las credenciales de AWS
            kubectl create secret generic route53-credentials --from-literal=aws_access_key_id=<AWS_ACCESS_KEY_ID> --from-literal=aws_secret_access_key=<AWS_SECRET_ACCESS_KEY> -n tanzu-system-service-discovery
    
    5. We will add the certificate to the supervisor using the update-registry.yaml from this git. 
    6. You could define a serie of environment variables to easily connect to the supervisor, for example the following
            export KUBECTL_VSPHERE_PASSWORD='!BlNCj7RDQ19OqMugej' where 'xxx' is the vSphere password 
    7. Run the following        
            kubectl apply -f update-certificate.yaml
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
## Configure TAP-01 cluster
```
    1. Nos conectamos al cluster, corremos allow-run-as.yaml and then run the following
            kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated
    2. Deploy Cert-Manager from the TMC Catalog using the defaults. The current version is 1.7.2
    3. Deploy Contour from the TMC Catalog using the contour-values.yaml from this git. The current version is 1.20.2
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
        Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.4.0
```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 10.220.49.133
```      

## Create a namespace called tap-install

## Create the harbor secret
```
        kubectl get secret -n tap tap-default-image-pull-secret -o yaml > image-pull-secret.yaml
            editar image-pull-secret.yaml y modificar namespace a tap-install
        kubectl get secret -n tap tap-01-kubeconfig -o jsonpath='{.data.value}' | base64 -d > cluster-kubeconfig
        kubectl --kubeconfig=cluster-kubeconfig apply -f image-pull-secret.yaml
            luego agregue namespace default y build-service (una vez desplegado TAP)

```

## Install Tanzu Application Platform package from TMC Catalog
```
    1. Select the Tanzu Application Platform
    2. Select tap as name and 1.4.0 for the version
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

