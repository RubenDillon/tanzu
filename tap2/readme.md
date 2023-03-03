# Deploy Tanzu Application Platform v1.3.5 using Azure

## Requirements
```
    1. An Azure subscription
    2. A management cluster deployed
    4. Integration with Tanzu Mission Control
    5. To deploy 
        - TAP 1.3.5 we need Kubernetes v1.22, v1.23 or v1.24. We will be using 1.22.5
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
            
            kubectl create secret generic route53-credentials --from-literal=aws_access_key_id=ASIA33SE4POIFC74GSDM --from-literal=aws_secret_access_key=VmM0EpFiTZM8RiNEWmsPVGzgO9AkD617Uy/u8OJo -n tanzu-system-service-discovery
            
    7. Deploy DNS-external from TMC Catalog using the DNS-external-values.yaml from this git
    8. Create the namespace tanzu-system-registry
            kubectl create ns tanzu-system-registry
    9. Create a Cluster issuer using the following command

kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-contour-cluster-issuer
  namespace: tanzu-system-ingress
spec:
  acme:
    email: "rdillon@vmware.com"
    privateKeySecretRef:
      name: acme-account-key
    server: https://acme-v02.api.letsencrypt.org/directory
    solvers:
    - http01:
        ingress:
          class: contour
EOF
    
    10. Request the certificate as follow
    
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: harbor-cert
  namespace: tanzu-system-registry
spec:
  secretName: harbor-cert-tls
  duration: 2160h
  renewBefore: 360h
  subject:
    organizations:
    - vmware
  commonName: harbor.solateam.be
  isCA: false
  privateKey:
    size: 2048
    algorithm: RSA
    encoding: PKCS1
  dnsNames:
  - harbor.solateam.be
  - notary.harbor.solateam.be
  issuerRef:
    name: letsencrypt-contour-cluster-issuer
    kind: ClusterIssuer
EOF
    
    11. Now we will expect to wait more or less 10 minutes until we could have the certificate. To verify it use the following command
    
            kubectl get certificates -n tanzu-system-registry harbor-cert
    
    12. When you have TRUE in ready column, you will continue to the next step that is obtain the certificates
        
            kubectl get secret harbor-cert-tls -n tanzu-system-registry -o=jsonpath={.data."tls\.crt"} | base64 --decode > tls-crt.txt
            
            kubectl get secret harbor-cert-tls -n tanzu-system-registry -o=jsonpath={.data."tls\.key"} | base64 --decode > tls-key.txt
      
        Save both results
        
        
    13. Deploy Harbor from the TMC Catalog using harbor-values.yaml from this git. Modify the fields whith yours certificate.
    
    14. Connect to harbor and create "tap" project as public.
            
```       
      
## Configure the cluster
```
    1. Nos conectamos al cluster, corremos allow-run-as.yaml and then run the following
            kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated

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
        Repository URL: registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.3.5
```

## Obtain where is running ENVOY
```
      In our example we assume that is running on the following IP Address: 20.246.129.194
```      

## Create a namespace called tap-install

## Install Tanzu Application Platform package from TMC Catalog
```
    1. Select the Tanzu Application Platform
    2. Select tap as name and 1.3.5 for the version
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
  - https://confluence.eng.vmware.com/pages/viewpage.action?spaceKey=CNA&title=Installing+Harbor+with+LE+certs+via+TMC

